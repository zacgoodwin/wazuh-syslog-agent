# syslog-collector

Two services, no image builds. `rsyslog/rsyslog-collector` (stock config) receives syslog on
514/udp and 514/tcp and writes `/var/log/all.log` to the `syslog_data` volume. The Wazuh
agent mounts that volume at `/var/log/remote`, tails `all.log`, and forwards to the manager
on 1514/tcp. Size-based rotation runs as a background loop inside the rsyslog container.

## Run

```
cp .env.example .env        # set WAZUH_MANAGER_SERVER, match WAZUH_VERSION to your manager
docker compose up -d
```

## Verify

```
docker compose logs rsyslog | head        # entrypoint runs `rsyslogd -N1` and exits on a bad config
docker compose exec rsyslog cat /proc/1/comm                 # must print rsyslogd (PID 1)
docker compose restart rsyslog && docker compose ps rsyslog  # must come back Up, not restart-loop
logger -n <docker-host-ip> -P 514 -d "wazuh syslog test"     # from another host
docker compose exec rsyslog tail -2 /var/log/all.log
docker compose logs wazuh-agent | grep -E "Analyzing file|Connected to"

# force a rotation to prove the HUP path
docker compose exec rsyslog sh -c 'mv /var/log/all.log /var/log/all.log.1 && kill -HUP 1'
logger -n <docker-host-ip> -P 514 -d "after rotate"
docker compose exec rsyslog ls -l /var/log/all.log*          # new all.log must exist and hold the line
```

## Behaviour worth knowing

- **Line format.** The image writes `RSYSLOG_FileFormat` (`2026-09-20T06:43:00.123456-07:00 host prog: msg`).
  Wazuh's pre-decoder handles it (`src/analysisd/cleanevent.c:90`, v4.14.7).
- **Rotation is rename + HUP, not truncate.** Truncating a tailed file makes the agent send
  "File size reduced": rule 592, level 8, group `attacks`
  (`ruleset/rules/0015-ossec_rules.xml:298`). A rename shows up as "File rotated", rule 591,
  level 3, and the agent re-reads the new `all.log` from the start
  (`src/logcollector/logcollector.c:745-766`). Checked every 5 min against `SYSLOG_ROTATE_MB`.
- **rsyslog has native size rotation** (`rotation.sizeLimit` / `rotation.sizeLimitCommand` on
  omfile). Not used: it means replacing the image's `80-file-output.conf` and mounting a script.
- **Restarts resume.** `only-future-events=no` plus the `/var/ossec/queue/logcollector` volume
  resumes from the saved offset. Offsets are saved every 64s and at exit, so a hard kill can
  replay up to ~64s of lines. If more than `max-size` piled up while the agent was down it
  skips to END. Wazuh's default is 10M (`src/config/localfile-config.h:23`); `ossec.conf`
  sets 200M. Keep it above `SYSLOG_ROTATE_MB`.
- **Lines in `all.log.N.gz` are never read.** If the agent is down across a rotation, whatever
  it had not read from the rotated file is lost.
- **rsyslogd must stay PID 1.** As PID 1 it skips its PID file (`tools/rsyslogd.c`, "running as
  pid 1, enabling container-specific defaults"). As any other PID it writes `/run/rsyslogd.pid`,
  which survives a container restart and makes the next start abort with "pidfile ... already
  exist". The `exec` in the compose `command` is what keeps it PID 1. Do not add `init: true`.
- **Editing `ossec.conf`.** `./wazuh-agent/` is mounted as a directory and copied into
  `/var/ossec/etc` at each start. Edit, then `docker compose restart wazuh-agent`. Keep other
  files out of that directory; they get copied too.
- **If UDP logs stop right after recreating the rsyslog container:** a device that sends
  constantly from one source port can stay pinned to the old container IP by a stale conntrack
  entry. `conntrack -D -p udp --dport 514` on the Docker host clears it. Current Docker
  releases are supposed to flush these on container start; I did not verify that.
- **One file for all devices.** The device is the hostname field of each line. Per-device
  files need a drop-in in `/etc/rsyslog.d/`.
- **Source IP under Docker.** LAN traffic keeps its real source through the published port.
  Traffic from the Docker host itself, or rootless Docker, arrives as the bridge gateway.
- **client.keys persists** in `wazuh_agent_etc`. `docker compose down -v` deletes it and the
  agent re-enrolls; remove the old agent on the manager first.
- **Agent upgrades.** `wazuh_agent_etc` holds all of `/var/ossec/etc`, so `internal_options.conf`
  stays at the version first installed. Same approach upstream uses in its 5.x compose.
- Wazuh 5.x changes agent transport (HTTPS on 1517, different env vars). This targets 4.14.x.

## What was checked before delivery

- `docker compose config`: passes; fails loudly when `WAZUH_MANAGER_SERVER` is unset.
- `ossec.conf`: well-formed XML; placeholders match the agent image's init script.
- Rotation wrapper: syntax-checked in dash and run end to end against stubs: pre-create,
  recovery after `all.log` is deleted by hand, three rotations, shift order, KEEP limit, and
  `exec` keeping the same PID. `rsyslogd` and `kill -HUP 1` were stubbed.
- Image behaviour (paths, env vars, entrypoint, root user): read from
  `packaging/docker/rsyslog/` in the rsyslog repo. Wazuh behaviour: read from v4.14.7 source.
- NOT checked: any container actually running. Docker Hub was blocked from the build
  sandbox. Unverified at runtime: that the image has `stat` and `gzip` (Ubuntu Essential
  packages, nothing in the Dockerfiles removes them), the real HUP, and the
  `RSYSLOG_TAG` value. Pin a tag from https://hub.docker.com/r/rsyslog/rsyslog-collector/tags.
