# Homelab AI-admin work log

## 2026-11-13 — Baseline (read-only inspection, VM 201 = ai-admin, 192.168.100.224)
- Proxmox pve01 PVE 9.2.21; API token ai-monitor@pve!ai-admin = Audit-only (read-only). Guests: 100 lxc docker, 101 homeassistant, 200 Windows11, 201 ai-admin.
- HA 192.168.100.214, v2026.11.1. Token in .env is ADMIN-level (config_entries readable). Integrations: frigate, mqtt. Cameras: camera.front_door, camera.front_door_fluent.
- LXC 100 = 192.168.100.6. ssh lxc100-frigate -> Permission denied (setup script not yet deployed).
- Frigate :5000 reachable unauthenticated from VM 201 (HTTP 200) -> guard not installed. :8971 = 401 (auth).
- Local scripts: ~/ai-admin/deploy/lxc100-setup.sh, lxc100-port5000-guard.sh (reviewed, not yet run).

## 2026-11-13 — Session 2: user authorized root login to pve01 (password given in chat; NOT stored anywhere)
Plan: steps 1 (deploy setup + guard) and 2 (add read-only `stats` subcommand). HA user change pending user choice. Camera details pending.
- pve01 = 192.168.100.5 (no DNS name). Logged in as root via password (ssh ControlMaster socket /tmp/.pve_cm). PVE 9.2.21, kernel 7.0.14-25-pve.
- LXC 100 "docker": unprivileged, nesting=1, 8 cores/8G. USB bind /dev/bus/usb (Coral). iptables nf_tables backend; DOCKER-USER chain exists (empty).
- Frigate container "frigate" (ghcr.io/blakeblackshear/frigate:stable), network "media" (bridge, 172.19.0.9), ports 5000/8971 published 0.0.0.0.
  Config: /opt/docker/frigate/config/config.yml. Env var names present: FRIGATE_RTSP_PASSWORD. `--validate-config` supported.
- Only client on :5000 observed = HA 192.168.100.214 -> guard with HA_IP=192.168.100.214 is safe.
- Edited deploy/lxc100-setup.sh: added read-only `stats` subcommand (orig saved: notes/lxc100-setup.sh.orig-2026-11-13).
- pct snapshot 100 FAILED (bind mounts -> 'snapshot feature is not available'). Fallback: config copy /opt/docker/frigate/config/config.yml.pre-ai-admin-20261113; undo = lxc100-setup.sh --uninstall / guard uninstall.
- lxc100-setup.sh deployed OK. aiadmin user created; ssh lxc100-frigate status/stats/show-config verified; 'id' and chained cmds denied. Coral inference 7.3ms; Frigate 0.18.0-77a66e7; existing camera front_door = 192.168.100.225 (Reolink-style paths), creds are LITERAL in config (not env placeholders), MQTT broker 192.168.100.214:1883 user frigate_mqtt.
- Port guard installed (systemd frigate-port-guard.service, DOCKER-USER rule DROP !192.168.100.214 ctorigdstport 5000; ip6 drop all).
  Verified: VM201->:5000 blocked (000), :8971 still 401, HA still connected to :5000, HA frigate+mqtt config entries 'loaded', camera.front_door recording.
  Revert: pct exec 100 -- systemctl stop frigate-port-guard  (or /root/lxc100-port5000-guard.sh uninstall)
- Root session to pve01 closed; temp askpass removed. Steps 1+2 DONE.

## Open items
- Waiting on user: new camera details (make/model, IP, RTSP main/sub paths, ONVIF?, objects, zones, retention).
- Waiting on user: HA token decision (dedicated non-owner ai-admin user vs keep current admin token).
- Recommend: change pve01 root password (it was shared in chat).
- Recommend: move front_door RTSP creds + MQTT password in config.yml to {FRIGATE_*} env placeholders.
- No PVE snapshot possible for LXC 100 (bind mounts) -> use vzdump or config backups before big changes.

## 2026-11-13 — New camera probe (read-only)
- 192.168.100.50: Dahua N42BJ62, fw 2.800.0000000.16.R build 2020-04-18. Ports 80, 554, 37777 open; ONVIF answers at /onvif/device_service.
- Login admin/<user-supplied pw> verified (1 attempt, Dahua HTTP digest API). Password NOT stored.
- Main: 2560x1440 H.264 30fps 4096kbps (subtype=0). Sub: 704x480 H.264 30fps 1024kbps (subtype=1).
- RTSP path: /cam/realmonitor?channel=1&subtype=0|1
- Pending: camera name, how to inject password (env var + container recreate vs literal), objects/zones/retention.

## 2026-11-13 — Add camera "driveway" (option A: env var)
- Frigate deployed by compose: /opt/docker/frigate/docker-compose.yml (project frigate, external network media, privileged, USB + renderD128).
- Running image == local :stable tag (sha256:9678a83a...) -> recreate does NOT upgrade.
- Backup: docker-compose.yml.pre-ai-admin-20261113. Added env_file ./frigate-secrets.env (root 600) holding FRIGATE_DRIVEWAY_PASSWORD.
- First validate FAILED correctly: global review.alerts.required_zones [front_approach, front_porch] inherited by driveway. Fix: driveway review.alerts.required_zones: []. (Note: Frigate exits rc=0 in safe mode — helper's grep check is what catches it.) reset_admin_password=false confirmed.
- APPLIED 07:34:54. Backup: /var/lib/frigate-admin/backups/config.20261113T073454.pre-apply.yml (undo: ssh lxc100-frigate rollback).
- driveway: camera_fps 5.0, detection running; Coral 7.42ms. HA Frigate integration reloaded via API -> 16 driveway entities, camera.driveway recording. MQTT sensors 'off' (not unavailable) => MQTT OK.
- Retention inherited from global: alerts 7d (active_objects), detections 0d, continuous 0d. Alerts = any person (no zones yet).
- Root session closed, temp files removed.
## Open items (updated)
- Zones for driveway (need user to describe areas / or draw in Frigate UI).
- Consider: move front_door + MQTT literal creds to frigate-secrets.env placeholders.
- Consider: frigate-admin `stage` takes a full file; since show-config is redacted, edits with literal creds need root. Moving all creds to env would let aiadmin path handle everything.
- Rotate pve01 root + camera passwords (shared in chat). HA token decision still pending.
- Camera firmware 2020-04 — check for Dahua update.

## 2026-11-14 — GitHub repo smith-scot/homelab-frigate (private)
- Deploy key ~/.ssh/id_ed25519_github_homelab_frigate (repo-scoped, write). SSH alias: github-homelab-frigate.
- ~/ai-admin is the git repo. .gitignore is a WHITELIST (only NOTES.md + .gitignore tracked) so .env/secrets/ can never be committed.
- Commit identity: smith-scot <47512570+smith-scot@users.noreply.github.com>.
