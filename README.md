# kubecon_India_demo_vllm_kagent_kgateway
Demo of Kubecon India 2026 talk - "Zero-GPU Autopilot: Orchestrating Kagent and Kgateway for Private, Self-Healing Clusters"

Step 1- Prerequisite: 
 - Create a Cluster
 - Have a dedicated node for VLLM 

Step 2 — Pin vLLM to a node
Edit manifests/vllm/deployment.yaml:
```
nodeSelector:
  kubernetes.io/hostname: <dedicated-vllm-node-name>
```
Use a node from kubectl get nodes.

Step 3 — Kgateway + Gateway API
Kgateway works well with gateway api CRDs 1.5.1 version. If your cluster is already having that CRD or above then you can skip the CRDs installation step and move towards kgateway installation.
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
helm upgrade -i kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds `
  --create-namespace --namespace kgateway-system --version v2.3.1
helm upgrade -i kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway `
  --namespace kgateway-system --version v2.3.1

kubectl wait -n kgateway-system --for=condition=Ready pod -l app=kgateway --timeout=300s
```

Step 4 — Namespaces + sample app
```
kubectl apply -f manifests\namespaces.yaml
kubectl apply -f manifests\sample_app\shop-api.yaml
kubectl wait -n demo-apps --for=condition=Ready pod -l app=shop-api --timeout=300s
```
Step 5 — vLLM (allow 10–20 min first boot)
```
kubectl apply -f manifests\vllm\pvc.yaml
kubectl apply -f manifests\vllm\service.yaml
kubectl apply -f manifests\vllm\deployment.yaml
kubectl rollout status deployment/vllm-cpu -n vllm --timeout=1200s
```
Verify:
```
kubectl run vllm-test --rm -it --restart=Never --image=curlimages/curl -n vllm -- `
  curl -s http://vllm.vllm.svc:8000/v1/models
```
Step 6 — Gateway + HTTPRoute
```
kubectl apply -f manifests\kgateway\gateway.yaml
kubectl get gateway shop-gateway -n demo-apps -w
```
When an address appears:
```
$GW = kubectl get gateway shop-gateway -n demo-apps -o jsonpath='{.status.addresses[0].value}'
curl.exe -s -H "Host: shop.demo.local" "http://$GW/health"
```
Step 7 — Prometheus installation: 
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade -i prom prometheus-community/kube-prometheus-stack `
  --namespace monitoring --create-namespace `
  -f manifests\05-prometheus\values-minimal.yaml
kubectl apply -f manifests\sample_app\podmonitor.yaml
kubectl apply -f manifests\autonomous\prometheus-rule.yaml
```
Step 8 — If you are installing Kagent in DOKS Cluster then use --server-side=false 
```
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds `
  --namespace kagent --create-namespace --version 0.9.7 --server-side=false
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent `
  --namespace kagent --version 0.9.7 --server-side=false `
  -f manifests\kagent\kagent-values.yaml
kubectl wait -n kagent --for=condition=Available deployment/kagent-controller --timeout=300s
```
Step 9 — Model, SOPs, agents
```
kubectl apply -f manifests\kagent\secret-vllm.yaml
kubectl apply -f manifests\kagent\modelconfig-vllm.yaml
kubectl apply -f manifests\kagent\sre-sops-configmap.yaml
kubectl apply -f manifests\kagent\agent-local-sre.yaml
kubectl get pods -n kagent -l 'kagent in (local-sre)' -w
```
Step 10 - Demo of injecting failure: 
```
kubectl patch deployment shop-api-v2 -n demo-apps --type=strategic `
  --patch-file manifests\kagent\failure-injection-patch.yaml
for ($i=0; $i -lt 200; $i++) { curl.exe -s -o NUL -H "Host: shop.demo.local" "http://$GW/" }
kubectl logs -n kagent -l kagent=local-sre -f
kubectl get httproute shop-route -n demo-apps -w
```
