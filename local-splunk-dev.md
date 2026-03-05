# Local Splunk Enterprise Dev Setup

This workspace now includes a minimal local Splunk Enterprise environment for API and search development.

## Files Added

- `docker-compose.yml`
- `splunk/defaults/default.yml`
- `PODMANIA-thoughts.md`

## What This Setup Does

- Runs the official `splunk/splunk:10.2.1` container.
- Exposes Splunk Web on `https://localhost:8000`.
- Exposes the `splunkd` management and REST API on `https://localhost:8089`.
- Exposes Splunk-to-Splunk forwarding on `tcp://localhost:9997`.
- Persists Splunk state in Docker named volumes.
- Creates a local index named `devsample`.
- Creates a local index named `podmania` for forwarded lab data.
- Monitors `/data/sample/events.log` inside the container.
- Runs a second service, `sample-log-writer`, that appends JSON fixture events to that file every 2 seconds.

## Start It

```bash
SPLUNK_PASSWORD='StrongPassw0rd!' docker compose up -d
```

If Docker on Apple Silicon reports an image architecture mismatch, uncomment `platform: linux/amd64` in `docker-compose.yml`.

## Stop It

```bash
docker compose down
```

To also remove persisted Splunk state:

```bash
docker compose down -v
```

## Access

- Splunk Web: `https://localhost:8000`
- REST API: `https://localhost:8089`
- Forwarder receiver: `tcp://localhost:9997`
- Username: `admin`
- Password: the value of `SPLUNK_PASSWORD`

The container uses Splunk's self-signed certificate by default, so browser and `curl` certificate warnings are expected unless you replace the certs.

## Verify the Instance

Check service state:

```bash
docker compose ps
```

Check Splunk logs:

```bash
docker compose logs -f splunk
```

Check the REST API:

```bash
curl -k -u "admin:$SPLUNK_PASSWORD" \
  "https://localhost:8089/services/server/info?output_mode=json"
```

## Verify Sample Data

Once the stack is up, the `sample-log-writer` service continuously writes JSON events like:

```json
{"timestamp":"2026-03-05T22:00:00Z","service":"checkout","env":"local","event":"sample_request","seq":42,"level":"INFO","ok":true,"duration_ms":175}
```

Search for them in Splunk:

```spl
index=devsample
```

Useful starter searches:

```spl
index=devsample | stats count by level
```

```spl
index=devsample | timechart count by level
```

```spl
index=devsample | stats avg(duration_ms) as avg_ms p95(duration_ms) as p95_ms by service
```

## Podman Lab Integration

The local Splunk instance is also configured to receive data from a Splunk Universal Forwarder on port `9997` into the `podmania` index.

The recommended pattern for your Podman lab is:

1. Each mock Linux server container forwards syslog to the jump container on a high TCP port such as `1514`.
2. The jump container runs a Splunk Universal Forwarder that listens on `1514`.
3. That forwarder sends the events to this local Splunk instance on `host:9997`.

Detailed notes and example configs are in `PODMANIA-thoughts.md`.

## Notes

- This is intended for local development against real Splunk Enterprise behavior, especially the `splunkd` REST endpoints described in `splunk-data-mining-research.md`.
- I validated the Compose configuration with `docker compose config`.
- I did not start the containers in this session, so image pull, first-run startup, and data ingestion have not been exercised here yet.
