# Troubleshooting

Common issues and their solutions when installing or configuring OpenShift.

## Useful Commands

```bash
# Check cluster operator status
oc get clusteroperators

# View installer logs
openshift-install wait-for install-complete --log-level=debug

# Check node status
oc get nodes

# Inspect a failing pod
oc describe pod <pod-name> -n <namespace>
oc logs <pod-name> -n <namespace>

# Gather cluster diagnostics
oc adm must-gather
```

## Common Issues

Topics to be covered:

- Bootstrap failures
- Certificate and DNS issues
- Stuck cluster operators
- Network connectivity problems
- Storage provisioning errors

!!! warning
    Always collect `must-gather` output before opening a support case.
