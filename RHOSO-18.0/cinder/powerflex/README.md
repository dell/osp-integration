# Dell EMC PowerFlex Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

Deployment tools for Dell EMC PowerFlex (formerly VxFlex OS/ScaleIO) support in Red Hat OpenStack Services on OpenShift (RHOSO) 18.0

## Overview

This instruction provides detailed steps on how to enable PowerFlex.

**NOTICE**: this README represents only the **basic** steps necessary to enable PowerFlex driver. It does not contain steps other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- Red Hat OpenStack Services on OpenShift 18.0 with RHEL 9.4.
- PowerFlex 4.x cluster.
- PowerFlex Storage Data Client (SDC) for RHEL9.4.

## Steps

### Prepare the environment for PowerFlex cinder backend
To enable the OpenStack workloads to consume PowerFlex as backend follow the below instructions:

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
    san_ip=<PF_Manager_IP>
    san_login=<PF_Manager_login>
    san_password=<PF_Manager_password>
type: Opaque
```

* Apply the CR file 
```yaml
oc create -f powerflex-secret.yaml
```

* Verify the secret is created
```yaml
oc describe secret/cinder-volume-powerflex-secrets
```
For full detailed instruction of all options please refer to [PowerFlex Backend Configuration](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/dell-emc-powerflex-driver.html).

* In the OpenStackControlPlane CR, esnure you have the below section

Use the cinderVolumes section as follows
```
...
cinderVolumes:
        powerflex:
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
Save and exit the file. Apply the CR. 
```
oc apply -f openstackcontrolplane.yaml
```
Verify the pod is running

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

To install SDC on the controller nodes, deploy the Dell Container Storage Module operator. From the OpenShift operator hub, search Dell Container Storage modeules and pick the certified operator. Follow the wizard with all the deault values and install the operator. 

Follow the documentation located here https://dell.github.io/csm-docs/docs/deployment/csmoperator/drivers/powerflex/ to install a PowerFlex CSI driver for the remaining steps.

Create a config json file as follows
* username : Username for accessing PowerFlex system. If authorization is enabled, username will be ignored.
* password : Password for accessing PowerFlex system. If authorization is enabled, password will be ignored. 
* systemID : PowerFlex system name or ID.	
* endpoint :	REST API gateway HTTPS endpoint/PowerFlex Manager public IP for PowerFlex system. 
* skipCertificateValidation :	Determines if the driver is going to validate certs while connecting to PowerFlex REST API interface. 
* isDefault: An array having isDefault=true is for backward compatibility.  
* mdm: mdm defines the MDM(s) that SDC should register with on start. This should be a list of MDM IP addresses or hostnames separated by comma.	
* nasName:	nasName defines what NAS should be used for NFS volumes. NFS volumes are supported on arrays version >=4.0.x	

```ini
vi config.json
[
     {
         "username": "<PF_Manager_login>",
         "password": "<PF_Manager_password>",
         "systemID": "<SYSTEM_ID>",
         "endpoint": "<PF_MGR-IP>:443",
         "insecure": true,
         "isDefault": true,
         "mdm": "<MDM_IP>",
         "nasName": "none"
      }
 ]
```

Create a secret using

```
oc create secret generic vxflexos-config -n vxflexos --from-file=config=./config.json
```

From the sample available at https://github.com/dell/csm-operator/blob/main/samples/storage_csm_powerflex_v2130.yaml , create a CR file. Replace the MDM virtual IP by your environment.

```
vi storage_csm_powerflex_v2130.yaml
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
oc create -f storage_csm_powerflex_v2130.yaml
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

To [install SDC](https://www.dell.com/support/manuals/en-ie/scaleio/powerflex_install_upgrade_guide_4.5.x/install-the-storage-data-client-on-a-linux-based-server?guid=guid-edaac602-f18b-4fe6-b825-ee09c6cdddd1&lang=en-us) on the compute(EDPM) nodes

* Get the MDM IP's from PowerFlex
* Copy the EMC-ScaleIO-sdc-*.rpm which corresponds to your RHEL OS level version to the EDPM nodes
* Install the RPM as the root user
* Repeat the steps for every remaining EDPM node
* Confirm the SDC is connected to the PowerFlex system by logging on to PowerFlex Manager, navigate to Block → Hosts 

## Configure the connector

Before using attach/detach volume operations PowerFlex connector must be properly configured. Before the dataplane is installed, on each of the EDPM nodes do the following:

Create `/opt/emc/scaleio/openstack/connector.conf` if it does not exist.

```bash
$ touch /opt/emc/scaleio/openstack/connector.conf
```
Edit the `/opt/emc/scaleio/openstack/connector.conf` and populate it with passwords.

Example:

```
[powerflex]
san_password = <PF_Manager_password>

[powerflex-new]
san_password = <PF_Manager_password>
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

Create PowerFlex connector secret 
```yaml
vi powerflex-connector-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: powerflex-connector-conf-files
stringData:
  connector.conf: |
    [powerflex]
    san_password=<PF_Manager_password>
type: Opaque
```

In the OpenStackControlPlane CR and add an extraMount specification as follows
```
...  
spec:
  extraMounts:
  - extraVol:
    - extraVolType: powerflex
      mounts:
      - mountPath: /opt/emc/scaleio/openstack
        name: powerflex-connector
        readOnly: true
      propagation:
      - CinderVolume
      volumes:
      - name: powerflex-connector
        secret:
          secretName: powerflex-connector-conf-files
    name: v1
    region: r1
 ...
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
```
Check if a volume type 'powerflex' is created.
```
sh-5.1$ openstack volume type show powerflex
+--------------------+------------------------------------------------------------+
| Field              | Value                                                      |
+--------------------+------------------------------------------------------------+
| access_project_ids | None                                                       |
| description        |                                                            |
| id                 | e00d4970-25fa-41bc-a962-261f886ddd8e                       |
| is_public          | True                                                       |
| name               | powerflex                                                  |
| properties         | pool_name='PD-1:SP-SSD-1', volume_backend_name='powerflex' |
| qos_specs_id       | None                                                       |
+--------------------+------------------------------------------------------------+

sh-5.1$ openstack volume type list
+--------------------------------------+--------------------------------+-----------+
| ID                                   | Name                           | Is Public |
+--------------------------------------+--------------------------------+-----------+
| e8a9dcd6-e063-43ea-8132-e6e5e801530e | powerflex                      | True      |

```

Create a volume using the type created above without error to ensure the availability of the backend.
```
sh-5.1$ openstack volume create --type powerflex --size 8 pf_vol1
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-12-17T04:52:49.774884           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 68645dfc-b946-4d76-9875-195095c0e9ec |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | pf_vol1                              |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 8                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | powerflex                            |
| updated_at          | None                                 |
| user_id             | 3bbbbe7f703b4c75a8d3030b7ed43d84     |
+---------------------+--------------------------------------+

```
Confirm the volume was created successfully
```
sh-5.1$ openstack volume list
+--------------------------------------+---------+--------+------+-----------------+
| ID                                   | Name    | Status     | Size | Attached to |
+--------------------------------------+---------+--------+------+-----------------+
| 68645dfc-b946-4d76-9875-195095c0e9ec | pf_vol1 | available  | 8    |             |
+--------------------------------------+---------+--------+------+-----------------+
```
