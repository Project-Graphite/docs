# OVH VPS and Coolify runbook

This is the completed Project Graphite procedure from VPS purchase through automatic application
redeployment. It is also the recovery and onboarding reference for the current host.

The host is a small, single-server portfolio environment. It is not highly available, and every
application shares one Docker host. Do not place untrusted workloads on it.

## Current deployment

| Item | Current value |
| :--- | :--- |
| Provider | OVHcloud |
| Product class | VPS-1 |
| Approved recurring checkout price | USD 5.35 |
| CPU | 2 vCores |
| Memory | 4 GB advertised, approximately 3.7 GiB visible |
| Storage | 40 GB advertised, approximately 38 GB root filesystem |
| Operating system | Ubuntu 24.04.4 LTS |
| Hostname | `vps-2283122d` |
| Public IPv4 | `135.125.254.59` |
| Emergency swap | 2 GiB |
| Runtime | Docker Compose managed by self-hosted Coolify |
| Coolify dashboard | `https://coolify.project-graphite.com` |
| Test application | `https://message-lab.project-graphite.com` |

The accepted USD 5.35 checkout was a billing decision made by Amr. Before renewing, resizing, or
adding an OVH option, verify the current invoice currency, tax, term, and renewal price. Never infer
approval for a new charge from this historical purchase.

## 1. Buy the VPS

1. Open the OVHcloud VPS catalogue and select the 4 GB VPS-1 option.
2. Choose Ubuntu 24.04 LTS.
3. Keep the standard CPU, RAM, and local disk.
4. Do not add another IP, control panel licence, managed database, extra disk, or paid backup unless
   its exact recurring cost has been reviewed.
5. Review the final order page instead of relying on the advertised starting price.
6. Confirm the final recurring amount, tax, currency, term, location, and optional items.
7. Purchase only after those exact values are accepted.
8. Wait for OVH to finish provisioning before attempting SSH.

The completed order was accepted at USD 5.35 for the 2-vCore, 4-GB, 40-GB configuration.

## 2. Create and register the SSH key

From Windows PowerShell, create a dedicated key:

~~~powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\graphite-ovh" -C "graphite-ovh"
~~~

Confirm both files exist:

~~~powershell
Test-Path "$env:USERPROFILE\.ssh\graphite-ovh"
Test-Path "$env:USERPROFILE\.ssh\graphite-ovh.pub"
ssh-keygen -lf "$env:USERPROFILE\.ssh\graphite-ovh.pub"
~~~

Add only `graphite-ovh.pub` to OVH. Never upload or paste the private key. If the VPS was installed
without the correct key, use OVH's reinstall or console recovery path to add the public key.

Connect with:

~~~powershell
ssh -i "$env:USERPROFILE\.ssh\graphite-ovh" ubuntu@135.125.254.59
~~~

At the first connection, compare the host fingerprint with the value shown by OVH before accepting
it. The prompt should become `ubuntu@vps-2283122d`.

## 3. Verify and update Ubuntu

Run:

~~~sh
cat /etc/os-release
nproc
free -h
df -h /
id
~~~

The completed host reported Ubuntu 24.04.4 LTS, 2 CPUs, approximately 3.7 GiB RAM, approximately
38 GB storage, and the non-root `ubuntu` user.

Install updates and reboot:

~~~sh
sudo apt update
sudo apt full-upgrade
sudo reboot
~~~

Reconnect with the same SSH command after the server returns.

## 4. Add emergency swap

Create one 2-GiB swap file:

~~~sh
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
grep -q '^/swapfile ' /etc/fstab || echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
sudo swapon --show
~~~

The output must show `/swapfile` with 2 GiB. Swap is an emergency buffer, not extra working memory.
Sustained swap use requires stopping a workload or reviewing capacity.

If pasted terminal text starts with characters such as `^[[200~`, cancel the command and paste it
again as plain text. Those characters are bracketed-paste control sequences.

## 5. Configure the OVH edge firewall

In the OVH Control Panel:

1. Open **Network**.
2. Open **Public IP Addresses**.
3. Locate `135.125.254.59`.
4. Open its edge firewall controls. The UI used during setup did not show an **Advanced mode**
   switch, so do not wait for one.
5. Create allow rules before the final deny rule.
6. Enable the completed rules and verify SSH in a second session before closing the first.

The final public boundary is:

| Protocol | Port | Source | Purpose |
| :--- | :--- | :--- | :--- |
| TCP | 22 | Amr's current public IP when practical | SSH |
| TCP | 80 | Any | HTTP redirect and certificate validation |
| TCP | 443 | Any | HTTPS |
| UDP | 443 | Any | HTTP/3 handled by the Coolify proxy |
| Any | Any | Any | Deny remaining inbound traffic |

Ports 6001, 6002, 8000, and 8080 must not be reachable from the internet after setup. PostgreSQL
5432, Redis 6379, and Docker must never be published publicly.

When changing the SSH source address, add and verify the replacement allow rule before removing the
old one.

## 6. Install Coolify

Confirm outbound access:

~~~sh
getent hosts github.com
curl -I https://cdn.coollabs.io
~~~

Download, inspect, and run the supported installer:

~~~sh
curl -fsSL https://cdn.coollabs.io/coolify/install.sh -o /tmp/install-coolify.sh
less /tmp/install-coolify.sh
sudo bash /tmp/install-coolify.sh
~~~

Press `q` to exit `less`. Continue only after the installer prints its success message.

Open `http://135.125.254.59:8000`, immediately create the first Coolify administrator, and finish
the initial setup. Use a password that satisfies Coolify's displayed requirements.

## 7. Validate this machine as the Coolify server

During setup, choose **This machine**.

If validation reports:

~~~text
root@host.docker.internal: Permission denied (publickey)
~~~

copy the public key shown by Coolify into root's SSH authorization file:

~~~sh
sudo install -d -m 700 /root/.ssh
sudoedit /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
~~~

Add the public key as one complete line, save, return to Coolify, and validate again. Compare the
fingerprint shown by Coolify before authorizing it. Do not copy a private key.

## 8. Put Coolify behind its domain

In Cloudflare, create:

| Type | Name | Content | TTL | Proxy status |
| :--- | :--- | :--- | :--- | :--- |
| A | `coolify` | `135.125.254.59` | 300 or Auto | DNS only during setup |

In Coolify's global **Settings**, set:

~~~text
https://coolify.project-graphite.com
~~~

Save, wait for the certificate, and sign in through HTTPS before blocking direct dashboard ports.

## 9. Remove direct Coolify port exposure

Create `/data/coolify/source/docker-compose.custom.yml`:

~~~yaml
services:
  coolify:
    ports: !reset []
  soketi:
    ports: !reset []
~~~

Apply it from the directory containing Coolify's real `.env` and base Compose file:

~~~sh
cd /data/coolify/source
sudo docker compose --env-file .env \
  -f docker-compose.yml \
  -f docker-compose.custom.yml \
  up -d
~~~

Do not run the command from `/home/ubuntu`; it will look for the wrong files.

Verify:

~~~sh
sudo ss -lntup
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
~~~

The `coolify` and real-time containers must not publish 8000, 6001, or 6002. The
`coolify-proxy` owns public 80 and 443. It may bind 8080 locally, but the OVH firewall blocks it.

## 10. Harden and maintain Coolify

The completed settings are:

- administrator registration disabled after the first account was created;
- two-factor authentication enabled;
- API access enabled only for the authenticated GitHub deployment webhook;
- Coolify update check daily;
- Coolify automatic update Sunday at 04:00;
- Docker cleanup Sunday at 05:00;
- cleanup does not delete volumes or networks;
- Coolify instance backup daily at 03:00;
- seven local instance backups retained;
- a manual instance backup completed and was downloaded.

Keep a protected copy of `/data/coolify/source/.env` outside the VPS. Its application key is needed
to decrypt restored Coolify secrets. Never commit or paste that file.

A Coolify instance backup does not replace a logical PostgreSQL backup. Application database backup
and restore testing remain separate operational work.

## 11. Publish deployable images

Message Lab builds in GitHub Actions so the VPS does not compile application images. The workflow
publishes:

~~~text
ghcr.io/project-graphite/graphite-message-lab/web:sha-<commit>
ghcr.io/project-graphite/graphite-message-lab/worker:sha-<commit>
ghcr.io/project-graphite/graphite-message-lab/web:main
ghcr.io/project-graphite/graphite-message-lab/worker:main
~~~

SHA tags are immutable rollback points. The `main` tags move only after tests pass.

Both Message Lab packages are public so Coolify can pull them without a GitHub package token:

~~~sh
sudo docker manifest inspect ghcr.io/project-graphite/graphite-message-lab/web:main
sudo docker manifest inspect ghcr.io/project-graphite/graphite-message-lab/worker:main
~~~

Private projects should use a narrowly scoped registry credential instead.

## 12. Create Graphite Message Lab in Coolify

1. Open **Projects** and create or open **Graphite**.
2. Open its production environment.
3. Select **+ New**.
4. Choose **Public Repository**.
5. Enter `https://github.com/Project-Graphite/graphite-message-lab`.
6. Select branch `main`.
7. Choose Docker Compose.
8. Set the Compose file to `/compose.production.yaml`.
9. Save and use **Reload Compose** after source-definition changes.

The parsed services are `web`, `worker`, `postgres`, and `redis`. The application must show
two persistent storages. Only `web` receives a public domain.

## 13. Configure Message Lab

Under **Environment Variables**, set:

| Name | Value |
| :--- | :--- |
| `IMAGE_TAG` | `main` |
| `ALLOWED_HOSTS` | `message-lab.project-graphite.com,localhost` |
| `CSRF_TRUSTED_ORIGINS` | `https://message-lab.project-graphite.com` |
| `POSTGRES_PASSWORD` | unique generated secret |
| `SECRET_KEY` | separate unique generated secret |

Keep secret values only in Coolify. Do not place them in GitHub, Compose, screenshots, shell
history, documentation, or chat.

Set the `web` service domain to:

~~~text
https://message-lab.project-graphite.com:8000
~~~

The suffix identifies the container's internal port. Visitors still connect to HTTPS 443.

## 14. Create application DNS and issue TLS

In Cloudflare, create:

| Type | Name | Content | TTL | Proxy status |
| :--- | :--- | :--- | :--- | :--- |
| A | `message-lab` | `135.125.254.59` | 300 | DNS only |

Confirm resolution:

~~~sh
getent hosts message-lab.project-graphite.com
~~~

Deploy from Coolify and wait for every service to become healthy. Coolify obtains the Let's Encrypt
certificate after DNS resolves and ports 80 and 443 answer.

The first deployment exposed two corrections:

- the host Compose validator rejected an empty worker `entrypoint`, so production Compose now
  relies on the image entrypoint and sets the Celery command;
- the health check calls `localhost`, so `ALLOWED_HOSTS` includes the hostname and `localhost`.

A temporary **Not secure** warning is acceptable only while the certificate is being issued. The
finished site must return HTTP 200 with a valid certificate.

## 15. Enable automatic deployments

Keep Coolify's application **Auto Deploy** switch off. GitHub Actions controls the order so Coolify
cannot deploy before tests and images finish.

### Enable API access

1. Open global **Settings**.
2. Open **Configuration → Advanced**.
3. Enable **API Access**.
4. Save.

### Create a deploy-only token

1. Open **Keys & Tokens → API Tokens**.
2. Select **Add**.
3. Name the token for GitHub deployment.
4. Grant only `deploy`.
5. Choose a finite expiration.
6. Create it and copy it once into a password manager.

### Copy the resource webhook

1. Open the Graphite Message Lab Compose application.
2. Open its **Settings** or configuration sidebar.
3. Open **Webhooks**.
4. Copy **Deploy Webhook (auth required)**.

### Store GitHub secrets

In the Message Lab repository, open **Settings → Secrets and variables → Actions** and create:

| Secret | Value |
| :--- | :--- |
| `COOLIFY_WEBHOOK` | authenticated resource deploy webhook URL |
| `COOLIFY_TOKEN` | deploy-only Coolify API token |

Never store either value as a normal variable or in workflow YAML.

### Deployment order

For each push to `main`, `.github/workflows/main.yml`:

1. runs the application test suite;
2. stops if tests fail;
3. builds the web and worker image;
4. publishes immutable `sha-<commit>` tags;
5. updates both `main` tags;
6. records immutable deployment metadata when enabled;
7. sends an authenticated `POST` request to the Coolify webhook.

Use `POST`; this installation returned HTTP 405 for `GET`.

Coolify then reads the current Compose source, pulls the new `main` images, runs the web startup
migration, waits for health, and replaces the containers. A successful webhook response only means
the deployment was queued; confirm completion under **Deployments**.

## 16. Verify the completed system

Check externally:

~~~sh
curl -I https://coolify.project-graphite.com
curl -I https://message-lab.project-graphite.com
curl https://message-lab.project-graphite.com/api/messages/
~~~

The site must return HTTP 200. The public endpoint must return only administrator-approved messages.

Check the VPS:

~~~sh
free -h
df -h /
sudo swapon --show
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
sudo docker stats --no-stream
~~~

For Message Lab, verify private submissions do not appear publicly, publication requests await
approval, the worker processes messages, and the administrator can publish, unpublish, search,
filter, and delete.

Create an administrator interactively. Identify the web container:

~~~sh
sudo docker ps --format 'table {{.Names}}\t{{.Image}}' | grep graphite-message-lab
~~~

Then run:

~~~sh
sudo docker exec -it CONTAINER_NAME python manage.py createsuperuser --username USERNAME
~~~

Enter the password only at Django's hidden prompt. Django stores a salted hash in PostgreSQL. Never
put an administrator password into Compose, Git, documentation, or a command argument.

## 17. Roll back

1. Identify the last known-good `sha-...` image tag.
2. Check whether a database migration prevents image rollback.
3. Change Coolify's `IMAGE_TAG` from `main` to that exact SHA tag.
4. Select **Deploy**, not Restart.
5. Verify health and the public site.
6. Set `IMAGE_TAG` back to `main` and deploy once to resume automatic updates.

Never roll back a database by deleting volumes.

## 18. Add another small project

1. Keep it in its own repository.
2. Add production Compose that pulls prebuilt images.
3. Add health checks, restart policies, resource limits, and named volumes.
4. Publish SHA and `main` tags only after tests.
5. Create a separate Coolify application and network.
6. Store runtime secrets only in Coolify.
7. Attach a unique subdomain only to the public service.
8. Create a Cloudflare DNS-only A record to the VPS.
9. Use a resource-specific webhook and deploy-only token.
10. Verify backup, restore, rollback, memory, disk, TLS, and background work.

Do not add Kubernetes, a service mesh, Elasticsearch, or a monitoring stack to this host. Keep
static sites on Cloudflare Pages.

## 19. Capacity and billing boundaries

Review capacity before another project when steady memory exceeds 75%, swap grows, a container is
OOM-killed, free disk approaches 10 GB, or builds occur on the VPS.

A resize, additional IP, paid backup, object storage, managed database, or second server is a new
billing decision. Prepare the exact resource, recurring total, tax, and downtime, then obtain
approval before changing it.

## 20. Destructive boundaries

Deleting a Coolify application, Docker volume, database, DNS record, backup, token, or VPS can
remove data or access. Identify the exact resource, export and verify required state, show the
deletion and impact, and obtain separate approval.

## References

- [OVHcloud VPS plans](https://us.ovhcloud.com/vps/)
- [Coolify self-hosted installation](https://coolify.io/docs/start-with-self-hosted)
- [Coolify firewall requirements](https://coolify.io/docs/core/infrastructure/servers/firewall)
- [Coolify Docker Compose applications](https://coolify.io/docs/applications/builds/docker-compose)
- [Coolify GitHub Actions deployment](https://coolify.io/docs/applications/sources/github/actions)
- [Coolify deployment webhooks](https://coolify.io/docs/core/automation/deploy-webhooks)
- [Coolify database backups](https://coolify.io/docs/databases/backups)
- [OVHcloud Network Security Dashboard](https://help.ovhcloud.com/csm/en-au-network-security-dashboard?id=kb_article_view&sysparm_article=KB0060692)
