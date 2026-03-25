# Rotate SSH Keys after OpenShift Install

[Back to Post-Install](index.md)

Below is the procedure to rotate the SSH keys.

```shell
ssh-keygen -t rsa -b 4096 -f ~/.ssh/new_ocp_key -C "ocp-key-rotation"
oc get machineconfig 99-master-ssh -o yaml > 99-master-ssh.yaml
```

Update the MachineConfig to include the new key:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-ssh
spec:
  config:
    ignition:
      version: 3.2.0
    passwd:
      users:
      - name: core
        sshAuthorizedKeys:
        - ssh-rsa AAAA...your_old_key_here...
        - ssh-rsa AAAA...your_new_key_here...
```

Apply the updated MachineConfig:

```shell
oc apply -f 99-master-ssh.yaml
```

Repeat for worker nodes: If you also want to rotate keys for worker nodes, repeat the steps above using `99-worker-ssh` instead of `99-master-ssh`.
