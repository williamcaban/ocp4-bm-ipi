# OpenShift 4.2.x Bare-Metal IPI

- Define environment variables to use during the installation:
```
export PULL_SECRET="pull-secret.json"
export INSTALLER="openshift-baremental-install"
export RELEASE_IMG=`curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.2/release.txt | grep Pull | cut -f3 -d' '`

```
- Obtain a pull-secret from https://try.openshift.com and save it as `pull-secret.json`
- Obtain the `oc` client for your platform from https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/
- Download the bare-metal IPI installer:
```
oc adm release extract --registry-config "${PULL_SECRET}" --command=${INSTALLER} --to "${extract_dir}" ${RELEASE_IMG}
```

## Preparing the Provisioning Host

NOTE: The provisioning host must be RHEL8 or later as it need a `qemu-kvm` with support for the `-fw_cfg` flag to inject the ingiiton files during the installation.

For this example, the node is using the interfaces:
- External Network is interface `p1p1`
- Provisining Network is interface `em3`


- Create a `baremetal` bridge for the external network
```
nmcli con add type bridge ifname baremetal con-name baremetal
nmcli con add type bridge-slave ifname p1p1 master baremetal
nmcli con up baremetal

nmcli con show
```

- Create a `provisioning` bridge for the provisioning network
```
nmcli con add type bridge ifname provisioning con-name provisioning
nmcli con add type bridge-slave ifname em3 master provisioning
nmcli con up provisioning

nmcli con show
```

- Setup environment variables to override and specify the NIC cards used by the 
```
export TF_VAR_external_bridge=p1p1
export TF_VAR_provisioning_bridge=em3
```



