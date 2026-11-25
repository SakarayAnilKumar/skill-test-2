# Microservices on Kubernetes (`kind`) — Implementation & Verification Guide

---

## 1. Architecture Overview & Docker Compose Translation

The initial system ran four Node.js/Express microservices—`user-service` (port `3000`), `product-service` (port `3001`), `order-service` (port `3002`), and `gateway-service` (port `3003`)—connected over a single Docker bridge network (`microservice-network`) using Docker Compose. In Compose, `gateway-service` received downstream endpoints (`USER_SERVICE_URL`, `PRODUCT_SERVICE_URL`, `ORDER_SERVICE_URL`) via environment variables and resolved container hostnames on the bridge network.

Moving to Kubernetes replaces Docker Compose's single-host container orchestration with declarative cluster objects:

* **Deployments (`apps/v1`):** Manage `ReplicaSet` controllers and Pods for each microservice, enforcing desired replica counts (`1`) and self-healing if a Pod crashes or is deleted.
* **Services (`v1`, `ClusterIP`):** Provide stable virtual IPs and internal CoreDNS hostnames (`user-service:3000`, `product-service:3001`, `order-service:3002`, `gateway-service:3003`) so `gateway-service` reaches downstream Pods regardless of ephemeral Pod IP changes.
* **Ingress (`networking.k8s.io/v1`) & `ingress-nginx`:** Acts as the Layer-7 HTTP entry point (`localhost:80` on Windows via `kind` port mapping), routing external requests into `gateway-service:3003` using `proxy_pass http://upstream_balancer`.
* **Local Image Loading (`kind load docker-image`):** Transfers locally built `v1.0.0` images from the Windows Docker Desktop daemon into the `kind` node's `containerd` runtime with `imagePullPolicy: Never`.

---

## 2. Cluster Setup on Windows (`kind` & Port Mapping)

On Windows with Docker Desktop, `kind` runs Kubernetes control-plane and worker nodes as Docker containers. When `kind` reported `ERROR: no nodes found for cluster "kind"`, it indicated either no cluster existed with the default name `kind` or another cluster already existed for a separate project.

To keep this project isolated, a dedicated cluster named `microservices-dev` was created with host port `80` mapped into the control-plane container so external HTTP traffic on `http://localhost/...` reaches the Ingress controller:


Cluster creation and image loading workflow on Windows PowerShell:

```powershell
# 1. Create dedicated cluster and switch kubectl context to kind-microservices-dev
kind create cluster --name microservices-dev --config k8s/kind-config.yaml
kubectl get nodes

# 2. Build all four v1.0.0 microservice images locally
docker build -t user-service:1.0 ./user-service
docker build -t product-service:1.0 ./product-service
docker build -t order-service:1.0 ./order-service
docker build -t gateway-service:1.0 ./gateway-service

# 3. Load local images into the kind node's containerd store
kind load docker-image user-service:1.0 product-service:1.0 order-service:1.0 gateway-service:1.0 --name microservices-dev
```

## 3. Deployments & Service Manifests

### Deployments (`deployments/*.yaml`)
Each Deployment defines `replicas: 1`, a label selector (`app: <service-name>`), and a Pod template with `imagePullPolicy: Never` so `containerd` uses the pre-loaded `1.0` image. `gateway-deployment.yaml` injects the internal service URLs:

### ClusterIP Services (`services/*.yaml`)
Each Service maps a stable cluster `port` (`3000`, `3001`, `3002`, `3003`) to the container's `targetPort` and selects Pods matching `app: <service-name>`:

CoreDNS registers `<service-name>.default.svc.cluster.local` (resolving `<service-name>` within the `default` namespace), and `kube-proxy` routes Service traffic to healthy backend Pod endpoints.

---

## 4. Ingress Controller & Routing Configuration (`ingress-nginx`)

Kubernetes `Ingress` resources define routing rules declaratively, while an Ingress Controller implements them. On `kind`, the `ingress-nginx` controller was installed into the `ingress-nginx` namespace:

```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=120s
```

### How Binding & Reverse Proxying Work
1. `ingressClassName: nginx` in `k8s/ingress.yaml` matches the `nginx` `IngressClass` (`spec.controller: k8s.io/ingress-nginx`).
2. The `ingress-nginx` controller Pod watches `Ingress`, `Service`, and `EndpointSlice` objects via Kubernetes informers and dynamically generates NGINX configuration in `/etc/nginx/nginx.conf`, reloading NGINX gracefully (`nginx -s reload`).
3. Rather than overriding a static home page (`index.html`), NGINX acts as a Layer-7 reverse proxy generating `location` blocks with `proxy_pass http://upstream_balancer` (plus internal endpoints like `/healthz`, `/nginx_status`, and `/configuration`).

### Path Matching vs. Rewrite Behavior

* **With rewrite (`rewrite-target: /$2` on `/api(/|$)(.*)`):** `/api/orders` is rewritten to `/orders` before reaching `gateway-service`. If Express expects `/api/orders`, it responds with `404 Cannot GET /orders`.
* **Without rewrite (`path: /api`, `pathType: Prefix`):** `/api/orders` reaches `gateway-service` intact as `/api/orders`.

---

## 5. Local `kind` (`ingress-nginx`) vs. AWS Load Balancer (EKS)

While `ingress-nginx` runs inside the cluster on `kind`, cloud deployments on **Amazon EKS** can use the **AWS Load Balancer Controller** to provision an external **AWS Application Load Balancer (ALB)**.

| Dimension | `ingress-nginx` (Local `kind` / Any Cluster) | AWS Load Balancer Controller (Amazon EKS) |
| :--- | :--- | :--- |
| **Execution Model** | Runs as reverse-proxy Pods inside the cluster | Provisions and manages external AWS ALBs outside the cluster |
| **Local `kind` Support** | Supported on Windows + `kind` | Not applicable on local `kind` (requires AWS / EKS) |
| **Ingress Annotations** | `nginx.ingress.kubernetes.io/*` | `alb.ingress.kubernetes.io/*` |
| **Target Routing** | Routes via NGINX upstream to Service / Pod endpoints | Targets Pods (`target-type: ip`) or NodePorts (`target-type: instance`) |
| **Edge Integrations** | NGINX modules, Lua, custom config snippets | AWS WAF, AWS Certificate Manager (ACM) TLS, ALB features |

---

## 6. Operational Lessons & Troubleshooting Notes

* **Cluster name mismatch (`no nodes found for cluster "kind"`):** Always verify `kind get clusters` and `kubectl config current-context` when running multiple clusters on one machine, and pass `--name <cluster-name>` to `kind` commands.
* **Why `localhost:3000` failed while `http://localhost/api/orders` reached Express:** `user-service` uses a `ClusterIP` Service (`10.244.0.13:3000`) and is not bound to Windows `localhost:3000` unless you run `kubectl port-forward svc/user-service 3000:3000`. Meanwhile, `http://localhost/api/orders` reached `ingress-nginx` on port `80` and was proxied to `gateway-service:3003`, where Express returned `404 Cannot GET /orders` due to route/rewrite alignment.
* **Startup ordering & updates:** Unlike Docker Compose `depends_on`, Kubernetes starts Deployments independently—services should tolerate transient retry delays on startup. After rebuilding a Docker image, re-run `kind load docker-image` and `kubectl rollout restart deployment/<name>`.

---

## 7. Test Suite & Screenshot Checklist

Execute the following tests in PowerShell and capture a screenshot for each test case:

### Test 1: Cluster Context & Node Readiness
* **Command:**
  ```powershell
  kind get clusters
  kubectl config current-context
  kubectl get nodes
  ```

* 

### Test 2: Deployments, Pods, Services & Endpoints
* **Command:**
  ```powershell
  kubectl get deploy,pods,svc,endpoints
  ```
* 

### Test 4: `ingress-nginx` Controller & Generated `nginx.conf` Rules
* **Command:**
  ```powershell
  kubectl get pods -n ingress-nginx
  kubectl get ingress microservices-ingress
  $POD = kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath="{.items[0].metadata.name}"
  kubectl exec -n ingress-nginx $POD -- cat /etc/nginx/nginx.conf | findstr /C:"location " /C:"proxy_pass"
  ```
* **Screenshot:** 

### Test 5: Gateway Logs & End-to-End HTTP Request via Ingress
* **Command:**
  ```powershell
  kubectl logs deploy/gateway-service --tail=50
  curl.exe -i http://localhost/api/orders
  ```
* 

### Test 6: Pod Self-Healing & On-Demand `port-forward` Access
* **Command:**
  ```powershell
  kubectl delete pod -l app=user-service
  kubectl get pods -w
  # In a separate terminal for direct service access:
  kubectl port-forward svc/user-service 3000:3000
  curl.exe -i http://localhost:3000
  ```
* 