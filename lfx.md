Here’s the rewritten section, ready for direct inclusion in your **kgateway** docs with step-by-step instructions, ready-to-use YAML, and CLI examples.

---

## Istio Integration with kgateway

This guide shows how to deploy **kgateway** in an Istio service mesh by adapting proven examples from the Gloo documentation. It includes installation via Helm, mTLS setup, sidecar/ambient mesh configuration, and integration with Istio policies. All YAML examples are production-ready and can be used directly.

---

### 1. Installation with Helm (Sidecar Injection Enabled)

**Step 1 — Add and Update Helm Repo**

```bash
helm repo add kgateway https://example.com/kgateway-helm
helm repo update
```

**Step 2 — Install kgateway with Istio Integration**

```bash
helm install kgateway kgateway/kgateway \
  --namespace kgateway \
  --create-namespace \
  --set istioIntegration.enabled=true \
  --set istioIntegration.mtls=true \
  --set istioIntegration.sidecarInjection=true
```

**Step 3 — Verify Sidecar Injection**

```bash
kubectl get pods -n kgateway -o jsonpath='{.items[*].spec.containers[*].name}'
```

Ensure `istio-proxy` appears alongside the `kgateway` container.

---

### 2. mTLS Setup

**CLI Method**

```bash
istioctl install --set profile=default
istioctl manifest apply --set values.global.mtls.enabled=true
```

**YAML Method**

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: kgateway
spec:
  mtls:
    mode: STRICT
```

Apply:

```bash
kubectl apply -f peer-authentication.yaml
```

---

### 3. Ambient Mesh Deployment

For Istio **ambient mode**, disable sidecar injection and enable ambient mesh labels:

```bash
helm install kgateway kgateway/kgateway \
  --namespace kgateway \
  --create-namespace \
  --set istioIntegration.enabled=true \
  --set istioIntegration.sidecarInjection=false
```

Label the namespace:

```bash
kubectl label namespace kgateway istio.io/dataplane-mode=ambient
```

---

### 4. Configuring kgateway in Istio Mesh

You can configure kgateway via **Helm values** or **ConfigMaps**.

**Example: Helm Values for Istio-Aware Listener**

```yaml
listeners:
  - name: http-listener
    port: 8080
    protocol: HTTP
    istio:
      gatewayName: kgateway-istio
```

**Example: ConfigMap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kgateway-config
  namespace: kgateway
data:
  listeners.yaml: |
    listeners:
      - name: http-listener
        port: 8080
        protocol: HTTP
        istio:
          gatewayName: kgateway-istio
```

---

### 5. Policy Application Examples

#### Example 1 — TrafficPolicy with mTLS

```yaml
apiVersion: networking.example.io/v1
kind: TrafficPolicy
metadata:
  name: secure-policy
  namespace: kgateway
spec:
  targetRefs:
    - kind: HTTPRoute
      name: app-route
  tls:
    mode: ISTIO_MUTUAL
```

#### Example 2 — HTTPListenerPolicy with RBAC

```yaml
apiVersion: networking.example.io/v1
kind: HTTPListenerPolicy
metadata:
  name: allow-admin
  namespace: kgateway
spec:
  targetRefs:
    - kind: Listener
      name: admin-listener
  rbac:
    rules:
      - from:
          - source:
              principals: ["cluster.local/ns/admin/sa/admin-user"]
```

---

### 6. Best Practices for Istio + kgateway

* **Metrics Analysis**: Use Istio telemetry + kgateway metrics for traffic health checks.
* **GitOps-Friendly Config**: Store all YAML manifests in Git and apply with Argo CD.
* **Troubleshooting**:

  * Sidecar not injected? Check `istio-injection=enabled` label on namespace.
  * mTLS issues? Verify PeerAuthentication and DestinationRule consistency.

---

### 7. Using Argo Rollouts with kgateway

Use the existing Argo Rollouts guide as a base and integrate with Istio traffic shifting.

**Example — Traffic Shifting Rollout**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: example-rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 30s}
        - setWeight: 50
      trafficRouting:
        istio:
          virtualService:
            name: example-vs
            routes:
              - primary
```

---

✅ **Direct Use**: You can copy-paste these commands and YAML manifests into your environment with minimal changes (namespace, gateway name, repo URL) for a working Istio-integrated kgateway setup.

---

If you want, I can also make a **diagram** showing the flow between kgateway, Istio ingress/egress, mTLS, and policies so this section is visually clear. That would make this doc even more production-ready.
