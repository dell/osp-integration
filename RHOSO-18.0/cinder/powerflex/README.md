# Dell EMC PowerFlex Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

## Overview

This instructions provide detailed steps to enable Dell PowerFlex(formerly know as VxFlex OS/ScaleIO) as a Cinder block storage backend for ge backend with RHOSO 18.0.

**NOTICE**: This README represents only the **basic** necessary steps to enable Cinder to use a Dell PowerFlex as backend. It does not contain steps of other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- A working Red Hat OpenStack Services on OpenShift 18.0.14 environment.
- Dell PowerFlex 4.x cluster with at least one Storage Pool available.
- PowerFlex Storage Data Client (SDC) for RHEL9.4.

## Steps

### Prepare the environment for PowerFlex cinder backend
To enable the OpenStack workloads to consume PowerFlex as backend follow the below instructions:

* Create a Secret CR file `powerflex-secret.yaml` which contains PowerFlex backend credentials.
  
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

Apply the CR file. 
```yaml
oc create -f powerflex-secret.yaml
```

Verify the secret is created.
```yaml
oc describe secret/cinder-volume-powerflex-secrets
```

For full detailed instruction of all options please refer to [PowerFlex Backend Configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-powerflex-driver.html).

* Configure `OpenStackControlPlane` with PowerFlex details by editing the OpenStackControlPlane CR to add the PowerFlex backend information.

Use the `cinderVolumes` section as follows:
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

Verify the pod is running.
```
oc get pods -n openstack | grep powerflex
cinder-volume-powerflex-0                                       2/2     Running     0             4d
```

### PowerFlex SDC Deployment
PowerFlex requires the Storage Data Client (SDC) to be present on both control plane and data plane nodes.

**Install PowerFlex SDC on Control Plane (OpenShift)**:

PowerFlex SDC must be available on OpenShift worker nodes hosting OpenStack services which is achieved using Dell Container Storage Modules (CSM).
- Install Dell Container Storage Modules (CSM) Operator (pick the certified operator) from OperatorHub. 
- Deploy PowerFlex CSM using the link provided below so that SDC runs as a container. 
- Ensure SDC is connected to the PowerFlex system.
 
Follow the documentation located here https://dell.github.io/csm-docs/docs/getting-started/installation/openshift/powerflex/csmoperator/ for detailed steps.
 
Confirm the SDC are now running as containers on worker nodes.
```
oc get pods -n vxflexos
NAME                                   READY   STATUS    RESTARTS   AGE
vxflexos-controller-8657b7748c-5x8pc   5/5     Running   0          10d
vxflexos-node-b6c28                    2/2     Running   0          10d
vxflexos-node-cmrt2                    2/2     Running   0          10d
vxflexos-node-xvd58                    2/2     Running   0          10d
```

Verify that the SDC is successfully registered with PowerFlex.
- From PowerFlex Manager navigate → Block → Hosts.
- OpenShift worker nodes should appear as connected hosts.

**Install PowerFlex SDC on EDPM (dataplane) Nodes**:

PowerFlex SDC must also be installed on all EDPM compute nodes using the appropriate RPM.

To [install SDC](https://www.dell.com/support/manuals/en-ie/scaleio/powerflex_install_upgrade_guide_4.5.x/install-the-storage-data-client-on-a-linux-based-server?guid=guid-edaac602-f18b-4fe6-b825-ee09c6cdddd1&lang=en-us) on the compute (EDPM) nodes.
- Get the MDM IP's from PowerFlex.
- Copy the EMC-ScaleIO-sdc-*.rpm which corresponds to your RHEL OS level version to the EDPM nodes.
- Install the RPM as the root user.
- Repeat the steps for every remaining EDPM node.
- Ensure nodes are registered and connected in PowerFlex Manager. 

Verify where SDC installed nodes are connected in PowerFlex Manager.
- From PowerFlex Manager navigate → Block → Hosts.
- EDPM nodes appear as connected.

### Configure the connector
PowerFlex Connector Secret file required for volume attach and detach operations.This is consumed by os-brick during Nova attach workflows.
Before the dataplane is installed, on each of the EDPM nodes do the following:

* Create `/opt/emc/scaleio/openstack/connector.conf` if it does not exist.

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

* Before the dataplane is deployed, in the `dataplane-nodeset.yaml` enter the following:
```
edpm_network_config_template: |
...
edpm_nova_extra_bind_mounts:
       - dest: /opt/emc/scaleio/openstack
          options: ro
          src: /opt/emc/scaleio/openstack
        - dest: /usr/lib/python3.9/site-packages/os_brick/initiator/connectors/scaleio.py
          options: ro
          src: /opt/patches/os_brick/initiator/connectors/scaleio.py
```

**NOTICE**: The same powerflex-connector secret can be used with the ExtraMount so the nova-compute pod has access to the `connector.conf` file. This would eliminate the need to manually configure the file on the host.

* Create PowerFlex Connector Secret file. 
```yaml  
apiVersion: v1
kind: Secret
metadata:
  name: powerflex-connector-conf-file
  labels:
    component: cinder-volume
    service: cinder
type: Opaque
stringData:
  connector.conf: |
    [powerflex]
    san_password = <PF_Manager_password>
```

Apply the CR file. 
```yaml
oc create -f powerflex-connector-secret.yaml
```

Verify the secret is created.
```yaml
oc describe secret/powerflex-connector-conf-file
```

In the OpenStackControlPlane CR add an extraMount specification as follows:

* Mount connector secret using under extraMounts:
```
...
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
          secretName: powerflex-connector-conf-file
    name: v1
    region: r1
...
```

Save and exit the file. Apply the CR.

### Test the configured Backend
Finally, create a PowerFlex volume type and test if you can successfully create and attach volumes of that type.

* Run the following command to check whether the cinder-volume service is up. 
```
oc rsh openstackclient
sh-5.1$ openstack volume service list
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                         | Zone | Status  | State | Updated At                 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                           | nova | enabled | up    | 2026-01-28T09:25:27.000000 |
| cinder-backup    | cinder-backup-0                              | nova | enabled | up    | 2026-01-28T09:25:21.000000 |
| cinder-volume    | cinder-volume-powerflex-0@powerflex          | nova | enabled | up    | 2026-01-28T09:25:20.000000 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
```

* Create volume type for the powerflex
```
sh-5.1$ openstack volume type create powerflex
sh-5.1$ openstack volume type set --property volume_backend_name=powerflex powerflex
```

* Check if a volume type 'powerflex' is created.
```
sh-5.1$ openstack volume type show powerflex
+--------------------+-------------------------------------------------------+
| Field              | Value                                                 |
+--------------------+-------------------------------------------------------+
| access_project_ids | None                                                  |
| description        |                                                       |
| id                 | b7bff211-a6cd-435d-aa27-2187c3e46f68                  |
| is_public          | True                                                  |
| name               | powerflex1                                            |
| properties         | pool_name='PD1:SP1', volume_backend_name='powerflex ' |
| qos_specs_id       | None                                                  |
+--------------------+-------------------------------------------------------+
```

* Create a volume using the type created above without error to ensure the availability of the backend.
```
sh-5.1$ openstack volume create --type powerflex --size 8 pf_vol1
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2026-01-28T09:24:35.074112           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | e13cabcd-4ed1-425a-8f95-48eef95368fc |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | pf_vol1                              |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 8                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | Available                            |
| type                | powerflex                            |
| updated_at          | None                                 |
| user_id             | 9356914d779d443f8cc09ebdf62ef7c9     |
+---------------------+--------------------------------------+
```
Confirm the volume was created successfully
```
sh-5.1$ openstack volume list
+--------------------------------------+---------+-----------+------+-------------+
| ID                                   | Name    | Status    | Size | Attached to |
+--------------------------------------+---------+-----------+------+-------------+
| e13cabcd-4ed1-425a-8f95-48eef95368fc | pf_vol1 | available |    8 |             |
+--------------------------------------+---------+-----------+------+-------------+
```
