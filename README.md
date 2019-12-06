# OpenShift 4.2.x/4.3.x Bare-Metal IPI

- Obtain a pull-secret from https://try.openshift.com and save it as `pull-secret.json`
- Obtain the `oc` client for your platform from https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/

- Define environment variables to use during the installation:
    ```
    export PULL_SECRET="pull-secret.json"
    export INSTALLER="openshift-baremetal-install"
    export RELEASE_IMG=`curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.2/release.txt | grep Pull | cut -f3 -d' '`
    export EXTRACT_DIR="./"
    ```
- Download the bare-metal IPI installer:
    ```
    oc adm release extract --registry-config "${PULL_SECRET}" --command=${INSTALLER} --to ${EXTRACT_DIR} ${RELEASE_IMG}
    ```

## Preparing the Provisioning Host

NOTE: The provisioning host must be RHEL8 or later as it needs a `qemu-kvm` with support for the `-fw_cfg` flag to inject the ingition files during the installation.

- For this example, the node is using the interfaces:
  - External Network is interface `enp130s0f0`
  - Provisining Network is interface `eno3`


- Create a `baremetal` bridge for the external network
    ```
    nmcli con add type bridge ifname baremetal con-name baremetal
    nmcli con add type bridge-slave ifname enp130s0f0 master baremetal
    nmcli con up baremetal

    nmcli con show
    ```
    **NOTE:** Internet connectivity is required over the `baremetal` network to reach repositories at `quay.io`.


- Create a `provisioning` bridge for the provisioning network
    ```
    nmcli con add type bridge ifname provisioning con-name provisioning
    nmcli con add type bridge-slave ifname eno3 master provisioning
    nmcli con up provisioning

    nmcli con show
    ```
    **NOTE:** The `provisioning` network is an isolated network the Nodes will use during the provisioning stages. Configure the bridge with the network `172.22.0.0/24` and the provisioning machine with the IP `172.22.0.1`

- Setup environment variables to override and specify the NIC cards used by the installer
    ```
    export TF_VAR_external_bridge=enp130s0f0
    export TF_VAR_provisioning_bridge=eno3
    ```

### Overriding RHCOS Image
If using a custom RHCOS image or using proxy configurations:

- Download and publish the OpenStack QCOW image into a local web serve
    ```
    curl -O https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/latest/rhcos-43.81.201912030353.0-openstack.x86_64.qcow2.gz
    ```
    NOTE1: Do not uncompress the qcow2 image as the installer is expecting it to be compressed
    NOTE2: The OpenStack images are the ones REQUIRED for bare-metal deployment as they are the RHCOS images that are supported by the Ironic (part of the Metal3).
- Setup the `OPENSHIFT_INSTALL_OS_IMAGE_OVERRIDE` environment variable before running the installer
    ```
    export OPENSHIFT_INSTALL_OS_IMAGE_OVERRIDE=http://myserver.local/rhcos-openstack.x86_64.qcow2.gz
    ```

## Deploying OpenShift
- Create directory for the installation and copy the `install-config.yaml` there
    ```
    mkdir ./mycluster
    cp ./install-config.yaml ./mycluster/install-config.yaml
    ```
- Deploy the cluster using the `openshift-baremetal-install` binary following the standard OCP IPI installation process:
    ```
    ./openshift-bearemetal-install create cluster --dir ./mycluster
    ```

## Deploying with customizations

- Generate the manifests
    ```
    ./openshift-bearemetal-install create manifests --dir ./mycluster
    ```
- Update manifests with desired customziations
- Generate new ignition files
    ```
    ./openshift-bearemetal-install create ignition-configs --dir ./mycluster
    ```
- Proceed with installation
    ```
    ./openshift-bearemetal-install create cluster --dir ./mycluster
    ```

## Troubleshooting

- Login into the `bootstrap` machine
  - Identify the IP of the bootstrap machine
      ```
      virsh net-dhcp-leases baremetal
      ```
  - Login to the bootstrap machine
      ```
      slogin core@<ip-of-bootstrap>
      ```

- Validate there is a `default` storage pool
  ```
  virsh pool-list
  ```

- Check logs for the `ironic` service
    ```
    journalctl -u ironic
    ```

- Check the logs of `coreos-downloader` or `ipa-downloader` on the bootstrap node

- To enable verbose logs set the `TF_LOG` environment variable:
    ```
    export TF_LOG=trace
    ```


## Acknowledgements

Thanks to Stepehn [Benjamin](https://github.com/stbenjam) and [Doug Hellmann](https://github.com/dhellmann)