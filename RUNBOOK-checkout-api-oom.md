1. Diagnosis
1.1. Confirm pod instability
Check pod status and restart counts:

bash
kubectl get pods -n checkout \
  -l app=checkout-api \
  -o wide
What to look for:

STATUS showing CrashLoopBackOff or frequent restarts

RESTARTS steadily increasing

1.2. Confirm OOMKilled as the reason
Describe an affected pod:

bash
POD=$(kubectl get pods -n checkout -l app=checkout-api -o jsonpath='{.items[0].metadata.name}')

kubectl describe pod "$POD" -n checkout
What to look for (in Containers → Last State):

Reason: OOMKilled

Exit Code: 137 (typical for OOM kill)

Any recent events mentioning OOM or restarts

1.3. Inspect container logs
Previous container logs (before restart):

bash
kubectl logs "$POD" -n checkout --previous
Goal:

Confirm there is no obvious application-level error preceding the kill

If logs are truncated or missing, that’s consistent with OOMKilled

1.4. Check current resource configuration
Get the deployment manifest (focused on resources):

bash
kubectl get deployment checkout-api -n checkout -o yaml
What to verify:

spec.template.spec.containers[].resources.limits.memory

Compare the configured limit (e.g., 8Mi) with known-good value (e.g., 128Mi)

Confirm whether the recent change lowered the limit below realistic usage

1.5. (Optional) Check live memory usage
If metrics-server or Prometheus is available:

bash
kubectl top pod -n checkout \
  -l app=checkout-api
Goal:

Validate that normal memory usage is higher than the current limit

Distinguish “limit too low” from “memory leak” (steady growth over time)

2. Resolution

sed -i 's/"8Mi"/"128Mi"/g' checkout-api-deployment.yaml
kubectl apply -f checkout-api-deployment.yaml

3. Follow-up / Preventive Actions
These are operational steps tied to the incident’s action items:

3.1. Document OOMKilled response in the runbook
Include:

The diagnosis steps above

The known-good memory limit for checkout-api

Where to find historical memory usage (Grafana/Prometheus dashboards, kubectl top)

3.2. Validate future resource changes against real usage
Before changing checkout-api memory limits:

Check current usage:

bash
kubectl top pod -n checkout -l app=checkout-api
Compare proposed limit to observed usage plus buffer

Requests ≈ typical usage

Limits ≈ peak usage + safety margin (e.g., 20–30%)

Only apply changes that remain above realistic usage.