# Adding a Worker Node to an Existing Cluster

[Back to Post-Install](index.md)

## Adding a Worker Node to an existing bare-metal on-premise OpenShift cluster

This guide describes the steps to add a new worker node to an existing OpenShift cluster. This example assumes you have a bare-metal on-premise OpenShift cluster and you want to add an additional worker node.

### Create the YAML configuration file

Create a file named `nodes-config.yaml` with the following content. Modify the values to match your environment. This should look very familiar as it is similar to the install-config.yaml file used during installation. If you are readding a node to an existing cluster, you can copy the install-config.yaml file and modify it as needed.

```yaml
hosts:
- hostname: extra-worker-1
  rootDeviceHints:
   deviceName: /dev/sda
  interfaces:
   - macAddress: 00:00:00:00:00:00
     name: eth0
  networkConfig:
   interfaces:
     - name: eth0
       type: ethernet
       state: up
       mac-address: 00:00:00:00:00:00
       ipv4:
         enabled: true
         address:
           - ip: 192.168.122.2
             prefix-length: 23
         dhcp: false
```

> [YAML file parameters](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/nodes/index#adding-node-iso-yaml-config_adding-node-iso)

??? example "nodes-config.yaml with bond0"
    This example includes two nics and a bond.

    ```yaml
    hosts:
    - hostname: extra-worker-1
      rootDeviceHints:
        deviceName: /dev/sda
      interfaces:
        - macAddress: A1:B2:3C:4D:1E:11
          name: eno1
        - macAddress: A1:B2:3C:4D:2E:11
          name: eno2
      networkConfig:
        interfaces:
          - name: eno1
            type: ethernet
            state: up
            mac-address: A1:B2:3C:4D:1E:11
            ipv4:
              enabled: false
            ipv6:
              enabled: false
          - name: eno2
            type: ethernet
            state: up
            mac-address: A1:B2:3C:4D:2E:11
            ipv4:
              enabled: false
            ipv6:
              enabled: false
          - name: bond0
            type: bond
            state: up
            ipv4:
              enabled: true
              address:
                - ip: 192.168.122.2
                  prefix-length: 23
              dhcp: false
            link-aggregation:
              mode: 802.3ad
              port:
                - eno1
                - eno2
              options:
                miimon: "100"
                lacp_rate: fast
            ipv6:
              enabled: false
        dns-resolver:
          config:
            server:
              - 9.9.9.9
              - 149.112.112.112
        routes:
          config:
            - destination: 0.0.0.0/0
              next-hop-address: 192.168.122.1
              next-hop-interface: bond0
              table-id: 254
    ```

### Create the ISO image

Run the following command to create the ISO image. Modify the parameters to match your environment.

```bash
oc adm node-image create nodes-config.yaml --dir=add --registry-config=/path/to/pull-secret.txt
```

> [Command flag options](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/nodes/index#adding-node-iso-flags-config_adding-node-iso)

Verify that a new `node.<arch>.iso` file is present in the asset directory. The asset directory is your current directory, unless you specified a different one when creating the ISO image.

Boot the node with the generated ISO image.

### Monitor the progress

```bash
oc adm node-image monitor --ip-addresses <ip_addresses>
```

`<ip_addresses>` is a comma-separated list of IP addresses assigned to the new nodes.

### Approve any CSRs that are pending

```bash
oc get csr
oc get csr | awk '{print $1}' | grep -v NAME | xargs oc adm certificate approve
```

### Documentation Links

* [Adding worker node to an on-premise cluster](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/nodes/index#adding-node-iso)
