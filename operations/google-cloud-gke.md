# Google Cloud GKE runbook

This is the complete user-run procedure for the shared Project Graphite cluster. Google Cloud and
Cloudflare changes are performed manually in their web consoles. Local commands are used only to
connect to Kubernetes, review generated resources, create Secrets, and deploy workloads.

No operator or automation is authorized by this document to create, resize, apply, or delete cloud
resources. Confirm the exact settings and displayed estimate before clicking a create or delete
button.

## Target and cost boundary

| Setting | Value |
| :--- | :--- |
| Project ID | `project-c7674ac5-1dc5-4723-97e` |
| Cluster | `graphite-lab` |
| Mode | GKE Standard |
| Location | zonal, `us-central1-a` |
| Release channel | Regular |
| Nodes | one `e2-small` |
| Node image | Container-Optimized OS with containerd |
| Boot disk | 30 GB `pd-standard` |
| VPC | `graphite-lab` |
| Subnet | `graphite-lab-us-central1` |
| Pod range | `graphite-pods`, `10.21.0.0/20` |
| Service range | `graphite-services`, `10.22.0.0/24` |
| Planned monthly cost | approximately USD 18-20 |
| Review ceiling | USD 25 per month |

The USD 10 budget already configured is an early warning, not a spending cap. Google Cloud budgets
do not automatically stop resources. Trial credit is temporary and does not make a resource free.

At the prices reviewed on 2026-09-16, the plan consists of approximately USD 12.23 per month for
the node, USD 1.20 for its standard boot disk, USD 3.65 for its external IPv4 address, and less than
USD 1 for the initial persistent volumes. The monthly USD 74.40 GKE free-tier credit covers the
management fee for one eligible zonal Standard cluster, not its compute resources.

Recheck the official [GKE pricing](https://cloud.google.com/kubernetes-engine/pricing) and
[Compute Engine pricing](https://cloud.google.com/products/compute/pricing) before creation. Stop
if the Console estimate exceeds USD 25 per month.

Do not add Cloud NAT, a cloud load balancer, multi-zone nodes, autoscaling, Spot nodes, a managed
database, or another node under this plan. The initial node can run the Kubernetes system pods and
one modest full-stack project. Multiple namespaces are supported, but multiple always-on stacks
need a separate capacity and cost decision.

## 1. Confirm the project

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Open the project selector in the top bar.
3. Select the project whose ID is `project-c7674ac5-1dc5-4723-97e`.
4. Keep the project ID visible while making every change below.
5. Open **Billing → Budgets & alerts** and confirm the existing USD 10 budget is active.
6. Do not change billing, upgrade the billing account, or create a second budget as part of this
   runbook.

## 2. Create the VPC network

1. Open **VPC network → VPC networks**.
2. Click **Create VPC network**.
3. Enter `graphite-lab` as the network name.
4. Set **Subnet creation mode** to **Custom**.
5. Under **New subnet**, enter:

   | Field | Value |
   | :--- | :--- |
   | Name | `graphite-lab-us-central1` |
   | Region | `us-central1` |
   | IP stack type | IPv4 (single-stack) |
   | Primary IPv4 range | `10.20.0.0/24` |

6. In that subnet's **Secondary IPv4 ranges**, click **Create secondary IP range** and enter the Pod
   range:

   | Field | Value |
   | :--- | :--- |
   | Range name | `graphite-pods` |
   | IPv4 range | `10.21.0.0/20` |

7. Click **Add IP range** and enter the Service range:

   | Field | Value |
   | :--- | :--- |
   | Range name | `graphite-services` |
   | IPv4 range | `10.22.0.0/24` |

8. Leave **Private Google Access** off. GKE enables what its public nodes require during cluster
   creation.
9. Leave **VPC Flow Logs** off.
10. Do not add another subnet.
11. In **Firewall rules**, leave every suggested rule unchecked. Do not create SSH, RDP, ICMP, or
    broad internal-access rules. GKE creates the firewall rules its cluster requires.
12. Leave **Dynamic routing mode** set to **Regional**.
13. Leave the MTU at its default value and leave IPv6 disabled.
14. Click **Create**.
15. Reopen `graphite-lab` and confirm:

    - subnet mode is Custom;
    - `graphite-lab-us-central1` is in `us-central1`;
    - its primary range is `10.20.0.0/24`;
    - its secondary ranges are `graphite-pods` with `10.21.0.0/20` and `graphite-services` with
      `10.22.0.0/24`;
    - no user-created firewall rule was added.

Google documents custom networks and secondary ranges under
[Create and modify VPC networks](https://cloud.google.com/vpc/docs/create-modify-vpc-networks).

## 3. Enable the Kubernetes Engine API

1. Open **APIs & Services → Library**.
2. Search for **Kubernetes Engine API**.
3. Open the result named **Kubernetes Engine API** with service name
   `container.googleapis.com`.
4. Click **Enable**.
5. Wait until its page shows **API enabled** or offers a **Manage** button.

Enabling the API does not create a cluster. Do not enable unrelated APIs.

## 4. Create the node service account

GKE recommends a dedicated, minimally privileged node service account instead of the Compute
Engine default account.

1. Open **IAM & Admin → Service Accounts**.
2. Click **Create service account**.
3. Enter:

   | Field | Value |
   | :--- | :--- |
   | Service account name | `Graphite GKE nodes` |
   | Service account ID | `graphite-gke-nodes` |
   | Description | `Node identity for the Project Graphite GKE cluster` |

4. Click **Create and continue**.
5. Under **Grant this service account access to project**, add exactly
   **Kubernetes Engine Default Node Service Account**. Its role ID is
   `roles/container.defaultNodeServiceAccount`.
6. Do not add Editor, Owner, Storage Admin, or Service Account Token Creator.
7. Click **Continue**.
8. Leave the optional service-account users and administrators fields empty.
9. Click **Done**.
10. Confirm the resulting email is
    `graphite-gke-nodes@project-c7674ac5-1dc5-4723-97e.iam.gserviceaccount.com`.

Google documents this minimum node role and custom-account recommendation in
[GKE service accounts](https://cloud.google.com/kubernetes-engine/docs/how-to/service-accounts).

## 5. Configure the cluster

This section prepares the cost-incurring resource. Do not click **Create** until every section and
the displayed estimate have been reviewed.

1. Open **Kubernetes Engine → Clusters**.
2. Click **Create**.
3. Choose **GKE Standard**. If the page initially presents Autopilot, select **Switch to Standard
   cluster** or **Configure** under Standard.

### Cluster basics

Set:

| Field | Value |
| :--- | :--- |
| Name | `graphite-lab` |
| Location type | Zonal |
| Zone | `us-central1-a` |
| Release channel | Regular |

- Do not select additional node locations.
- Do not register the cluster to a fleet.
- Leave the control-plane version on the Regular channel default.

### Networking

Open **Cluster → Networking** in the left side of the creation form. Work from top to bottom.

#### Control Plane Access

The screen first shows checkboxes for DNS and IPv4 access:

| On-screen option | Selection |
| :--- | :--- |
| Access using DNS | Checked |
| Enable Kubernetes tokens via DNS | Unchecked |
| Enable Kubernetes certificates via DNS | Unchecked |
| Access using IPv4 addresses | Unchecked |

Using only the DNS endpoint allows local `gcloud` and `kubectl` access with Google IAM credentials
without exposing the Kubernetes API on a public IPv4 endpoint. When IPv4 access is unchecked,
authorized-network and control-plane IP firewall options are not needed.

#### Cluster networking

The next group controls node connectivity and VPC selection:

| On-screen option | Selection |
| :--- | :--- |
| Enable private nodes | Unchecked |
| Network | `graphite-lab` |
| Node subnet | `graphite-lab-us-central1` |
| IP stack type | IPv4 |

The single node needs outbound internet access to pull public GHCR images and connect to Cloudflare.
Leaving **Enable private nodes** unchecked gives it an external IP and avoids the additional Cloud
NAT design and cost. This does not expose a Kubernetes application Service.

#### Advanced networking options

Expand **Advanced networking options** if the range fields are hidden, then set:

| On-screen option | Selection |
| :--- | :--- |
| Enable VPC-native traffic routing (uses alias IP) | Checked |
| Automatically create secondary ranges | Unchecked |
| Pod secondary CIDR range | `graphite-pods` |
| Services secondary CIDR range | `graphite-services` |
| Maximum Pods per node | Leave the displayed default |

Both dropdown values must already exist on `graphite-lab-us-central1`. If
`graphite-services` is missing, stop without creating the cluster, open the subnet in another tab,
and add `graphite-services` as `10.22.0.0/24`.

#### Dataplane and optional networking features

Continue down the Networking screen:

| On-screen option | Selection |
| :--- | :--- |
| Enable Dataplane V2 | Checked |
| Enable Kubernetes Network Policy | Leave disabled; Dataplane V2 provides it |
| Enable Dataplane V2 metrics | Unchecked |
| Enable Dataplane V2 flow observability | Unchecked |
| Enable intra-node visibility | Unchecked |
| Enable L4 internal load balancer subsetting | Unchecked |
| Enable multi-networking | Unchecked |
| Enable FQDN network policy | Unchecked |
| Enable Gateway API | Unchecked |

If separate DNS settings appear, keep the default cluster DNS provider and default DNS scope. Do
not enable Cloud DNS for GKE, NodeLocal DNSCache, or custom DNS settings for this lab.

Dataplane V2 is required because the generated project isolation relies on Kubernetes
NetworkPolicy. See [Using GKE Dataplane V2](https://cloud.google.com/kubernetes-engine/docs/how-to/dataplane-v2).

### Security

Open the cluster **Security** section:

- Enable **Workload Identity Federation for GKE**.
- Enable **Shielded GKE nodes**.
- Leave legacy authentication and client certificate options disabled.
- Do not enable Google Groups for RBAC, Binary Authorization, or managed service integrations for
  this lab.

### Observability

Open **Observability**:

- Keep system-component logging enabled.
- Keep system-component monitoring enabled.
- Disable workload logging for the initial cost-bounded lab.
- Do not enable control-plane logs, managed Prometheus, application metrics, or advanced
  observability yet.

Application logs remain available through `kubectl logs`.

## 6. Configure the default node pool

In the left navigation of the cluster form, open **Node Pools → default-pool**.

### Node pool details

| Field | Value |
| :--- | :--- |
| Name | `default-pool` |
| Number of nodes | `1` |
| Node locations | only `us-central1-a` |
| Autoscaling | Disabled |
| Auto-upgrade | Enabled |
| Auto-repair | Enabled |

Do not enable Spot VMs, queued provisioning, or multiple zones.

### Nodes

Open **Nodes** and set:

| Field | Value |
| :--- | :--- |
| Image type | Container-Optimized OS with containerd |
| Machine family | General purpose |
| Series | E2 |
| Machine type | `e2-small` |
| Boot disk type | Standard persistent disk |
| Boot disk size | `30 GB` |

Leave GPUs, local SSDs, sole tenancy, confidential nodes, and custom machine settings disabled.

### Node security

Open the node pool's **Security** or **Advanced settings → Security** section:

1. Set **Service account** to `graphite-gke-nodes`.
2. Keep **Secure Boot** enabled.
3. Keep **Integrity monitoring** enabled.
4. Leave the default access scopes unchanged.
5. Do not add SSH keys or additional IAM roles.

The node service account cannot be changed on an existing node pool, so verify it before creation.

## 7. Review and create

1. Review the summary and estimated monthly cost.
2. Confirm it describes one `e2-small` node in one zone with a 30 GB standard disk.
3. Confirm autoscaling and additional zones are disabled.
4. Confirm no load balancer, Cloud NAT, or managed database is listed.
5. Stop and review if the projected recurring cost exceeds USD 25 per month.
6. Click **Create** only after explicitly accepting that estimate.
7. Wait for the cluster status to become green or **Running**.

Google's current Console procedure is documented in
[Creating a zonal cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-zonal-cluster).

## 8. Verify the cluster in the Console

Open `graphite-lab` under **Kubernetes Engine → Clusters** and confirm:

- type is Standard and location is `us-central1-a`;
- total nodes is one;
- machine type is `e2-small`;
- release channel is Regular;
- Dataplane V2 and Workload Identity Federation are enabled;
- network and subnet are the Graphite values;
- no external load balancer exists.

Open **Billing → Reports** and filter to the current project. Record the current cost and check it
daily during the experiment.

## 9. Connect from Windows PowerShell

The remaining deployment steps require `kubectl`; the Google Cloud Console does not render the
repository's Kubernetes resources.

1. In **Kubernetes Engine → Clusters**, find `graphite-lab`.
2. Click its action menu and choose **Connect**.
3. Copy the displayed `gcloud container clusters get-credentials` command, but do not run it until
   the local components below are installed.
4. Open a new PowerShell window and run:

```powershell
gcloud components install gke-gcloud-auth-plugin kubectl
gcloud container clusters get-credentials graphite-lab --zone=us-central1-a --project=project-c7674ac5-1dc5-4723-97e --dns-endpoint
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node | Select-String -Pattern "Allocatable" -Context 0,8
```

Confirm the context names project `project-c7674ac5-1dc5-4723-97e`, cluster `graphite-lab`, and zone
`us-central1-a`. Stop if it points anywhere else.

## 10. Render and review the Kubernetes resources

From PowerShell:

```powershell
Set-Location C:\Projects\Project-Graphite\platform
python -m pip install pyyaml jsonschema
python scripts/validate.py
python scripts/test_render_kubernetes.py
python scripts/render_kubernetes.py .rendered
kubectl apply --dry-run=client -f .rendered/cluster/bootstrap.yaml
kubectl apply --dry-run=client -f .rendered/cluster/workloads.yaml
kubectl apply --dry-run=client -f .rendered/graphite-message-lab/bootstrap.yaml
kubectl apply --dry-run=client -f .rendered/graphite-message-lab/workloads.yaml
```

The dry runs must not contain a LoadBalancer or NodePort Service. Review the generated files before
applying them. Applying resources is a separate cluster mutation.

## 11. Create the namespaces and policies

After reviewing the dry-run output and confirming the Kubernetes context:

```powershell
kubectl apply -f .rendered/cluster/bootstrap.yaml
kubectl apply -f .rendered/graphite-message-lab/bootstrap.yaml
kubectl get namespace graphite-system graphite-message-lab
kubectl get resourcequota,limitrange,networkpolicy -n graphite-message-lab
```

This creates namespaces, quotas, configuration, and isolation policies. It does not deploy the
applications or create secret values.

## 12. Create the Cloudflare Tunnel in the UI

1. Sign in to the [Cloudflare dashboard](https://dash.cloudflare.com/).
2. Select the account that owns `project-graphite.com`.
3. Open **Networking → Tunnels**.
4. Click **Create a tunnel**.
5. Choose **Cloudflared** if prompted for a connector type.
6. Name it `graphite-gke`.
7. Click **Save tunnel** or **Create tunnel**.
8. On the connector setup page, choose **Docker** only to display the token-based command.
9. Copy only the tunnel token into your password manager. Do not run the Docker command and do not
   place the token in Git, chat, screenshots, shell history, or documentation.
10. Leave this browser tab open. The connector becomes healthy after the Kubernetes workload is
    deployed.

Cloudflare's current dashboard flow is documented in
[Create a tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/).

## 13. Create Kubernetes Secrets

The following PowerShell prompts do not echo the tunnel token. Django and PostgreSQL values are
generated locally and are not printed:

```powershell
$TunnelToken = [System.Net.NetworkCredential]::new("", (Read-Host "Cloudflare tunnel token" -AsSecureString)).Password
$DjangoSecret = python -c "import secrets; print(secrets.token_urlsafe(64))"
$PostgresPassword = python -c "import secrets; print(secrets.token_urlsafe(32))"

kubectl create secret generic cloudflare-tunnel `
  --namespace=graphite-system `
  --from-literal="TUNNEL_TOKEN=$TunnelToken"

kubectl create secret generic graphite-message-lab-secrets `
  --namespace=graphite-message-lab `
  --from-literal="SECRET_KEY=$DjangoSecret" `
  --from-literal="POSTGRES_PASSWORD=$PostgresPassword"

$TunnelToken = $null
$DjangoSecret = $null
$PostgresPassword = $null
Remove-Variable TunnelToken,DjangoSecret,PostgresPassword
```

Confirm only the objects and key names, not their values:

```powershell
kubectl get secret cloudflare-tunnel -n graphite-system
kubectl describe secret graphite-message-lab-secrets -n graphite-message-lab
```

The required contract is:

| Namespace | Secret | Keys |
| :--- | :--- | :--- |
| `graphite-system` | `cloudflare-tunnel` | `TUNNEL_TOKEN` |
| `graphite-message-lab` | `graphite-message-lab-secrets` | `SECRET_KEY`, `POSTGRES_PASSWORD` |

## 14. Deploy the internal gateway and tunnel

```powershell
kubectl apply -f .rendered/cluster/workloads.yaml
kubectl rollout status deployment/graphite-gateway -n graphite-system
kubectl rollout status deployment/cloudflared -n graphite-system
kubectl get pods,service -n graphite-system
```

There should be one internal `ClusterIP` gateway Service and no public load balancer.

Return to **Cloudflare → Networking → Tunnels**. Wait for `graphite-gke` to report **Healthy**.

## 15. Add the Message Lab public route in Cloudflare

1. Open the `graphite-gke` tunnel.
2. Open **Routes**.
3. Click **Add route → Published application**.
4. Enter:

   | Field | Value |
   | :--- | :--- |
   | Subdomain | `message-lab` |
   | Domain | `project-graphite.com` |
   | Path | leave empty |
   | Service type | HTTP |
   | Service URL | `http://graphite-gateway.graphite-system.svc.cluster.local:8080` |

5. Save the route.

The resulting hostname is `message-lab.project-graphite.com`. Do not create a wildcard hostname or
change the Hub's apex-domain records.

## 16. Deploy Message Lab

```powershell
kubectl apply -f .rendered/graphite-message-lab/workloads.yaml
kubectl get pods,pvc,service -n graphite-message-lab
kubectl rollout status statefulset/postgres -n graphite-message-lab
kubectl rollout status statefulset/redis -n graphite-message-lab
kubectl rollout status deployment/web -n graphite-message-lab
kubectl rollout status deployment/worker -n graphite-message-lab

$MigrationJob = kubectl get jobs -n graphite-message-lab -l app.kubernetes.io/name=migration -o jsonpath='{.items[0].metadata.name}'
kubectl wait --for=condition=complete "job/$MigrationJob" -n graphite-message-lab --timeout=180s
kubectl logs "job/$MigrationJob" -n graphite-message-lab
```

If a workload is not ready, inspect it without changing anything:

```powershell
kubectl get events -n graphite-message-lab --sort-by=.lastTimestamp
kubectl describe pod -n graphite-message-lab -l app.kubernetes.io/name=web
kubectl logs deployment/web -n graphite-message-lab
kubectl logs deployment/worker -n graphite-message-lab
```

Do not resize the node or loosen security policies to make a failing workload start.

## 17. Acceptance test

1. Open `https://message-lab.project-graphite.com`.
2. Submit a non-sensitive test message.
3. Confirm it changes from queued to processed.
4. Refresh and confirm the message remains.
5. Delete the web Pod and let its Deployment replace it:

   ```powershell
   kubectl delete pod -n graphite-message-lab -l app.kubernetes.io/name=web
   kubectl rollout status deployment/web -n graphite-message-lab
   ```

6. Refresh again and confirm the message remains in PostgreSQL.
7. Confirm cluster isolation remains present:

   ```powershell
   kubectl get networkpolicy -n graphite-message-lab
   kubectl get resourcequota,limitrange -n graphite-message-lab
   kubectl get service -A
   ```

No Service should have type `LoadBalancer` or `NodePort`.

## 18. Ongoing checks

- Check **Billing → Reports** daily during the experiment.
- Treat the USD 10 budget email as an early warning.
- Stop and review if projected recurring cost approaches USD 25 per month.
- Check `kubectl top` only after confirming Metrics Server is available; do not enable paid
  observability solely for this lab.
- Do not add projects whose combined requests exceed the node's allocatable CPU or memory.
- Backups are not yet configured, so Message Lab data remains disposable.

## 19. Cleanup procedure

Cleanup is destructive and is not implied by cluster creation. Preserve any required data first.

1. In Cloudflare, remove the `message-lab.project-graphite.com` published application route.
2. Delete the `graphite-gke` tunnel only after its connector is no longer needed.
3. In Google Cloud Console, open **Kubernetes Engine → Clusters**.
4. Select only `graphite-lab` and click **Delete**.
5. Confirm the exact cluster name and deletion.
6. Open **Compute Engine → Disks** and inspect whether any persistent data disks remain.
7. Do not delete a remaining disk until its contents and retention decision are understood.
8. Open **IAM & Admin → Service Accounts** and remove `graphite-gke-nodes` only when no cluster or
   node pool uses it.
9. Keep `graphite-lab`, `graphite-lab-us-central1`, and `graphite-pods` unless the entire lab network
   is being deliberately retired.
10. Confirm billing reports stop accumulating GKE node, disk, and external IPv4 charges.

Deleting a cluster, persistent disk, tunnel, Secret, or namespace can permanently destroy state.
Review and approve each deletion separately at cleanup time.
