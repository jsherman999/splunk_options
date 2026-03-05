# PODMANIA thoughts

This is the cleanest way to get logs from your Podman-based mock servers into the local Splunk instance in this repo without turning every container into a Splunk client.

## Recommended topology

```text
mock server containers
  -> syslog over TCP/1514
  -> jump server container
  -> Splunk Universal Forwarder
  -> local Splunk Enterprise on tcp/9997
  -> index=podmania
```

## Why this path

- It matches a real server environment better than scraping `podman logs`.
- The jump container becomes the only box that needs Splunk-specific configuration.
- The local Splunk instance in this repo is now configured to receive forwarded data on `9997`.
- You avoid putting Splunk credentials or HEC tokens into every mock server container.

## What I changed locally

- [docker-compose.yml](/Users/jay/splunk/docker-compose.yml#L15) now exposes `9997`.
- [splunk/defaults/default.yml](/Users/jay/splunk/splunk/defaults/default.yml#L4) now creates an index named `podmania`.
- [splunk/defaults/default.yml](/Users/jay/splunk/splunk/defaults/default.yml#L16) now enables Splunk's receiving port on `splunktcp://9997`.

## Start the local Splunk side

```bash
SPLUNK_PASSWORD='StrongPassw0rd!' docker compose up -d
```

Then confirm it is up:

```bash
docker compose ps
docker compose logs -f splunk
```

## Addressing from Podman

From Podman containers, the host machine is often reachable as `host.containers.internal`. Test this first from the jump container:

```bash
curl -sk https://host.containers.internal:8089/services/server/info?output_mode=json
nc -vz host.containers.internal 9997
```

If that name does not resolve in your setup, use the actual host-reachable IP instead and inject it into the jump container with `--add-host` or equivalent Podman network config.

## Jump container setup

Install a Splunk Universal Forwarder in the jump container. The forwarder should:

- listen for syslog on `tcp/1514`
- label it as `sourcetype=linux:syslog`
- send it to `host.containers.internal:9997`
- write into `index=podmania`

Example `inputs.conf` for the jump container's forwarder:

```ini
[tcp://1514]
disabled = 0
index = podmania
sourcetype = linux:syslog
connection_host = dns
```

Example `outputs.conf`:

```ini
[tcpout]
defaultGroup = local_splunk

[tcpout:local_splunk]
server = host.containers.internal:9997
```

If your jump container cannot use `host.containers.internal`, replace that with the actual host IP reachable from the Podman network.

## Mock server container setup

On each mock Linux server container, forward syslog to the jump container over TCP on port `1514`.

For `rsyslog`, the minimal classic form is:

```conf
*.* @@jump-hostname-or-ip:1514
```

The `@@` means TCP. Use the jump container's hostname or IP as seen from the other Podman containers.

If you want a more explicit `rsyslog` action:

```conf
action(
  type="omfwd"
  target="jump-hostname-or-ip"
  port="1514"
  protocol="tcp"
  action.resumeRetryCount="-1"
  queue.type="linkedList"
  queue.size="10000"
)
```

After updating `rsyslog`, restart it in each server container.

## If your mock servers only log to files

If the services inside those containers write to files instead of syslog, feed those files into `rsyslog` first and still forward from there.

Example `rsyslog` file input:

```conf
module(load="imfile")

input(
  type="imfile"
  File="/var/log/myapp/app.log"
  Tag="myapp"
  Severity="info"
  Facility="local0"
)
```

Then keep the same forwarding rule to the jump container.

## Search in Splunk

Once data is flowing:

```spl
index=podmania
```

Useful first checks:

```spl
index=podmania | stats count by host, sourcetype
```

```spl
index=podmania sourcetype=linux:syslog | stats count by host, source
```

```spl
index=podmania ("error" OR "failed" OR "denied")
```

## Alternative paths

- Direct syslog to Splunk Enterprise: workable for a lab, but less flexible than a forwarder on the jump box.
- HEC on Splunk plus Fluent Bit or Vector on the jump box: also good, but adds token management and is better suited to structured app logs than classic Linux syslog.
- Scraping `podman logs`: acceptable for container stdout testing, but weaker if you are trying to simulate real Linux hosts.

## Recommendation

Use the jump container as the only Splunk-aware node:

1. mock servers forward syslog to jump on `1514`
2. jump runs Splunk Universal Forwarder
3. forwarder ships to local Splunk on `9997`

That gives you realistic host-like logging with the least repeated configuration.

## Sources

- Podman networking and host aliases: https://docs.podman.io/
- Splunk data forwarding overview: https://help.splunk.com/en/data-management/forward-data/forwarding-and-receiving-data
- Splunk `inputs.conf` reference: https://docs.splunk.com/Documentation/Splunk/latest/Admin/Inputsconf
