# Troubleshooting log

Real issues encountered while building this project, how they were diagnosed, and how they were fixed. Newest at the bottom.

## 1. API container crashed instantly: `npm start` — missing script

**Symptom.** `docker compose up` showed `bezkoder-api` exiting immediately; `docker compose logs bezkoder-api` printed `npm ERR! missing script: start`.

**Cause.** The upstream repo's API `package.json` had no `start` script, but its Dockerfile runs `CMD npm start`.

**Fix.** Added `"start": "node server.js"` to `bezkoder-api/package.json`.

## 2. `mysql:5.7` image would not run on Apple Silicon

**Symptom.** Compose failed to pull/run the database on an M-series Mac: no matching manifest for `linux/arm64`.

**Cause.** MySQL 5.7 images were never published for arm64.

**Fix.** Upgraded to `mysql:8.0`, which is multi-arch. Same image is used in production for parity.

## 3. Table `bezkoder_db.tutorials` doesn't exist — MySQL 8 first-boot race

**Symptom.** All API endpoints returned `Table 'bezkoder_db.tutorials' doesn't exist`, even though the API was gated on a MySQL healthcheck.

**Diagnosis.** `docker compose logs bezkoder-api` showed `SequelizeConnectionRefusedError: connect ECONNREFUSED` at startup — the schema sync (`sequelize.sync()`) had failed once and never retried, so the table was never created. The healthcheck (`mysqladmin ping -h localhost`) was passing **too early**: during MySQL 8's first-boot initialization a temporary server answers on the local socket while TCP networking is still disabled.

**Fix (both sides).**
- Healthcheck now pings `127.0.0.1` to force a TCP check: `mysqladmin ping -h 127.0.0.1`.
- `server.js` wraps the initial `sequelize.sync()` in a retry loop (10 attempts, 3 s apart) and exits non-zero if the DB never comes up, so `restart: unless-stopped` can take over.

## 4. UI Dockerfile: `EXPOSE $REACT_DOCKER_PORT` with an undefined variable

**Symptom.** Docker build warning/failure in the Nginx stage.

**Cause.** `REACT_DOCKER_PORT` was a compose-time variable, never defined as a build `ARG` in the final stage.

**Fix.** Hardcoded `EXPOSE 80` (the Nginx default we actually serve on).

## 5. Jenkins bootstrap failed: repository signing key 404

**Symptom.** First cloud-init run on the Jenkins EC2 instance failed; `/var/log/cloud-init-output.log` showed `curl: (22) The requested URL returned error: 404` for `https://pkg.jenkins.io/debian-lts/jenkins.io-2023.key`.

**Cause.** Jenkins rotates its Linux repository signing key roughly every three years; the 2023 key file most tutorials reference was removed after the December 2025 rotation.

**Fix.** Use the current key and repo path: `https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key`. The user-data script in `infra/` is updated accordingly.

## 6. Jenkins service crash-looped: Java 17 no longer supported

**Symptom.** `systemctl status jenkins` → failed; `journalctl -u jenkins` showed `Running with Java 17 ... Supported Java versions are: [21, 25]`.

**Cause.** Jenkins LTS ≥ 2.541 requires Java 21.

**Fix.** `apt install openjdk-21-jre-headless`, pointed the `java` alternative at 21, `systemctl reset-failed jenkins && systemctl restart jenkins`.

## 7. First push to main did not trigger a build (webhook OK)

**Symptom.** Merging to `main` produced no Jenkins build, yet GitHub's webhook deliveries page showed `200 OK` responses from `/github-webhook/`.

**Cause.** The "GitHub hook trigger for GITScm polling" trigger only fires for jobs Jenkins has built at least once — before the first run it has no SCM data to match the webhook payload against.

**Fix.** Triggered build #1 manually; subsequent pushes to `main` trigger automatically.

## Everyday debugging commands used

```bash
docker compose ps                     # container state + health
docker compose logs <service>         # service logs
docker exec -it <c> mysql -uroot -p   # poke the DB directly
sudo journalctl -u jenkins -e         # Jenkins service logs
sudo tail -f /var/log/cloud-init-output.log   # EC2 bootstrap progress
curl -v http://<host>/api/tutorials   # API reachability end-to-end
```
