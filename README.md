# ntp-server — chrony on Alpine

Self-contained NTP server. Everything is configured in `.env`; `chrony.conf` is generated at container start.

## Install (first time)

Run from your home folder. This installs through `deploy-update.sh` itself, so the update baseline is recorded from day one and later updates apply cleanly:
```
cd ~ && \
curl -fsSL -o /tmp/ntp.zip https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/ntp-server.zip && \
python3 -c "import zipfile; open('/tmp/deploy-update.sh','wb').write(zipfile.ZipFile('/tmp/ntp.zip').read('ntp-server/deploy-update.sh'))" && \
sh /tmp/deploy-update.sh https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/ntp-server.zip ntp-server && \
cp /tmp/deploy-update.sh ntp-server/ && chmod +x ntp-server/*.sh && rm -f /tmp/ntp.zip /tmp/deploy-update.sh
```
```
nano ~/ntp-server/.env
```
If `NTP_SET_SYSTEM_CLOCK=true` (the default), stop the host's own time sync first so the two don't fight over the clock:
```
systemctl disable --now systemd-timesyncd chrony ntp 2>/dev/null
```
```
cd ~/ntp-server && ./update.sh
```
`update.sh` builds, starts and verifies the server. `docker compose up -d --build` works too, but skips the verification and rollback.

## Update from GitHub (keeps your .env and data)

```
cd ~/ntp-server
./deploy-update.sh https://raw.githubusercontent.com/AngelGonePro/docker-compose-proxy/refs/heads/main/ntp-server.zip .
./update.sh
```
`deploy-update.sh` never touches `.env`, `data/` or `update.log`. It appends any new `.env` keys at the bottom for you to review, and leaves any file you hand-edited alone (it reports those instead). `update.sh` then rebuilds and verifies, and rolls back if the new version doesn't sync and serve.

To update the image only (Alpine/chrony security rebuilds within the pinned versions), with no new files from GitHub:
```
cd ~/ntp-server && ./update.sh
```

## Check it

```sh
docker compose exec chrony chronyc sources -v     # upstreams + selection
docker compose exec chrony chronyc tracking       # this server's accuracy
docker compose exec chrony chronyc -N authdata    # NTS status per source
docker compose exec chrony chronyc clients        # who's querying
docker compose exec chrony chronyc serverstats    # packets served/dropped
```

Docker's health status turns `healthy` only when chrony is synchronised **and** answers a real NTP query on its serving port.

## Notes

- **Ports:** UDP `NTP_PORT` inbound. Outbound: UDP 123 plus TCP 4460 (NTS key exchange).
- **Lowest jitter:** `NTP_NETWORK_MODE=host` skips Docker's NAT; needed for `NTP_HWTIMESTAMP=true`.
- **Clients behind one NAT IP** share a rate-limit bucket; raise `NTP_RATELIMIT_BURST` or set `NTP_RATELIMIT=false` if that applies.
- **Real-time priority** (`sched_priority`) works with `SYS_NICE` on cgroup v2 hosts (Debian 12 default). On cgroup v1 kernels with RT group scheduling, Docker gives containers no RT budget and chrony silently runs at normal priority. It still works fine either way.
- **Image updates:** `./update.sh` rebuilds at the pinned `ALPINE_VERSION`/`CHRONY_VERSION`, does nothing if no package or script changed, otherwise recreates and verifies, and on failure restores the previous image and sends `UPDATE_ALERT_WEBHOOK`. To move to a new release, bump the versions in `.env` and run it. Cron example: `17 4 * * 1 /root/ntp-server/update.sh >/dev/null 2>&1`.
