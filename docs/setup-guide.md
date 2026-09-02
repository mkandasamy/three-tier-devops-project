# Setup guide — from zero to automated deployments

The exact steps used to build this environment. Assumes a Mac/Linux workstation with `git`, `aws` CLI, and Docker Desktop.

## 1. Fork and run locally

```bash
gh repo fork bezkoder/docker-compose-react-nodejs-mysql --clone --fork-name three-tier-devops-project
cd three-tier-devops-project
cp .env.example .env
docker compose up -d --build
open http://localhost:8888
```

## 2. AWS account and CLI

1. Create an AWS account (Free plan) and enable MFA on root.
2. Create an IAM user with `AdministratorAccess` and a CLI access key.
3. `aws configure` → key, secret, region `us-east-1`, output `json`.
4. Guardrail: `aws budgets create-budget` — $5/month with an 80% email alert (see git history for the exact JSON).

## 3. Provision the infrastructure

```bash
MY_IP=$(curl -s https://checkip.amazonaws.com) ./infra/provision.sh
```

Creates the `devops-key` key pair (saved to `~/.ssh/devops-key.pem`), the `jenkins-sg`/`app-sg` security groups, and two t3.micro instances bootstrapped by user-data:

- **jenkins-server** — 2 GB swap, Java 21, Jenkins LTS (2026 signing key), Docker
- **app-server** — 1 GB swap, Docker + Compose, `/opt/app` deploy directory

Wait for cloud-init to finish (`ssh ... cloud-init status --wait`), then create the production env file on the app server:

```bash
ssh -i ~/.ssh/devops-key.pem ubuntu@<APP_IP> 'cat > /opt/app/.env && chmod 600 /opt/app/.env' <<'ENV'
MYSQLDB_USER=root
MYSQLDB_ROOT_PASSWORD=<strong password>
MYSQLDB_DATABASE=bezkoder_db
CLIENT_ORIGIN=http://<APP_IP>
ENV
```

## 4. Configure Jenkins (one-time, web UI)

1. `http://<JENKINS_IP>:8080` → unlock with `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` → **Install suggested plugins** → create admin user.
2. Install plugins: **SSH Agent**, **Docker Pipeline**.
3. Credentials (Manage Jenkins → Credentials → Global):
   - `dockerhub-creds` — *Username with password*: Docker Hub username + a **Read & Write access token** (never the account password)
   - `app-server-ssh` — *SSH username with private key*: user `ubuntu`, key `~/.ssh/devops-key.pem`
4. Job: **New Item → Pipeline** named `three-tier-app-pipeline`:
   - Trigger: *GitHub hook trigger for GITScm polling*
   - Pipeline from SCM: this repo's URL, branch `*/main`, script path `Jenkinsfile`
5. GitHub repo → Settings → Webhooks → add `http://<JENKINS_IP>:8080/github-webhook/`, content type `application/json`, push events.
6. Run the job **once manually** (the hook trigger only arms after a first build), then every merge to `main` deploys automatically.

## 5. Stop / start for review sessions

```bash
# stop (NOT terminate) — data and configuration are preserved
aws ec2 stop-instances --ids <jenkins-id> <app-id>

# start again later
aws ec2 start-instances --ids <jenkins-id> <app-id>
```

After a start, public IPs have changed: update `APP_SERVER`/`APP_URL` in the Jenkinsfile, the GitHub webhook URL, and Jenkins' own configured URL if needed.
