# OVH VPS and Coolify runbook

This is the user-run procedure for the first Project Graphite host. It prepares one low-cost,
single-server environment for portfolio demonstrations. It is not highly available and does not
provide Kubernetes isolation.

No step in this runbook authorizes an automated agent to order resources, change billing, configure
OVH, change Cloudflare, or deploy an application. Review each external write before performing it.

## Target and cost boundary

| Item | Target |
| :--- | :--- |
| Provider | OVHcloud |
| Product | VPS-1 from the VPS 2027 range |
| CPU | 2 vCores |
| Memory | 4 GB |
| Storage | 40 GB NVMe |
| Operating system | Ubuntu 24.04 LTS |
| Runtime | self-hosted Coolify and Docker Compose |
| Published starting price | USD 4.54 per month |
| Hard checkout rule | stop if the final recurring total exceeds USD 5 before an explicit review |

OVH labels the price as starting at USD 4.54. Region, tax, currency, commitment, and optional items
can change the checkout total. Do not assume the product page is the invoice.

## 1. Review the order without purchasing

1. Open the OVHcloud VPS page and select **VPS-1** in the **VPS 2027** range.
2. On **Commitment period**, choose **No commitment** unless a different term and its complete
   charge have been explicitly approved.
3. On **Location**, choose an available standard European datacenter. Prefer France or Germany for
   latency from Egypt. Avoid **Local Zone** for the first host because its storage and feature set
   can differ.
4. On **Operating system**, choose **Ubuntu 24.04**. Do not select Windows, Plesk, cPanel, or a
   marketplace application.
5. Keep the included local storage. Do not add disks, additional IP addresses, premium backup,
   snapshots, licences, support plans, or managed databases.
6. The summary must still show 2 vCores, 4 GB RAM, 40 GB NVMe, and the included one-day automatic
   backup.
7. Stop at the final order summary. Record the recurring price, tax, one-time charge, renewal term,
   datacenter, and currency. Approve those exact values before selecting the purchase button.

## 2. Prepare SSH access

Generate a dedicated key on Windows PowerShell if one does not already exist:

```powershell
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\graphite-ovh" -C "graphite-ovh"
Get-Content "$env:USERPROFILE\.ssh\graphite-ovh.pub"
```

Protect the private key and its passphrase. Add only the `.pub` value when the OVH order or
installation screen offers **SSH key**. Do not paste the private key into OVH, GitHub, Coolify, a
repository, or chat.

After OVH reports the VPS as delivered, open **Bare Metal Cloud → Virtual private servers** and
select it. Record its IPv4 address and confirm its operating system is Ubuntu 24.04.

Connect from PowerShell using the username shown by OVH, normally `ubuntu`:

```powershell
ssh -i "$env:USERPROFILE\.ssh\graphite-ovh" ubuntu@VPS_IP
```

At the first connection, compare the displayed host fingerprint with the value in the OVH Control
Panel before accepting it.

## 3. Verify the fresh server

Run these read-only commands first:

```sh
cat /etc/os-release
nproc
free -h
df -h /
id
```

Expected results are Ubuntu 24.04 LTS, 2 processors, roughly 4 GB memory, roughly 40 GB root disk,
and a login user with `sudo` access. Stop if any material value differs from the approved order.

Install current security updates:

```sh
sudo apt update
sudo apt full-upgrade
sudo reboot
```

Reconnect only after SSH starts accepting connections again.

## 4. Add emergency swap

The server should not normally consume swap, but 2 GB reduces the chance that a temporary image
pull or deployment kills the host:

```sh
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
grep -q '^/swapfile ' /etc/fstab || echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
sudo swapon --show
```

The final output must show one `/swapfile` of 2 GB. Do not add more swap to hide sustained memory
pressure; stop an unused application or review a VPS resize instead.

## 5. Prepare the network boundary

Coolify needs these inbound TCP ports while it is being configured:

| Port | Source | Purpose |
| :--- | :--- | :--- |
| 22 | Amr's current public IP only when practical | SSH |
| 80 | anywhere | HTTP and certificate issuance |
| 443 | anywhere | HTTPS applications and dashboard |
| 8000 | Amr's current public IP during setup | initial Coolify dashboard |
| 6001 | Amr's current public IP during setup | dashboard real-time connection |
| 6002 | Amr's current public IP during setup | dashboard terminal |

In the OVH Control Panel, open **Network → Public IP Addresses**, enable **Advanced mode**, locate
the VPS IP, and inspect the **Firewall** column. If **Create Firewall** is offered, prepare only the
rules above and review them before enabling the firewall. Keep SSH allowed while testing; an
incorrect rule can lock out the server.

After the Coolify dashboard works through HTTPS, close public access to 8000, 6001, and 6002. Keep
22 restricted and keep 80 and 443 public. Do not publish PostgreSQL, Redis, Docker, or application
container ports directly.

## 6. Install Coolify

Coolify's supported automated installation for an Ubuntu LTS server is:

```sh
curl -fsSL https://cdn.coollabs.io/coolify/install.sh -o /tmp/install-coolify.sh
less /tmp/install-coolify.sh
sudo bash /tmp/install-coolify.sh
```

Reviewing the downloaded file separates retrieval from root execution. Wait for the installer to
print **Congratulations** and its dashboard URL.

Immediately open `http://VPS_IP:8000` from the allowed address and create the first administrator
account. The first visitor to an unclaimed installation can become its administrator, so do not
leave the registration screen unattended or publicly reachable.

In Coolify:

1. Open the user menu and configure multi-factor authentication if the installed release offers it.
2. Open **Servers → localhost → General** and confirm validation succeeds.
3. Open **Servers → localhost → Docker Cleanup** and keep scheduled cleanup enabled.
4. Do not add another server, build server, or paid integration.

Copy `/data/coolify/source/.env` to a protected location outside the VPS. It contains Coolify's
`APP_KEY`, which is required to decrypt saved secrets after restoration. Never commit or print it in
an issue, workflow log, or chat.

## 7. Put the dashboard behind its hostname

Prepare these external changes and review them before saving:

1. In Cloudflare, open `project-graphite.com` and select **DNS → Records → Add record**.
2. Choose type **A**, name `coolify`, content the VPS IPv4, and initially set **Proxy status** to
   **DNS only**.
3. In Coolify, open **Settings** and set the instance domain to
   `https://coolify.project-graphite.com`.
4. Save, wait for the proxy and certificate to become healthy, then open the HTTPS address in a new
   browser session.
5. Confirm login works through the hostname before closing ports 8000, 6001, and 6002 at the
   provider firewall.

Do not expose the dashboard on the apex domain. Do not create a wildcard DNS record.

## 8. Verify deployment image access

After the Message Lab workflow has published its first immutable image tag, open the
**Project-Graphite** organisation on GitHub and select **Packages**. Check both
`graphite-message-lab/web` and `graphite-message-lab/worker`.

For this public demonstration app, each package must be publicly readable so the VPS does not need
a long-lived GitHub token. If a package is private, open **Package settings**, select **Change
visibility**, review the package name, and change it to **Public**. Changing package visibility is a
separate external write; review it when the package exists. Do not make private project images
public to reuse this setup.

Confirm anonymous access from a signed-out browser or a shell that is not signed in to GHCR:

```sh
docker manifest inspect ghcr.io/project-graphite/graphite-message-lab/web:IMAGE_TAG
docker manifest inspect ghcr.io/project-graphite/graphite-message-lab/worker:IMAGE_TAG
```

Replace `IMAGE_TAG` with the exact `sha-...` tag from the successful workflow. Both commands must
return a manifest before continuing.

## 9. Add Graphite Message Lab without deploying it

In Coolify:

1. Open **Projects → Add** and name the project `Graphite`.
2. Add a production environment named `production`.
3. Inside that environment, select **+ New**.
4. Choose **Public Repository** and enter
   `https://github.com/Project-Graphite/graphite-message-lab`.
5. Select **Check Repository**.
6. Under **Configuration → General**, set **Build Pack** to **Docker Compose**.
7. Set **Branch** to `main`.
8. Set **Base Directory** to `/`.
9. Set **Docker Compose Location** to `/compose.production.yaml`.
10. Save and reload the Compose definition.

Confirm the parsed services are `postgres`, `redis`, `web`, and `worker`. The web and worker must
use GHCR images; Coolify must not build them on the VPS. PostgreSQL and Redis must have named
volumes. Only `web` may receive a public domain.

## 10. Enter Message Lab configuration

Open the application's **Environment Variables** page. Coolify derives these fields from the
Compose file:

| Name | Value |
| :--- | :--- |
| `IMAGE_TAG` | an approved immutable `sha-...` tag, or `latest` only for the first disposable test |
| `ALLOWED_HOSTS` | `message-lab.project-graphite.com` |
| `CSRF_TRUSTED_ORIGINS` | `https://message-lab.project-graphite.com` |
| `POSTGRES_PASSWORD` | a newly generated random value |
| `SECRET_KEY` | a separately generated random value |

Generate the two secrets locally or with a password manager. Do not reuse them and do not place
their values in the platform registry, Compose file, shell history, screenshots, or chat.

For the `web` service, set **Domains** to:

```text
https://message-lab.project-graphite.com:8000
```

The `:8000` suffix tells the Coolify proxy which internal container port to use; visitors still use
normal HTTPS on port 443. Do not add domains to PostgreSQL, Redis, or the worker.

## 11. Prepare the public application hostname

In Cloudflare, prepare an **A** record named `message-lab` pointing to the VPS IPv4. Use
**DNS only** for the first certificate and application test. After HTTPS is healthy, Cloudflare
proxying can be enabled and verified separately.

Do not change the apex records used by Graphite Hub. Do not expose ports 5432, 6379, or 8000 at the
VPS firewall; Coolify routes the public hostname through 443.

## 12. Review before the first deployment

Before selecting **Deploy**, confirm:

- the exact repository, branch, and Compose file are correct;
- required secrets show as configured without revealing their values;
- the image tag exists in public GHCR;
- each service has a memory and CPU limit;
- PostgreSQL and Redis use persistent named volumes;
- only the web service has a domain;
- no service publishes a host port;
- VPS memory and disk have sufficient free capacity.

Selecting **Deploy** creates containers, networks, and persistent volumes on the VPS and is a
separate external mutation. Review and approve that deployment at the time it is performed.

## 13. Acceptance checks after an approved deployment

In Coolify, open **Deployments** and wait for success. Then check each service under **Logs**.
PostgreSQL and Redis should become healthy, the web startup should complete migrations, and the
worker should connect to Redis.

Open `https://message-lab.project-graphite.com`, submit a non-sensitive message, and confirm it
moves from queued to processed. Refresh the page and confirm it persists.

Read-only server checks are:

```sh
free -h
df -h /
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
docker stats --no-stream
```

Target steady-state use is below 75% RAM and below 70% disk. Stop adding projects if swap grows
continuously, an application is OOM-killed, or free disk approaches 10 GB.

## 14. Backups

The included OVH backup retains only the previous daily server image. It does not replace logical
database backups.

In Coolify, open the PostgreSQL service, select **Backups**, and add a daily schedule. Keep seven
small local dumps initially and select **Backup Now**. Verify the execution is successful and the
file size is non-zero. A backup is not proven until it has been restored into a disposable database
and the application can read it.

Adding S3-compatible storage, paid retention, or another backup product is a separate billing and
credential decision. Prepare its exact provider, capacity, permissions, and expected price before
enabling it.

## 15. Capacity rules for a 4 GB VPS

- Keep Graphite Hub and other static frontends on Cloudflare Pages.
- Build images in GitHub Actions and pull them from GHCR.
- Prefer one shared PostgreSQL and Valkey service when later projects can be safely separated by
  database, role, and credentials.
- Keep each small web or worker container near 128-384 MB when the application permits it.
- Do not install Kubernetes, Elasticsearch, a service mesh, Prometheus, or a log aggregation stack.
- Do not resize the VPS without a current resource report, the exact new recurring price, and
  explicit approval.

## 16. Rollback and removal boundaries

Changing `IMAGE_TAG` to a previously verified immutable tag and redeploying is the normal
application rollback. A database migration can make image rollback unsafe, so inspect migration
compatibility first.

Deleting a Coolify application, volume, database, DNS record, backup, or VPS can permanently remove
data. Before any removal, identify the exact resource, export required state, verify the export,
show the destructive action, and obtain separate approval.

## Sources

- [OVHcloud VPS plans](https://us.ovhcloud.com/vps/)
- [Coolify self-hosted installation](https://coolify.io/docs/start-with-self-hosted)
- [Coolify firewall requirements](https://coolify.io/docs/core/infrastructure/servers/firewall)
- [Coolify Docker Compose applications](https://coolify.io/docs/applications/builds/docker-compose)
- [Coolify database backups](https://coolify.io/docs/databases/backups)
- [OVHcloud Network Security Dashboard](https://help.ovhcloud.com/csm/en-au-network-security-dashboard?id=kb_article_view&sysparm_article=KB0060692)
