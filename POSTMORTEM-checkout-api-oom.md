# Postmortem: checkout-api OOMKilled Incident

## Summary
checkout-api pods were repeatedly killed and restarted for approximately
12 minutes, causing intermittent request failures. The root cause was a
memory limit set below what the workload actually required under normal
load, not a memory leak in the application itself.

## Timeline (all times approximate)
- 14:02 - A routine resource-limit change is applied, lowering
  checkout-api's memory limit from 128Mi to 8Mi as part of a cost-tuning
  pass.
- 14:03 - checkout-api pods begin restarting repeatedly.
  `kubectl get pods` shows a climbing RESTARTS count.
- 14:06 - On-call engineer notices intermittent request failures and
  begins investigating.
- 14:09 - `kubectl describe pod` on the affected pod shows
  `Reason: OOMKilled` under the container's Last State.
- 14:14 - Memory limit is reverted to 128Mi and reapplied.
- 14:15 - Pods stabilize. RESTARTS stops climbing. Requests succeed
  consistently.

## Impact
Intermittent request failures for approximately 12 minutes. No data
was lost; failed requests received connection errors rather than
incorrect responses.

## Root Cause
The memory limit change was applied without checking checkout-api's
actual observed memory usage first. 8Mi was well below what the
application needs even at idle.

## Contributing Factors
- No automated check existed to catch an unrealistic resource value
  before it reached the cluster.
- The change was applied directly, without comparing it against any
  current usage data.

## Resolution
Reverting the memory limit to its previous, known-good value (128Mi)
resolved the incident immediately.

## Action Items
- Document a response runbook for OOMKilled incidents specifically.
- Consider validating resource-limit changes against real usage data
  before they reach the cluster.