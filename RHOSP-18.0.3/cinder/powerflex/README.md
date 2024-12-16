# Dell EMC PowerFlex Backend Deployment Guide for Red Hat OpenStack Platform 18.0

Deployment tools for Dell EMC PowerFlex (formerly VxFlex OS/ScaleIO) support in RedHat OpenStack Platform 18.0

## Overview

This instruction provide detailed steps on how to enable PowerFlex.

**NOTICE**: this README represents only the **basic** steps necessary to enable PowerFlex driver. It does not contain steps other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Platform 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- Red Hat OpenStack Platform 18.0 with RHEL 9.6.
- PowerFlex 4.x cluster.
- PowerFlex Storage Data Client (SDC) for RHEL9.6.

## Steps

### Prepare the environment for PowerFlex cinder backend
To enable the OpenStack workloads to consume PowwerFlex as backend follow the below instructions:

* Create a Secret CR file powerflex-secret.yaml
  
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-powerflex-secrets
stringData:
  powerflex-secrets.conf: |
    [powerflex]
    san_ip=PF_Manager_IP
    san_login=PF_Manager_login
    san_password=PF_Manager_password
type: Opaque
```

* Apply the CR file by
```yaml
oc create -f powerflex-secret.yaml
```

* Verify the secret is created
```yaml
oc describe secret/cinder-volume-powerflex-secrets
```
For full detailed instruction of all options please refer to [PowerFlex Backend Configuration](https://docs.openstack.org/cinder/wallaby/configuration/block-storage/drivers/dell-emc-powerflex-driver.html).

* Edit the existing OpenStackControlPlane CR
```
oc edit openstackcontrolplane
```
Modify the cinderVolumes section as follows
```
...
cinderVolumes:
        powerflex:
          containerImage: registry.redhat.io/rhosp-dev-preview/openstack-cinder-volume-rhel9:18.0
          customServiceConfig: |
            [powerflex]
            volume_driver = cinder.volume.drivers.dell_emc.powerflex.driver.PowerFlexDriver
            volume_backend_name = powerflex
            powerflex_storage_pools = Domain1:Pool1,Domain2:Pool2
          customServiceConfigSecrets:
          - cinder-volume-powerflex-secrets
          debug:
            service: false
          networkAttachments:
          - storage
          replicas: 1
          resources: {}
...
```
Save and exit the file, the prompt tells you that the CR was edited. Verify the pod is running

```
oc get pods -n openstack | grep powerflex

[admin@rhadmin ~]$ oc get pods -n openstack | grep powerflex
cinder-volume-powerflex-0                                       2/2     Running     0             4d
[admin@rhadmin ~]$

```

### Prepare custom volume mappings for connector configuration 

PowerFlex SDC needs to be installed on the following nodes:
* Controller nodes 
* Compute nodes

To install SDC, deploy the Dell Container Storage Module operator. From the OpenShift operator hub, search Dell Container Storage modeules and pick the certified operator. Follow the wizard with all the deault values and install the operator. 

Follow the documentation located here https://dell.github.io/csm-docs/docs/deployment/csmoperator/drivers/powerflex/ to install a PowerFlex CSI driver for the remaining steps.

Create a config json file as follows

```ini
vi config.json
[
     {
         "username": "PF_Manager_login",
         "password": "PF_Manager_password",
         "systemID": "SYSTEM_ID",
         "endpoint": "PF_MGR-IP:443",
         "insecure": true,
         "isDefault": true,
         "mdm": "MDM_IP",
         "nasName": "none"
      }
 ]
```

Create a secret using

```
oc create secret generic vxflexos-config -n vxflexos --from-file=config=./config.json
```

From the sample available at https://github.com/dell/csm-operator/blob/main/samples/storage_csm_powerflex_v2110.yaml , create a CR file. Replace the MDM virtual IP by your environment.

```
vi storage_csm_powerflex_v292.yaml
... [Truncated]
    initContainers:
      - image: dellemc/sdc:4.5
        imagePullPolicy: IfNotPresent
        name: sdc
        envs:
          - name: MDM
            value: "MDM_IP"
... [Truncated]
```
Apply the CR file to the OpenShift cluster. 
```
oc create -f storage_csm_powerflex_v2110.yaml
```

Confirm the SDC are now running as containers on worker nodes
```
[admin@rhadmin ~]$ oc get pods -n vxflexos
NAME                                  READY   STATUS    RESTARTS   AGE
vxflexos-controller-cb9855d69-7p9d4   5/5     Running   0          31d
vxflexos-node-9f69t                   2/2     Running   2          31d
vxflexos-node-kgrkd                   2/2     Running   2          31d
vxflexos-node-vzfpm                   2/2     Running   2          31d
[admin@rhadmin ~]$
```

## Post deployment tasks

### Configure the connector

Before using attach/detach volume operations PowerFlex connector must be properly configured. Before the daaplane is installed, on each of the EDPM nodes do the following:

Create `/opt/emc/scaleio/openstack/connector.conf` if it does not exist.

```bash
$ touch /opt/emc/scaleio/openstack/connector.conf
```
Edit the `/opt/emc/scaleio/openstack/connector.conf` and populate it with passwords.

Example:

```
[powerflex]
san_password = powerflex_password

[powerflex-new]
san_password = powerflex_password
```

Before the dataplane is deployed, in the `dataplane-nodeset.yaml` enter the following
```
edpm_network_config_template: |
...
edpm_nova_extra_bind_mounts:
  - src: /opt/emc/scaleio/openstack/connector.conf
    dest: /opt/emc/scaleio/openstack/connector.conf
    options: "ro"
```

### Test the configured Backend
Finally, create a PowerFlex volume type and test if you can successfully create and attach volumes of that type.

Run the following command to check whether the cinder-volume service is up. 
```
[admin@rhadmin ~]$ oc rsh openstackclient
sh-5.1$ openstack volume service list
+------------------+-------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                | Zone | Status  | State | Updated At                 |
+------------------+-------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                  | nova | enabled | up    | 2024-12-16T09:24:50.000000 |
| cinder-volume    | cinder-volume-powerflex-0@powerflex | nova | enabled | up    | 2024-12-16T09:24:58.000000 |
| cinder-backup    | cinder-backup-0                     | nova | enabled | up    | 2024-12-16T09:24:49.000000 |
+------------------+-------------------------------------+------+---------+-------+----------------------------+

Create a volume type mapped to the deployed backend.
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type create powerflex1
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property volume_backend_name=tripleo_dellemc_powerflex powerflex1
```
Create a volume using the type created above without error to ensure the availability of the backend.
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume create --type powerflex1 --size 8 powerflex_volume1
```
Confirm the volume was created successfully
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume list
+--------------------------------------+-------------------+-----------+------+-------------+
| ID                                   | Name              | Status    | Size | Attached to |
+--------------------------------------+-------------------+-----------+------+-------------+
| 35808e76-c4cd-4ff6-8829-f16c76ebad37 | powerflex_volume1 | available | 8    |             |
+--------------------------------------+-------------------+-----------+------+-------------+
```
