# Google Cloud message lab

This runbook prepares the disposable Graphite Message Lab on Google Cloud. It is intentionally a
manual checklist: no Google Cloud resource should be created, changed or billed until Amr reviews
the final screen and selects **Create**.

## Cost boundary

Use project `project-c7674ac5-1dc5-4723-97e`. Before provisioning, open **Billing > Overview** and
confirm that the project is attached to the intended Cloud Billing account and that the trial
credit is active. Create a budget alert if useful, but remember that a budget sends notifications;
it does not cap or stop spending.

The proposed `e2-micro`, 30 GB standard persistent disk and qualifying outbound traffic are within
Google Cloud's documented Free Tier limits when they run in an eligible US region. An external IPv4
address is billed separately at the current list price of USD 0.005 per hour on a standard VM,
about USD 3.65 for 730 hours. Trial credit can cover that charge, but it is not permanently free.
Review the estimate shown by Google before creating the VM because prices can change.

## Proposed resources

Create only these resources:

| Resource | Value |
| :--- | :--- |
| VPC | `graphite-lab` custom-mode network |
| Subnet | `graphite-lab-us-central1`, region `us-central1`, IPv4 range `10.20.0.0/24` |
| Firewall rule | ingress TCP 22 from `35.235.240.0/20`, target tag `iap-ssh` |
| VM | `graphite-lab-1`, zone `us-central1-a`, machine type `e2-micro` |
| Boot disk | Ubuntu 24.04 LTS, 30 GB `pd-standard`, delete with VM |
| Network | subnet above, ephemeral external IPv4, network tag `iap-ssh` |
| Service account | none |

Enable Shielded VM secure boot, vTPM and integrity monitoring. Enable OS Login in project or VM
metadata. Do not enable the HTTP or HTTPS firewall shortcuts and do not open port 8000. Connect with
SSH through Identity-Aware Proxy (IAP), which requires TCP 22 from Google's
`35.235.240.0/20` IAP range.

The external IPv4 keeps package downloads, GitHub Container Registry and Cloudflare Tunnel simple
for the first experiment. A no-IPv4 design needs IPv6 support throughout or Cloud NAT, which adds
moving parts and can add cost; it is not worthwhile for this disposable test.

## IAM and APIs

In the console, enable only the APIs requested by the VM and IAP workflows. Give the human who will
connect these roles if their existing project role does not already include them:

- IAP-secured Tunnel User;
- Compute OS Login, or Compute OS Admin Login when `sudo` is required.

Do not attach a service account to the VM. The application does not call Google Cloud APIs.

## Host preparation

After reviewing and creating the VM, connect with **SSH > Open in browser window** using IAP. Then
run the following on the host. Review Docker's current Ubuntu installation instructions before
installing Docker Engine and the Compose plugin; do not use an unreviewed convenience script.

Create 2 GB of swap because the complete stack has little headroom on a 1 GB VM:

```sh
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

After Docker is installed:

```sh
sudo install -d -o "$USER" -g "$USER" /opt/graphite
cd /opt/graphite
git clone https://github.com/Project-Graphite/graphite-message-lab.git
cd graphite-message-lab
cp deploy.env.example deploy.env
chmod 600 deploy.env
```

Edit `deploy.env`, replace every placeholder and use the immutable `sha-<commit>` image tag created
by GitHub Actions. Never paste those secret values into an issue, commit or chat.

## Start and verify

```sh
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml pull
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml --profile tools run --rm migrate
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml up -d web worker postgres redis
curl --fail http://127.0.0.1:8000/up
curl --fail http://127.0.0.1:8000/up/databases
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml ps
docker stats --no-stream
free -h
```

In Cloudflare Zero Trust, create a remotely managed tunnel and public hostname
`message-lab.project-graphite.com` pointing to `http://localhost:8000`. Install `cloudflared` and the
tunnel service using the exact command Cloudflare shows. This is a separate manual action and does
not require a public application port on the VM.

Submit a message in the browser. Confirm that it changes from `queued` to `processed`, restart the
Compose services, and confirm the message remains. Record memory and disk usage while the worker is
active. This profile proves the deployment path; it is not a production sizing recommendation.

## Remove the experiment

When testing is complete, remove the Cloudflare hostname and tunnel, then delete `graphite-lab-1` and
confirm its boot disk is also selected for deletion. Delete the lab firewall rule, subnet and VPC if
nothing else uses them. Finally, verify the Google Cloud Billing report no longer shows accumulating
Compute Engine, persistent disk or external IPv4 usage.

## References

- [Google Cloud Free Tier](https://docs.cloud.google.com/free/docs/free-cloud-features)
- [IAP TCP forwarding](https://docs.cloud.google.com/iap/docs/using-tcp-forwarding)
- [VPC pricing](https://cloud.google.com/vpc/network-pricing)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
