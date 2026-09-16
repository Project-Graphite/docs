# Google Cloud GKE runbook

This is the user-run procedure for the shared Project Graphite cluster. It does not authorize
automation or another operator to create, change, or delete Google Cloud resources.

## Cost boundary

The initial target is a zonal GKE Standard cluster in `us-central1-a` with one `e2-small` node and a
30 GB `pd-standard` boot disk. At the prices reviewed on 2026-09-16, plan for approximately USD
18-20 per month and treat USD 25 per month as a hard review ceiling:

| Item | Planning estimate |
| :--- | ---: |
| One `e2-small` node | USD 12.23/month |
| 30 GB standard disk | about USD 1.20/month |
| One external IPv4 address | about USD 3.65/month |
| Small persistent volumes | about USD 0.40-1.00/month |
| Zonal GKE management fee | covered by the monthly USD 74.40 GKE free-tier credit while eligible |

Trial credit is temporary and does not make a resource free. Check the console estimate before
creation and stop if it exceeds USD 25 per month. Do not enable Cloud NAT, a load balancer,
multi-zone nodes, managed databases, or autoscaling under this plan.

Recheck the official [GKE pricing](https://cloud.google.com/kubernetes-engine/pricing) and
[Compute Engine pricing](https://cloud.google.com/products/compute/pricing) before creation because
rates and free-tier terms can change.

The node has about 1.4 GiB allocatable after Kubernetes reservations. It can run the system pods
and one modest full-stack project. Multiple namespaces are supported, but multiple always-on
full-stack projects require a separately approved capacity increase.

## Fixed names

```powershell
$ProjectId = "project-c7674ac5-1dc5-4723-97e"
$Region = "us-central1"
$Zone = "us-central1-a"
$Network = "graphite-lab"
$Subnet = "graphite-lab-us-central1"
$Cluster = "graphite-lab"
$NodeServiceAccount = "graphite-gke-nodes@$ProjectId.iam.gserviceaccount.com"
```

Run every read-only check first and confirm that the selected project ID is correct:

```powershell
gcloud config set project $ProjectId
gcloud config get-value project
gcloud billing projects describe $ProjectId
gcloud compute instances describe graphite-lab-1 --zone=$Zone
gcloud compute networks subnets describe $Subnet --region=$Region
```

## Remove the abandoned VM path

The VPC and subnet are reused by GKE. Delete only the old VM and its IAP SSH firewall rule. First
stop the VM, confirm the application was never installed and no data needs to be retained, then run
the deletion commands yourself:

```powershell
gcloud compute instances stop graphite-lab-1 --zone=$Zone
gcloud compute instances describe graphite-lab-1 --zone=$Zone
gcloud compute instances delete graphite-lab-1 --zone=$Zone --delete-disks=all
gcloud compute firewall-rules delete graphite-lab-allow-iap-ssh
```

Do not delete `$Network` or `$Subnet`.

## Prepare the project

Review these exact changes in the console before running them. They enable the GKE API, reserve a
Pod range on the existing subnet, and create a least-privilege node service account:

```powershell
gcloud services enable container.googleapis.com
gcloud compute networks subnets update $Subnet --region=$Region --add-secondary-ranges=graphite-pods=10.21.0.0/20
gcloud iam service-accounts create graphite-gke-nodes --display-name="Graphite GKE nodes"
gcloud projects add-iam-policy-binding $ProjectId --member="serviceAccount:$NodeServiceAccount" --role="roles/container.defaultNodeServiceAccount"
```

Verify the subnet before creating the cluster:

```powershell
gcloud compute networks subnets describe $Subnet --region=$Region --format="yaml(name,network,ipCidrRange,secondaryIpRanges)"
```

## Create the cluster

This is the cost-incurring step. Confirm the console estimate is within the USD 25 ceiling, then
run it yourself:

```powershell
gcloud container clusters create $Cluster `
  --zone=$Zone `
  --network=$Network `
  --subnetwork=$Subnet `
  --enable-ip-alias `
  --cluster-secondary-range-name=graphite-pods `
  --services-ipv4-cidr=34.118.224.0/20 `
  --enable-dataplane-v2 `
  --workload-pool="$ProjectId.svc.id.goog" `
  --release-channel=regular `
  --machine-type=e2-small `
  --num-nodes=1 `
  --disk-type=pd-standard `
  --disk-size=30 `
  --service-account=$NodeServiceAccount `
  --enable-shielded-nodes `
  --shielded-secure-boot `
  --shielded-integrity-monitoring `
  --enable-autorepair `
  --enable-autoupgrade `
  --logging=SYSTEM `
  --monitoring=SYSTEM
```

No public Kubernetes Service is created. The node has an external IP so it can pull images and
reach Cloudflare without the added cost of Cloud NAT. Dataplane V2 enforces NetworkPolicy.

## Connect and verify

```powershell
gcloud components install gke-gcloud-auth-plugin kubectl
gcloud container clusters get-credentials $Cluster --zone=$Zone
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node | Select-String -Pattern "Allocatable" -Context 0,8
```

Cluster access is read-only by default. Review generated resources before any apply.

## Render and bootstrap

From a clone of the private `platform` repository:

```powershell
python -m pip install pyyaml jsonschema
python scripts/validate.py
python scripts/render_kubernetes.py .rendered
kubectl apply --dry-run=server -f .rendered/cluster/bootstrap.yaml
kubectl apply --dry-run=server -f .rendered/cluster/workloads.yaml
kubectl apply --dry-run=server -f .rendered/graphite-message-lab/bootstrap.yaml
kubectl apply --dry-run=server -f .rendered/graphite-message-lab/workloads.yaml
```

Inspect the dry-run output. Applying any of these files is a separate cluster write and requires a
specific approval at that time.

## Create secrets

Create a remotely managed Cloudflare Tunnel in the Cloudflare dashboard. Configure the public
hostname `message-lab.project-graphite.com` to use this internal origin:

```text
http://graphite-gateway.graphite-system.svc.cluster.local:8080
```

Do not paste secret values into chat or source control. Use PowerShell prompts or a local password
manager to populate these Kubernetes Secrets:

```powershell
$TunnelToken = Read-Host "Cloudflare tunnel token" -AsSecureString
$SecretKey = Read-Host "Django secret key" -AsSecureString
$PostgresPassword = Read-Host "Postgres password" -AsSecureString
```

Convert each value only within the local command that creates its Secret, then clear the variables.
Apply the reviewed cluster and project bootstrap resources first. Then create the required Secret
objects before applying either workloads file. The required objects and keys are:

| Namespace | Secret | Key |
| :--- | :--- | :--- |
| `graphite-system` | `cloudflare-tunnel` | `TUNNEL_TOKEN` |
| `graphite-message-lab` | `graphite-message-lab-secrets` | `SECRET_KEY`, `POSTGRES_PASSWORD` |

Creating Secrets and applying resources are intentional cluster mutations. Prepare the commands
locally, confirm the target context and namespace, and approve them separately.

## Acceptance checks

After an approved apply:

```powershell
kubectl get pods -A
kubectl get pvc -n graphite-message-lab
kubectl get networkpolicy -n graphite-message-lab
kubectl logs -n graphite-message-lab job/migration-REPLACE_WITH_RENDERED_NAME
kubectl rollout status -n graphite-message-lab deployment/web
kubectl rollout status -n graphite-message-lab deployment/worker
```

Open `https://message-lab.project-graphite.com`, submit a message, confirm it becomes processed,
restart the web pod, and confirm the message remains.

## Cost and cleanup checks

Check billing daily during the experiment. If the projected monthly cost passes USD 25, stop and
review before resizing anything.

Deleting the cluster, persistent disks, tunnel, or secrets destroys state. Prepare and approve
those cleanup operations separately; they are not implied by this runbook.
