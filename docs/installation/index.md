# Install

These install documents are focused on on-premise installations on bare metal. There are other install methods. If you can use the [Assisted Installer](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on-premise_with_assisted_installer/index#using-the-assisted-installer_installing-on-prem-assisted), do so.

You will need an entitled [Red Hat](https://www.redhat.com/wapps/ugc/register.html?_flowId=register-flow&_flowExecutionKey=e1s1) account.

Please read the entire page prior to starting. It is also recommended to acquire a git repository.

## Acquire the Hardware

### Required Hardware

Below is a list of the recommended hardware for a basic OpenShift install. Consider these minimum values - the more the better. Disks must be [SSD/NVME](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/etcd/index#recommended-etcd-practices_etcd-practices).

| Hosts (7 total)                    | CPU | Memory | Install Disk |
| ---                                | --- | ---    | ---          |
| bastion                            | 4   | 16 GB  | 60 GB        |
| openshift-control-plane-{1,2,3}    | 16  | 32 GB  | 120 GB       |
| openshift-worker-{1,2,3}           | 16  | 64 GB  | 120 GB       |

This also assumes your storage is handled elsewhere is already in place. If ODF will be used for storage, then worker machines would need additional CPU, RAM and an extra SSD/NVME drive in at least three of the worker machines to form a storage cluster.

There is an ability to run the control plane nodes as schedulable, making them effectively worker nodes as well. This is discouraged unless required by the environment constraints, such as testing to run edge systems.

!!! warning "etcd Disk Performance"
    Run etcd on a block device that can write at least 50 IOPS of 8KB sequentially, including fdatasync, in under 10ms.
    [Consider moving etcd to its own disk](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/etcd/index#move-etcd-different-disk_etcd-performance)

## Gather the Information

### Cluster Values

Below are the values an enterprise typically has to gather or create for installing OpenShift.

| &nbsp;                           | Example Value              | Description                                       |
| ---                              | ---                        | ---                                               |
| **Cluster Name**                 | poc                        | Name of the cluster                               |
| **Base Domain**                  | ocp.basedomain.com         | Name of the domain                                |
| **Cluster Suffix**               | poc.ocp.basedomain.com     | cluster name + base domain, for easier notation   |
| **Machine Subnet**               | 10.1.0.0/24 (vlan - 123)  | Subnet and vlan for all machines/VIPs in cluster  |
| **Pod Subnet**                   | 10.128.0.0/14              | Subnet for pod SDN                                |
| **Pod Subnet - Host Prefix**     | 23                         | Host prefix for Subnet for pod SDN                |
| **Service Subnet**               | 172.30.0.0/16              | Subnet for service SDN                            |
| **API VIP**                      | 10.1.0.9                   | VIP for the MetalLB API Endpoint *                |
| **Ingress VIP**                  | 10.1.0.10                  | VIP for the MetalLB Ingress Endpoint *            |
| **DNS**                          | dns1.basedomain.com,etc    | IP or hostname for the DNS hosts                  |
| **NTP**                          | ntp.basedomain.com,etc     | IP or hostname for the NTP hosts                  |

\* VIPs must be within the machine subnet

#### SDN Subnet Overlaps

The Pod Subnet and Service Subnet are run in the software defined network (SDN) of the cluster. If those values conflict with existing subnets **and** your pods in the cluster will want to route to those outside services with conflicting IPs, you will need to provide different subnets. It's much easier to ensure they are unique.

#### Pod Subnet and Host Prefix Explanation

If we are just doing a small cluster, think 500 pods per node and 6 nodes is 3000 ips. You could use a `/20` and then each node would have a `/23`. `10.128.0.0/20` with `hostPrefix: 23`.

```
10.128.0.0/23    (10.128.0.0 – 10.128.1.255)
10.128.2.0/23    (10.128.2.0 – 10.128.3.255)
10.128.4.0/23    (10.128.4.0 – 10.128.5.255)
10.128.6.0/23    (10.128.6.0 – 10.128.7.255)
10.128.8.0/23    (10.128.8.0 – 10.128.9.255)
10.128.10.0/23   (10.128.10.0 – 10.128.11.255)
10.128.12.0/23   (10.128.12.0 – 10.128.13.255)
10.128.14.0/23   (10.128.14.0 – 10.128.15.255)
```

> [Pod Subnet - Host Prefix](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#CO63-10)
> [CIDR Subnet Reference](https://docs.netgate.com/pfsense/en/latest/network/cidr.html#understanding-cidr-subnet-mask-notation)

### Machine Information Required

Typically, machines will have more than one NIC and these will be setup in a bond. Please collect the interface names and MAC addresses for ALL NICS and the install disk location on the machines. You provide the hostnames, IPs. IPs need to be located in the machine configuration subnet used on the install. Below is an example table of values needed for collection.

| Hostname                  | Interface | MAC Address        | Bond IP    | Disk Hint |
| ---                       | ---       | ---                | ---        | ---       |
| bastion                   | eno1      | A0-B1-C2-D3-E4-E1 | 10.1.0.5   | /dev/sda  |
| openshift-control-plane-1 | eno1      | A0-B1-C2-D3-E4-E1 | 10.1.0.11  | /dev/sda  |
|                           | eno2      | A0-B1-C2-D3-E4-E2 |            |           |
| openshift-control-plane-2 | eno1      | A0-B1-C2-D3-E4-E3 | 10.1.0.12  | /dev/sda  |
|                           | eno2      | A0-B1-C2-D3-E4-E4 |            |           |
| openshift-control-plane-3 | eno1      | A0-B1-C2-D3-E4-E5 | 10.1.0.13  | /dev/sda  |
|                           | eno2      | A0-B1-C2-D3-E4-E6 |            |           |
| openshift-worker-1        | eno1      | A0-B1-C2-D3-E4-F1 | 10.1.0.21  | /dev/sda  |
|                           | eno2      | A0-B1-C2-D3-E4-F2 |            |           |
| openshift-worker-2        | eno1      | A0-B1-C2-D3-E4-F3 | 10.1.0.22  | /dev/sda  |
|                           | eno2      | A0-B1-C2-D3-E4-F4 |            |           |
| openshift-worker-3        | eno1      | A0-B1-C2-D3-E4-F5 | 10.1.0.23  | /dev/sda  |
|                           | eno2      | A0-B1-C2-D3-E4-F6 |            |           |

The IPs in the example table represent a bonded IP or if the cluster does not use bonding, that IP that connects the machine to the machine network. Notice all machine IPs are inside of the machine subnet of 10.1.0.0/24 defined above.

On modern RHEL (RHEL CoreOS included), the names of your NICs aren't the old eth0, eth1 style anymore. They use predictable network interface names, which are generated at boot based on hardware topology and firmware information. Here's the gist of how RHEL decides what your NICs will be called:

* `eno1`, `eno2` — onboard NICs (from BIOS/firmware)
* `ens1f0`, `ens1f1` — PCI Express slots ("s" = slot, "f" = function)
* `enp3s0` — PCI bus location (p3 = bus 3, s0 = slot 0)
* `enx` — if nothing else matches, fall back to the MAC address

So the name is tied to where the NIC is physically, not just "first one detected." If you don't know what they will be, boot one of the boxes with a [RHEL ISO](https://access.redhat.com/downloads/content/rhel) and find out.

!!! note
    If you are using a hostname scheme that uses an integer at the end of the name, you should start with 1, not zero. But if you start with zero and you want it to match up with your ending IP, be consistent.

## Update the Network

### Firewall and Networking Requirements

Prior to the install, you must open the firewall to connect to Red Hat's servers and ports between the machines.

* [Configuring Firewall (note below)](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installation_configuration/configuring-firewall#configuring-firewall_configuring-firewall)
* [Network Connectivity Requirements](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#installation-network-connectivity-user-infra_installing-bare-metal)
* [Ensuring required ports are open](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#network-requirements-ensuring-required-ports-are-open_ipi-install-prerequisites)

!!! note
    If you are using third-party software from a particular external repository, you will need to provide access to that as well. It is recommended to provide access to docker.io.

#### MITM Proxy and Self-Signed Certificates

If you are using a MITM proxy which is doing TLS inspection, the Red Hat CDN for `dnf install` or `yum install` uses cdn.redhat.com which has a self-signed certificate. If your MITM proxy automatically blocks self-signed certificates, you will need to whitelist cdn.redhat.com.

It is recommended that all entries for the firewall as detailed above are also allowed through any TLS inspection processes.

### Create DNS Entries

Create the following A records in your DNS based on the values from above.

| A Record                                 | IP Address | Description                          |
| ---                                      | ---        | ---                                  |
| `api.<cluster_suffix>`                   | 10.1.0.9   | Virtual IP (VIP) for the API endpoint |
| `api-int.<cluster_suffix>`               | 10.1.0.9   | Internal VIP for the API endpoint    |
| `*.apps.<cluster_suffix>`                | 10.1.0.10  | Virtual IP for the ingress endpoint  |

You can validate the DNS using dig:

```shell
dig +noall +answer @<dns> api.<cluster_suffix>
dig +noall +answer @<dns> test.apps.<cluster_suffix>
```

> [DNS requirements](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#network-requirements-dns_ipi-install-prerequisites)
> [Validating DNS resolution for user-provisioned infrastructure](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/installing_on_bare_metal/index#installation-user-provisioned-validating-dns_installing-bare-metal-network-customizations)

#### Optional Helpful DNS Entries

| A Record                                  | IP Address | Description  |
| ---                                       | ---        | ---          |
| `bastion.<cluster_suffix>`                | 10.1.0.4   | IP for the bastion |
| `openshift-control-plane-1.<cluster_suffix>` | 10.1.0.11  | IP for cp1   |
| `openshift-control-plane-2.<cluster_suffix>` | 10.1.0.12  | IP for cp2   |
| `openshift-control-plane-3.<cluster_suffix>` | 10.1.0.13  | IP for cp3   |
| `openshift-worker-1.<cluster_suffix>`     | 10.1.0.21  | IP for w1    |
| `openshift-worker-2.<cluster_suffix>`     | 10.1.0.22  | IP for w2    |
| `openshift-worker-3.<cluster_suffix>`     | 10.1.0.23  | IP for w3    |

## Build the Bastion Host

The bastion host serves multiple purposes. First, all of our install work is done here. Second, it's used to validate the environment prior to installation. Third, it provides additional tools for installation such as a web server environment for the ISO images.

### Install the OS

* Download [Red Hat Enterprise Linux 9.x Binary DVD](https://access.redhat.com/downloads/content/rhel)
* Boot host from ISO and [perform install](rhel-install.md) as `Server with GUI`
* Make sure to enable SSH for user
* Reboot and SSH into bastion host as administrative user

### Register the Host

To be able to install necessary packages, you need to register the host with your Red Hat ID and password. You could have also accomplished this during the RHEL install.

```shell
sudo subscription-manager register # Enter username/password
sudo subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms
sudo dnf update -y
sudo reboot
```

### Download Required Tools

Install the required tools.

```shell
OCP_VERSION=4.20
wget "https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-${OCP_VERSION}/openshift-install-linux.tar.gz" -P /tmp
sudo tar -xvzf /tmp/openshift-install-linux.tar.gz -C /usr/local/bin
wget "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-${OCP_VERSION}/openshift-client-linux.tar.gz" -P /tmp
sudo tar -xvzf /tmp/openshift-client-linux.tar.gz -C /usr/local/bin
rm /tmp/openshift-install-linux.tar.gz /tmp/openshift-client-linux.tar.gz -y
sudo dnf install nmstate git podman
```

Check for the availability of the required tools.

```shell
openshift-install version
oc version
nmstatectl -V
git -v
podman --version
```

### Get the Pull Secret

Pull Secret is available at <https://console.redhat.com/openshift/install/pull-secret>.
Download to `~/.pull-secret`.

### Create SSH Key

```shell
ssh-keygen -t ed25519 -f ~/.ssh/ocp_ed25519
```

### Open Port 8080 for iso-http

```shell
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### Tools for Environment Validation

```shell
ping registry.redhat.io        # ICMP doesn't always work, but try
curl -vk https://registry.redhat.io/v2/
dig registry.redhat.io +short
nslookup registry.redhat.io
podman login registry.redhat.io
```

## Create the Configurations

The typical agent based install requires two configuration files.

* `install-config.yaml`
* `agent-config.yaml`

The `install-config.yaml` contains all the cluster level information while the `agent-config.yaml` provides host level configurations.

!!! tip
    You can accomplish all of this with bastion tools like `vi`. It is highly recommended to use tools like Visual Studio Code (vscode) to help with yaml formatting.

### Create the Working Directory

You are going to want to create a working directory and initialize it into a git repository.

```shell
mkdir -p ocp && cd ocp
git init
echo "install/" > .gitignore
echo "# POC install notes" > notes.md
touch install-config.yaml
touch agent-config.yaml
git add -A
git commit -m "repo initialized"
```

### Create the Install Config

Here is an example of the `install-config.yaml` contents.

```yaml
apiVersion: v1
baseDomain: ocp.basedomain.com
metadata:
  name: poc
controlPlane:
  name: master
  architecture: amd64
  hyperthreading: Enabled
  replicas: 3
compute:
- name: worker
  architecture: amd64
  hyperthreading: Enabled
  replicas: 3
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.1.0.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  baremetal:
    apiVIP: 10.1.0.9
    ingressVIP: 10.1.0.10
    additionalNTPServers:
      - 0.us.pool.ntp.org
      - 1.us.pool.ntp.org
pullSecret: 'value from ~/.pull-secret'
sshKey: 'value from ~/.ssh/ocp_ed25519.pub'
```

If your environment requires a proxy to access the internet then copy, modify and append to the end of the `install-config.yaml`:

```yaml
proxy:
  httpProxy: http://user:password@proxy.example.com:3128
  httpsProxy: http://user:password@proxy.example.com:3128
  noProxy: poc.ocp.basedomain.com,10.1.0.0/24,10.128.0.0/14,172.30.0.0/16,api.poc.ocp.basedomain.com,api-int.poc.ocp.basedomain.com,*.apps.poc.ocp.basedomain.com
```

### Create the Agent Config

The agent-config.yaml is basically a list of hosts' information. You'll create a list item for each of the hosts. Here's an example of a host with two ethernet connections bonded together in an LACP bond.

```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: poc
rendezvousIP: 10.1.0.11                     # IP of the first control-plane1 host
additionalNtpSources:
  - 0.us.pool.ntp.org
  - 1.us.pool.ntp.org
hosts:
  - hostname: openshift-control-plane-1     # hostname
    role: master                            # master/worker
    rootDeviceHints:
      deviceName: "/dev/sda"                # Disk Hint
    interfaces:
      - name: eno1                          # Interface Name 1
        macAddress: A1:B2:3C:4D:1E:11       # Mac Address 1
      - name: eno2                          # Interface Name 2
        macAddress: A1:B2:3C:4D:2E:11       # Mac Address 2
    networkConfig:
      interfaces:
        - name: eno1                        # Interface Name 1
          type: ethernet
          state: up
          mac-address: A1:B2:3C:4D:1E:11    # Mac Address 1
          ipv4:
            enabled: false
          ipv6:
            enabled: false
        - name: eno2                        # Interface Name 2
          type: ethernet
          state: up
          mac-address: A1:B2:3C:4D:2E:11    # Mac Address 2
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
              - ip: 10.1.0.11               # Host IP inside the machine subnet
                prefix-length: 24
            dhcp: false
          link-aggregation:
            mode: 802.3ad                   # LACP
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
            - dns1.basedomain.com           # DNS 1
            - dns2.basedomain.com           # DNS 2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.1.0.1
            next-hop-interface: bond0
            table-id: 254
```

!!! warning "Inconsistent Labels"
    Notice the inconsistent labels and spellings:

    * `macAddress` in the interfaces stanza, but `mac-address` in the networkConfig stanza.
    * `additionalNtpSources` is used here, but it's `additionalNTPServers` in the `install-config.yaml`.

> [Docs: Sample Config Bonds Vlans](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer#agent-install-sample-config-bonds-vlans_preparing-to-install-with-agent-based-installer)

## Generate the ISO

From the `~/ocp` directory, you can now create the agent iso. We create an install directory and copy the yaml files into the directory because the image creation process consumes and **destroys** the configuration files. We want to keep a copy in case the process needs to be repeated.

Here's an example script. I would recommend creating a small file named `create-iso.sh` in the `ocp` working directory with the contents. Don't forget to `chmod +x create-iso.sh`.

```shell
#!/bin/bash
rm -rf install
mkdir install
cp -r install-config.yaml agent-config.yaml install
openshift-install agent create image --dir=install
```

!!! tip
    You can always add `--log-level=debug` to any `openshift-install` commands for more output.

This is going to generate an `agent.x86_64.iso` file in the install directory which should be located at `~/ocp/install`. Remember, everything in the `install` directory has been excluded from the git repository.

## Boot the Machines

The best way to to provide the ISO as an HTTP source if you are using a BMC like iLo or iDRAC. Not doing so could result in issues during the install due to network contention using the web interfaces of those tools.

Luckily, on the bastion host, we already have podman installed so we can just use that to serve.

```shell
podman run -d --name iso-http \
  -p 8080:8080 \
  -v ~/ocp/install/agent.x86_64.iso:/var/www/html/agent.x86_64.iso:Z \
  registry.redhat.io/rhel9/httpd-24:9.6
```

Test retrieving the iso using the following command.

```shell
wget http://bastionhostname:8080/agent.x86_64.iso
```

!!! note
    Don't forget to [open the host firewall](#open-port-8080-for-iso-http) on the bastion.

If for some reason this solution will not work, you can install [nginx](httpd.md) and serve from there.

You can also copy it somewhere else from where the BMC has better access. Here's an example using scp.

```shell
scp user@192.168.122.187:~/ocp/install/agent.x86_64.iso ~/iso/
```

## Install the Cluster

When all the hosts are booted, wait for the bootstrap to complete.

```shell
openshift-install agent wait-for bootstrap-complete --dir=install
```

When the bootstrap is complete, wait for the install to complete.

```shell
openshift-install agent wait-for install-complete --dir=install
```

At the end of the process, you will be presented with the URL for the cluster endpoint, along with the kubeadmin credentials. They are also available in the install folder at `~/ocp/install` as files `kubeadmin-password` and `kubeconfig`.

Open the URL presented and log in. Wait until all checks are green before proceeding.

## Validate the Install

Login to the Cluster:

```shell
oc login --server=https://api.cluster.basedomain.com:6443 -u kubeadmin -p <password>
```

Test Connectivity:

```shell
oc debug node/<worker-node-name> -- chroot /host \
  podman pull registry.redhat.io/ubi9/ubi:latest
```

Cleanup the leftover install and configuration pods:

```shell
oc delete pods --all-namespaces --field-selector=status.phase=Succeeded
oc delete pods --all-namespaces --field-selector=status.phase=Failed
```

## Install Debugging

Modify the Boot Parameters (GRUB/Boot Menu)

This is often the most reliable way to get a shell or verbose output when direct TTY switching fails. You'll need to intercept the boot process.

* Reboot the machine with the ISO.
* At the GRUB (bootloader) menu: As soon as you see the initial boot menu, press an arrow key (up/down) to stop the automatic countdown.
* Edit the boot entry:
    * Select the installer's default boot entry (usually the first one).
    * Press the ++e++ key to edit the boot parameters.
    * Locate the `linux` or `linuxefi` line: This line contains the kernel arguments. Add debug/shell parameters to force an emergency shell. Go to the end of the `linux` or `linuxefi` line and add the param (see choices below). This will drop you into an initramfs shell before the root filesystem is mounted. It's a very minimal environment but allows `ip a show`, `dmesg`, and looking at files in the initramfs.
    * Press ++ctrl+x++ or ++f10++ to boot with the modified parameters.

The three main alternatives are:

### `rd.break`

This parameter tells dracut (the initramfs environment) to interrupt the normal boot process before the real root filesystem is mounted and before systemd takes over. You are dropped into an initramfs shell where you can manually mount `/sysroot` and chroot into it to make changes. Since this is very early in the boot sequence, the environment is limited, but it lets you work on the system before anything else initializes.

**When to use it:** This is the method of choice for situations where you need to fix problems directly on the root filesystem, especially when you can't log in normally—such as resetting a forgotten root password or repairing critical files before systemd runs. It's lower-level than `systemd.unit=emergency.target` and gives you control before the system even mounts root.

### `systemd.unit=emergency.target`

This parameter tells the systemd process to boot directly into a minimal shell. It will try to mount the root filesystem as read-only and start only the most essential services required for an emergency shell. This is often the preferred method for general system troubleshooting.

**When to use it:** This is a good choice for fixing issues that aren't related to the initial filesystem or the boot process itself, such as a corrupt `/etc/fstab` or a misconfigured service. It gives you a more complete environment than `rd.break` but still keeps things simple.

### `init=/bin/bash`

This is the most direct and basic method. It tells the kernel to bypass all the normal boot processes and execute `/bin/bash` directly as the first process (PID 1). This gives you a shell with no services, no network, and the root filesystem mounted as read-only.

**When to use it:** Use this as a last resort when other methods fail. It's the most primitive and powerful method, as it gives you control before any other processes or services start. It's ideal for a corrupted boot process or severe filesystem issues where `rd.break` or `emergency.target` might fail to load. It also requires you to manually remount the root filesystem as read-write, just like you were doing with `rd.break`.
