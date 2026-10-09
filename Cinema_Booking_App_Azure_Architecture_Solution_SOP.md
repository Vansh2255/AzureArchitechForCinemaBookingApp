# SOP: how the Cinema Booking Azure architecture was built

Scope: the private Azure design for https://github.com/Gorang1093/Cinema-Booking/.  
Region and account boundary: resource group `leetamRG`, East US.  
This is the process that was actually followed. It is not a procedure for recreating a chat.

Later deployment facts override earlier plans. The app and API behavior still come from the Cinema Booking API review. Custom domains `www.tatt3r.com` and `api.tatt3r.com` were proposed only. They were not verified.

## 1. Purpose

Put the cinema booking app on Azure so that:

- Users reach only Azure Front Door.
- The React frontend stays in a storage account with public network access disabled.
- The API, workers, PostgreSQL, and Redis stay on a private AKS cluster.
- The container registry stays private.
- The admin Linux VM has no public IP.
- Admin access is Point-to-Site VPN, then SSH to the VM private IP.

## 2. Starting point

Already present before this build:

- Resource group `leetamRG`
- VNet `leetamVNET`
- A workload subnet, `subnet1`
- A Windows machine for the VPN client and certificate work
- The backend repo, Node.js 22, Express 5, PostgreSQL, Prisma, Redis, BullMQ, Stripe, JWT

The frontend is a separate Vite/React app. It was built on the admin VM and uploaded to storage. It is not part of the backend repo.

## 3. Design rule that drove the order

Nothing that users do not need was given a public IP.

That forced this order:

1. Admin path first, or the private VM cannot be reached.
2. Outbound NAT second, or the private VM cannot install tools or call Azure.
3. Private registry and private cluster next.
4. Front Door last, because it is the only intended public entry and it depends on private origins that already exist.

Do not start from a fresh VNet. Do not change the AKS VNet. Do not delete the peering used by the admin VM.

## 4. Target architecture

```text
Browser
  |
  | HTTPS
  v
Azure Front Door Premium
  |
  +-- frontend route --> Private Link --> leetamstrgacc ($web, React)
  |
  +-- backend endpoint --> Private Link --> leetam-cinema-backend-pls
                                |
                                v
                         Internal load balancer 10.224.0.7:5000
                                |
                                v
                         cinema-backend pod on leetamAKS
                                |
                    +-----------+-----------+
                    v                       v
              PostgreSQL pod            Redis pod

Admin path, separate from user traffic:

Windows
  | P2S VPN, profile leetamVNET, client 172.16.201.2
  v
VPN gateway on leetamVNET
  |
  v
leetamVM 10.0.0.4:22   (no public IP)
  |
  +-- build/push --> leetamcr.azurecr.io (private endpoint 10.0.0.7)
  +-- kubectl -----> leetamAKS through VNet peering
  +-- upload ------> leetamstrgacc

Outbound from subnet1: leetamNATgtwy
```

Public entry is Front Door only. Storage, ACR, AKS, and the VM stay private.

## 5. Phase 1 — Admin VM and P2S VPN

Goal: a Linux admin VM with no public IP, reachable from Windows by VPN.

1. Keep the existing resource group, VNet, and `subnet1`. Do not rebuild them.
2. Add a subnet named exactly `GatewaySubnet` on `leetamVNET`. Do not put an NSG on it. Do not place the VM there.
3. Create the VPN gateway in `leetamRG`, same region as the VNet, gateway type VPN. Wait until provisioning finishes before configuring P2S. This step takes about 30–45 minutes.
4. Create `leetamVM` in `subnet1` with no public IP. The private IP seen in the successful test was `10.0.0.4`.
5. Open SSH (TCP 22) only from the VNet / P2S client range, not from the internet.
6. Configure P2S on the gateway with certificate authentication.
   - Root certificate: `AzureP2SRootCert`
   - Client certificate: `AzureP2SClientCert`
   - Generate both on the Windows machine. One client certificate can be used by one machine; each additional machine needs its own client certificate signed by the same root.
   - Export the root public data into the gateway. Install the client certificate in the Windows user certificate store.
7. Download the VPN client package. Use the Azure VPN folder, not Generic or OpenVPN, with the Azure VPN Client.
8. Import the profile. The working profile name was `leetamVNET`.
9. Connect, then test from Windows:

```powershell
Test-NetConnection 10.0.0.4 -Port 22
```

Pass condition from the project: `TcpTestSucceeded : True`, client address `172.16.201.2`, interface `leetamVNET`.

10. SSH as `azureuser` to `10.0.0.4`. Do not SSH to a public IP. There is not one.

## 6. Phase 2 — NAT gateway for outbound only

Goal: let the private subnet reach the internet without giving the VM an inbound public IP.

The VM had no public IP, so `apt`, Docker pulls, and Azure management calls failed until outbound NAT existed.

1. Create the NAT public IP.
2. Create NAT gateway `leetamNATgtwy` in `leetamRG`.
3. Attach the public IP to the NAT gateway.
4. Do not attach the subnet in the first wizard pass if that pass is still creating the gateway.
5. After the gateway exists, attach `leetamNATgtwy` to `subnet1`.
6. Wait until provisioning state is `Succeeded`.
7. From the VM, confirm outbound access, then confirm the VM managed identity:

```bash
az login --identity
```

NAT is outbound only. It does not make `leetamVM` reachable from the internet.

## 7. Phase 3 — Private ACR

Goal: store images where AKS and the admin VM can pull them, with ACR public access disabled.

1. Use registry `leetamcr` / `leetamcr.azurecr.io` in `leetamRG`.
2. Disable public network access on the registry.
3. Create a private endpoint in `leetamVNET`.
   - Registry private IP seen later: `10.0.0.7`
   - Data endpoint private IP seen later: `10.0.0.6`
4. Use private DNS zone `privatelink.azurecr.io`. Link it to `leetamVNET` and, after AKS exists, to the AKS VNet. A private endpoint without the DNS link is not enough.
5. Give the VM managed identity `AcrPush` and `AcrPull` on `leetamcr`. Give the AKS kubelet identity `AcrPull`.
6. On the VM, refresh the identity session before testing:

```bash
az logout
az login --identity
az acr login --name leetamcr
```

7. Verify DNS and reachability from the VM. A `401` from `https://leetamcr.azurecr.io/v2/` means the private path works and only auth remains.

## 8. Phase 4 — Private AKS and peering

Goal: run the API where it has no public service, and reach it from the admin VM.

1. Create `leetamAKS` in `leetamRG`, East US. Azure created VNet `aks-vnet-29219636`. Do not replace that VNet later.
2. Peer `leetamVNET` and `aks-vnet-29219636` both ways. The peering is what lets the admin VM run `kubectl` and what lets AKS reach the ACR private endpoint. Do not delete it.
3. Link `privatelink.azurecr.io` to the AKS VNet as well as `leetamVNET`.
4. Get credentials as `azureuser`. Do not use `sudo kubectl`. Root does not have the kubeconfig.
5. Check node architecture before building images:

```bash
kubectl get nodes -o wide
```

The nodes in this project were ARM64. An amd64-only image will not run.

## 9. Phase 5 — Data stores and backend on AKS

The early plan was Azure Database for PostgreSQL Flexible Server and Azure Cache for Redis. The verified deployment used pods in the cluster instead.

1. Namespace: `cinema`.
2. PostgreSQL: StatefulSet with a persistent volume on an Azure managed disk, about 20 GiB. Set `PGDATA` to a subdirectory such as `/var/lib/postgresql/data/pgdata`. The disk mount contains `lost+found`, and PostgreSQL will not initialize on that directory itself.
3. Use the StatefulSet-created PVC. An extra hand-made `postgres-pvc` stayed `Pending` and was not the volume in use.
4. Redis: deployment plus ClusterIP service. Test from a temporary pod with `redis-cli -h redis ping`. Pass result: `PONG`.
5. Build one multi-platform image from the VM and push it:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t leetamcr.azurecr.io/cinema-backend:v2 \
  --push .
```

`cinema-backend:v2` is the image that matched the ARM64 nodes. `az aks check-acr` can fail with `exec format error` on ARM64 even when the real pull works. Confirm with an actual pod pull.

6. Run the API as `cinema-backend`. The app also has email and cleanup workers. Keep them off the request path.
7. Expose the API with an internal load balancer, not a public one:

```yaml
spec:
  type: LoadBalancer
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

The internal frontend in this project was `10.224.0.7:5000`. ClusterIP is not enough for Front Door.

8. Create Private Link Service `leetam-cinema-backend-pls` on that internal load balancer.
9. Put secrets in `cinema-backend-secret`: `DATABASE_URL`, JWT secrets, Stripe keys, Redis URL, mail settings, `CORS_ORIGIN`, `APP_BASE_URL`. Point app URLs at the Front Door hostname, not the pod IP.
10. Apply Prisma migrations from a job or a one-off pod. Seed only if you still need the initial admin, then change `admin@movie.com` / `admin123`.

## 10. Phase 6 — Private storage and Front Door

Goal: one public HTTPS entry, two private origins.

1. Storage account `leetamstrgacc`, public network access Disabled, static website enabled, index and error document `index.html`.
2. Build the React app on the VM only after setting `VITE_API_BASE_URL` to the backend Front Door URL. The value is baked into `dist` at build time. Changing it later does nothing until you rebuild and upload again.
3. Upload with the VM identity. The identity needs Storage Blob Data Contributor. If the role is already there and upload still fails, refresh the CLI session:

```bash
az logout
az login --identity
az storage blob upload-batch \
  --account-name leetamstrgacc \
  --destination '$web' \
  --source ./dist \
  --auth-mode login \
  --overwrite
```

4. Create Front Door Premium profile `leetamAFD`.
5. Frontend origin: Private Link to the storage `web` sub-resource. Approve the private endpoint on `leetamstrgacc`. The storage private endpoint name used was `leetamSTRGaccPVTep`.
6. Do not add `/api/*` on the same endpoint that already has `/*`. That route conflict happened in this project. Use a separate backend endpoint, `cinema-backend-endpoint`, with origin group `cinema-backend-origin-group`, route `cinema-backend-route`, origin the Private Link Service.
7. Approve the Front Door private endpoint connection on the Private Link Service.
8. Test the data path, not only the portal status. Portal deployment can still say `NotStarted` while the endpoint already returns HTTP 200.

Verified checks from the project:

- Frontend path: browser to Front Door to private storage `$web`.
- `GET /health` through the backend Front Door endpoint returned HTTP 200.
- `GET /api/movies` returned `success: true`, `count: 0` before movies were loaded.
- A user was created through the API, not by a hand-written SQL insert.

## 11. Phase 7 — Custom domain, not done

Proposed only:

- `www.tatt3r.com` on the frontend Front Door endpoint
- `api.tatt3r.com` on the backend Front Door endpoint

Not done yet: DNS validation, managed certificate, and a rebuild of the frontend so `VITE_API_BASE_URL` uses `https://api.tatt3r.com`. Until that rebuild, the deployed React app still calls the Azure Front Door hostname it was built with.

## 12. Verification order

When something fails, test one layer. Do not rebuild the network.

| Check | Pass condition seen in this project |
|---|---|
| VPN | `Test-NetConnection 10.0.0.4 -Port 22` succeeded from `172.16.201.2` |
| NAT | `leetamNATgtwy` Succeeded; VM could reach outbound and `az login --identity` |
| ACR DNS | `leetamcr.azurecr.io` resolved to `10.0.0.7` |
| ACR reachability | HTTPS to `/v2/` returned 401, then login succeeded |
| AKS access | `kubectl get nodes` as `azureuser`, not root |
| Image | `cinema-backend:v2` pulled on ARM64 nodes |
| Redis | `PONG` from inside the cluster |
| PostgreSQL | pod running on the StatefulSet PVC, `PGDATA` subdirectory |
| Internal LB | `10.224.0.7:5000` reachable from the private path |
| Front Door API | `/health` HTTP 200 |
| App | `/api/movies` returned success |

## 13. Failures that were part of the build

- No public IP on the VM. Fix: P2S VPN, not a public IP.
- No outbound from `subnet1`. Fix: `leetamNATgtwy` on `subnet1`.
- `sudo kubectl` failed. Fix: run `kubectl` as `azureuser`.
- PostgreSQL init failed on `lost+found`. Fix: `PGDATA` subdirectory.
- Extra Postgres PVC stayed Pending. Fix: use the StatefulSet PVC.
- Image was amd64 on ARM64 nodes. Fix: buildx for `linux/amd64,linux/arm64`.
- ACR private DNS was linked to `leetamVNET` but not the AKS VNet. Fix: link the zone to both.
- `az aks check-acr` exec format error. Diagnostic only. The real pull worked after the multi-arch image.
- Storage upload failed with a stale CLI token. Fix: `az logout` then `az login --identity`. Role was already correct.
- `/api/*` on the existing `/*` Front Door endpoint conflicted. Fix: separate backend endpoint.
- Rate-limit warning `ERR_ERL_STORE_REUSE` at startup. API still served `/health`. Fix in app code later.
- PowerShell `curl` JSON quoting broke login tests. Use `Invoke-RestMethod`.

## 14. Do not change

- Do not add a public IP to `leetamVM`, `leetamstrgacc`, `leetamcr`, or the AKS load balancer.
- Do not delete peering between `leetamVNET` and `aks-vnet-29219636`.
- Do not move AKS to another VNet.
- Do not put workloads in `GatewaySubnet`.
- Do not treat NAT gateway as inbound access.
- Do not point the browser at `10.224.0.7` or at the storage static-website hostname. Both are private. The browser uses Front Door.
- Do not invent resource names, IPs, or hostnames that are not in this SOP.
