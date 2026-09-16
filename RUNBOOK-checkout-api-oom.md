# Runbook: checkout-api OOMKilled Incident

Use this runbook when `checkout-api` pods are restarting, requests are failing, or Kubernetes reports `OOMKilled`.

## Scope and safety

- Namespace: `checkout-api`
- Deployment: `checkout-api`
- Container: `checkout-api`
- Known-good memory limit from the incident: `128Mi`
- Do not reduce the limit during recovery. Capture current usage and evidence before making further resource changes.

## Diagnosis

### 1. Confirm the deployment and affected pods

```bash
kubectl get deployment checkout-api -n checkout-api
kubectl get pods -n checkout-api -l app=checkout-api -o wide
kubectl get pods -n checkout-api -l app=checkout-api \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[0].ready,RESTARTS:.status.containerStatuses[0].restartCount,PHASE:.status.phase'
```

Look for a rising `RESTARTS` count, a pod that is not ready, or a pod in `CrashLoopBackOff`.

### 2. Verify the termination reason

Set the affected pod name, then inspect its current and previous container state:

```bash
export POD_NAME="$(kubectl get pods -n checkout-api -l app=checkout-api \
  -o jsonpath='{.items[0].metadata.name}')"

kubectl describe pod "$POD_NAME" -n checkout-api
kubectl get pod "$POD_NAME" -n checkout-api -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
echo
```

Confirm that the terminated state includes `reason: OOMKilled`. A non-zero exit code without `OOMKilled` requires a different investigation.

### 3. Check configured resources and recent events

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="checkout-api")].resources}'
echo

kubectl get events -n checkout-api --sort-by='.lastTimestamp' \
  --field-selector involvedObject.kind=Pod
```

The memory limit must not be an unrealistic value such as `8Mi`. Compare the configured limit with observed usage when metrics are available:

```bash
kubectl top pod "$POD_NAME" -n checkout-api --containers
```

If `kubectl top` reports that metrics are unavailable, continue with the pod state and event evidence; do not treat missing metrics as proof that the pod is healthy.

### 4. Capture logs before restarting or replacing the pod

```bash
kubectl logs "$POD_NAME" -n checkout-api -c checkout-api --previous --timestamps
kubectl logs "$POD_NAME" -n checkout-api -c checkout-api --timestamps --tail=200
```

An empty `--previous` stream does not rule out an OOM. It means the immediately
previous container instance did not write anything to stdout/stderr, or that
the container was killed before it emitted output. This checkout-api stub does
not log startup and suppresses HTTP access logs, so an OOM during startup can
leave both log commands empty. In that case, use the termination state and pod
events as the primary evidence:

```bash
kubectl get pod "$POD_NAME" -n checkout-api \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
echo
kubectl describe pod "$POD_NAME" -n checkout-api | sed -n '/Last State:/,/Conditions:/p'
kubectl get events -n checkout-api --field-selector involvedObject.name="$POD_NAME" \
  --sort-by='.lastTimestamp'
```

Record the pod name, restart count, configured resources, termination reason, recent events, and relevant log output in the incident notes.

## Resolution

### 1. Restore the known-good resource limit

First inspect the live value:

```bash
kubectl get deployment checkout-api -n checkout-api \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="checkout-api")].resources.limits.memory}'
echo
```

If the limit is below the known-good value, apply the documented deployment manifest, which restores `requests.memory: 64Mi` and `limits.memory: 128Mi`:

```bash
kubectl apply -f checkout-api-deployment.yaml
```

From another directory, use the manifest's full path instead. If the manifest is not available, make the emergency change directly and record it for follow-up:

```bash
kubectl -n checkout-api set resources deployment/checkout-api \
  -c checkout-api --requests=memory=64Mi --limits=memory=128Mi
```

### 2. Wait for the rollout and verify the new pod

```bash
kubectl rollout status deployment/checkout-api -n checkout-api --timeout=120s
kubectl get pods -n checkout-api -l app=checkout-api -o wide
kubectl describe deployment checkout-api -n checkout-api
```

The rollout is healthy when the replacement pod is `Ready`, the restart count stops increasing, and no new `OOMKilled` events appear.

### 3. Verify the application responds

Find the service and test the health endpoint from inside the cluster or through the normal service access path:

```bash
kubectl get service -n checkout-api
kubectl run checkout-api-healthcheck -n checkout-api --rm -i --restart=Never \
  --image=curlimages/curl -- \
  curl --fail --silent --show-error http://checkout-api:8000/health
```

If the service has a different name, use the name returned by `kubectl get service`.

### 4. Confirm stability before closing

```bash
kubectl get pods -n checkout-api -l app=checkout-api \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[0].ready,RESTARTS:.status.containerStatuses[0].restartCount,PHASE:.status.phase'
kubectl get events -n checkout-api --sort-by='.lastTimestamp' \
  --field-selector involvedObject.kind=Pod
kubectl top pod -n checkout-api -l app=checkout-api --containers
```

Continue observing the deployment for several minutes under normal traffic. Close the incident only when requests succeed consistently, the pod remains ready, and the restart count is stable.

## Escalation and follow-up

Escalate to the service owner if the pod is still OOMKilled at `128Mi`, usage approaches the limit under normal traffic, or the health check fails after rollout. Preserve the captured evidence and investigate workload demand, traffic changes, and possible application memory growth before increasing the limit further.

Create a follow-up change to validate resource-limit edits against observed usage before deployment and to avoid applying unreviewed values directly to the cluster.
