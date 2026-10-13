# SOP: Migrate Cinema Booking Backend to Private-Only Access via BFF + AKS Ingress

## Purpose

Remove all public and Front Door paths to the backend while keeping the public React frontend working.  
After this change:

- Frontend remains publicly reachable via Azure Front Door → Private Link → Storage `$web`.
- Backend (`cinema-backend`) is reachable only from inside the cluster (ClusterIP).
- Public API traffic flows: Browser → AKS Ingress → BFF → private backend Service.
- Admin access continues via existing Point-to-Site VPN + admin VM.
- No Private Link Service and no Front Door origin exist for the backend.

## Prerequisites

- Admin access via the existing P2S VPN to `leetamVM`.
- `kubectl` configured for the AKS cluster (run as the normal user, not `sudo`).
- Helm installed on the admin VM (or ability to install it).
- Access to the Azure Portal or Azure CLI for Front Door and Private Link resources.
- Ability to rebuild and upload the React frontend.
- Private ACR access if you build a custom BFF image.

## Phase 1 — Remove existing backend public path

Perform these steps in order.

### 1.1 Remove backend origin from Azure Front Door

1. Open the Front Door Premium profile (`leetamAFD`).
2. Go to **Origin groups**.
3. Delete the origin group that contains the Private Link origin pointing at `leetam-cinema-backend-pls`.
4. Go to **Routes** and **Endpoints**.
5. Delete the backend route and any dedicated backend endpoint.
6. Leave the frontend origin (Storage `$web` via Private Link) unchanged.
7. Save and wait for the change to propagate.

### 1.2 Delete the Private Link Service

1. Go to **Private Link Center** → **Private link services**.
2. Select `leetam-cinema-backend-pls`.
3. Delete the Private Link Service.
4. Confirm no pending private endpoint connections remain.

### 1.3 Change the backend Service to ClusterIP

On the admin VM:

```bash
kubectl edit svc cinema-backend -n cinema
```

Change the Service to:

```yaml
spec:
  type: ClusterIP
```

Remove the annotation (if present):

```yaml
service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

Save and exit. Verify:

```bash
kubectl get svc cinema-backend -n cinema
```

Expected: only a ClusterIP, no EXTERNAL-IP. The Azure internal Load Balancer is removed automatically.

### 1.4 Confirm backend is no longer publicly reachable

- Front Door has no backend origin or route.
- Private Link Service is deleted.
- Backend Service is ClusterIP only.
- No public IP is associated with the backend.

## Phase 2 — Implement BFF + AKS Ingress

### 2.1 Create the BFF Deployment and Service

Deploy a reverse proxy that forwards `/api` (and any other needed paths) to the backend Service.

Example minimal Express BFF logic (adapt as needed):

```js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const app = express();

app.use('/api', createProxyMiddleware({
  target: 'http://cinema-backend.cinema.svc.cluster.local:5000',
  changeOrigin: true,
}));

app.get('/health', (req, res) => res.send('ok'));
app.listen(3000);
```

Apply a Deployment and ClusterIP Service for the BFF in the `cinema` namespace.  
Example Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cinema-bff
  namespace: cinema
spec:
  type: ClusterIP
  selector:
    app: cinema-bff
  ports:
    - port: 3000
      targetPort: 3000
```

### 2.2 Install NGINX Ingress Controller (if not already present)

On the admin VM:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

Wait for a public EXTERNAL-IP:

```bash
kubectl get svc -n ingress-nginx
```

### 2.3 Create Ingress for the BFF only

Apply an Ingress that routes traffic only to the BFF Service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cinema-bff
  namespace: cinema
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
    - host: api.yourdomain.com   # replace with your hostname or omit for IP testing
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: cinema-bff
                port:
                  number: 3000
```

Do not create any Ingress rule that points at `cinema-backend`.

### 2.4 Update the frontend API base URL and redeploy

The React frontend calls the backend using the value of `VITE_API_BASE_URL`, which Vite embeds at build time. This must be changed to the public hostname of the new Ingress (BFF).

1. Update the environment variable before building.  
   Example (`.env`, `.env.production`, or inline):

   ```bash
   # Old value (Front Door backend hostname) — remove
   # VITE_API_BASE_URL=https://old-backend-endpoint.azurefd.net

   # New value — public hostname or IP of the Ingress
   VITE_API_BASE_URL=https://api.yourdomain.com
   ```

   Also check for any other hardcoded API URLs in the frontend source and update them if present.

2. Rebuild the frontend:

   ```bash
   cd frontend
   npm run build
   ```

3. Upload the contents of `frontend/dist/` (not the `dist` folder itself) to the Storage Account `$web` container, overwriting the previous files.

4. After upload, hard-refresh the site in a browser and confirm in the Network tab that API requests now go to the Ingress hostname and succeed.

**Note:** Changing `VITE_API_BASE_URL` only in Azure or in a runtime config has no effect. The value is baked into the JavaScript at build time, so a rebuild + re-upload is required.

### 2.5 Optional hardening

- Add rate-limiting or authentication annotations on the Ingress.
- Place Azure Front Door in front of the Ingress public IP for WAF (Front Door origin = Ingress, not the backend).
- Restrict the Ingress controller Load Balancer with NSG rules if desired.

## Verification

| Check | Expected result |
|-------|-----------------|
| Frontend Front Door URL | React site loads |
| Browser API calls | Reach Ingress → BFF → backend and succeed |
| `kubectl get svc cinema-backend -n cinema` | ClusterIP only, no EXTERNAL-IP |
| Direct public access to backend | Fails |
| Admin VPN + kubectl | Still works |
| Front Door backend origin | Does not exist |
| Private Link Service | Deleted |

## Rollback notes

- Re-create the internal Load Balancer Service and Private Link Service if you must restore the previous Front Door path.
- Re-add the Front Door backend origin and approve the Private Link connection.
- Revert `VITE_API_BASE_URL` and rebuild the frontend.

## Safety notes

- Never commit secrets, kubeconfig, or real credentials.
- The Kubernetes manifests in the repository are templates; review them before applying.
- Confirm Storage public network access remains disabled.
- Test the data path (not only portal status) after each major change.

## Resulting architecture

```text
Public browser
  ├─ Frontend → Azure Front Door → Private Link → Storage $web
  └─ API calls → AKS Ingress → BFF → cinema-backend ClusterIP → pods

Admin → P2S VPN → admin VM → private cluster access
```

Backend has no public IP, no Private Link Service, and no Front Door origin.
