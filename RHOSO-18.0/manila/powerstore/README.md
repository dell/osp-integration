# Dell PowerStore Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

## Overview

These instructions provide detailed steps on how to enable Dell PowerStore storage backend with RHOSO 18.0.

**NOTICE**: this README represents only the **basic** necessary steps to enable Manila to use a Dell PowerStore as backend. It does not contain steps of other components in the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- A working Red Hat OpenStack Services on OpenShift environment with a version 18.0.14 or greater.
- Dell PowerStore (4.2) storage cluster.

## Steps

### Configure Dell PowerStore details
Manila uses a DriverBackedShare mechanism to manage share backends. You must create a Secret containing PowerStore credentials and then reference this secret in OpenStack control plane configuration under the Manila section.

* Create a Secret CR file manila-powerstore-secrets.yaml.
  
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: manila-share
    service: manila
  name: manila-powerstore-secrets
stringData:
  powerstore-secrets.conf: |
    [powerstore]
    emc_nas_server = <DellPowerStoreIP>
    emc_nas_login = <DellPowerStoreUser>
    emc_nas_password = <DellPowerStorePassword>
type: Opaque
```

Apply the CR file.
```yaml
oc create -f manila-powerstore-secrets.yaml
secret/manila-powerstore-secrets created
```

* Verify the secret is created.
```yaml
oc get secret/manila-powerstore-secrets -o jsonpath={.data.powerstore-secrets} | base64 -d
[powerstore]
emc_nas_server = <DellPowerStoreIP>
emc_nas_login = <DellPowerStoreUser>
emc_nas_password = <DellPowerStorePassword>
```
For full detailed instructions of available options please refer to [PowerStore Backend Configuration](https://docs.openstack.org/manila/latest/configuration/shared-file-systems/drivers/dell-emc-powerstore-driver.html).

### Configure Manila to Use the Dell PowerStore Backend
Edit your OpenStackControlPlane CR file.

* Add the following under spec.manila.manilaShares:
```
...
       manilaShares:
        powerstore:
          customServiceConfig: |
            [DEFAULT]
            enabled_share_backends = powerstore
            enabled_share_protocols = NFS,CIFS
            [powerstore]
            driver_handles_share_servers = False
            emc_share_backend = powerstore
            share_backend_name = powerstore
            share_driver = manila.share.drivers.dell_emc.driver.EMCShareDriver
            dell_nas_server = <NAS_SERVER_NAME>
          customServiceConfigSecrets:
          - manila-share-powerstore-secrets
          networkAttachments:
          - storage
...
```
* Save and exit the file. Apply the CR. 
```
oc apply -f openstackcontrolplane.yaml
```
* Verify the pod is running.

```
oc get pods -n openstack | grep powerstore
manila-share-powerstore-0                                         2/2     Running     0             21h
```

### Test the configured Backend

* Ensure Manila share service is up and running.
Open a session on the openstackclient pod.
```
oc rsh openstackclient
```
* Run the following command to check whether the Manila services is up:
```
sh-5.1$ openstack share service list
+------------------+-----------------------+------+---------+-------+----------------------------+
| Binary           | Host                  | Zone | Status  | State | Updated At                 |
+------------------+-----------------------+------+---------+-------+----------------------------+
| manila-scheduler | hostgroup             | nova | enabled | up    | 2026-01-13T07:12:05.446113 |
| manila-share     | hostgroup@powerstore  | nova | enabled | up    | 2026-01-13T07:12:13.045688 |
+------------------+-----------------------+------+---------+-------+----------------------------+
```

* Create a Share Type.

Finally, create a PowerStore share type and test if you can successfully create and attach shares of that type.
Create a share type name powerstore.
```
openstack share type create powerstore false
```
Map it to backend name that you previously defined.
```
openstack share type set --property share_backend_name=powerstore powerstore
```
Verify that the share type exists and is successfully configured.
```
sh-5.1$ openstack share type show powerstore
+----------------------+-------------------------------------------+
| Field                | Value                                     |
+----------------------+-------------------------------------------+
| id                   | 801bd773-ca14-459b-8742-76c551b40008      |
| name                 | powerstore                                |
| visibility           | public                                    |
| is_default           | False                                     |
| required_extra_specs | driver_handles_share_servers : False      |
| optional_extra_specs | share_backend_name : powerstore           |
| description          | None                                      |
+----------------------+-------------------------------------------+
```
* Validate a Share Type.

Create a volume using the type created above and ensure there is no error to confirm the availability of the backend.
```
sh-5.1$ openstack share create --name testshare --share-type powerstore nfs 1

+---------------------------------------+--------------------------------------+
| Field                                 | Value                                |
+---------------------------------------+--------------------------------------+
| access_rules_status                   | active                               |
| availability_zone                     | None                                 |
| create_share_from_snapshot_support    | True                                 |
| created_at                            | 2026-01-13T08:06:44.316732           |
| description                           | None                                 |
| has_replicas                          | False                                |
| host                                  |                                      |
| id                                    | 737c5f25-b678-4731-88ff-647fd203db3e |
| is_public                             | False                                |
| is_soft_deleted                       | False                                |
| metadata                              | {}                                   |
| mount_snapshot_support                | False                                |
| name                                  | testshare                            |
| progress                              | None                                 |
| project_id                            | 9a15a3e6d6254c3d9f88d086b6b0d40a     |
| replication_type                      | None                                 |
| revert_to_snapshot_support            | True                                 |
| scheduled_to_be_deleted_at            | None                                 |
| share_group_id                        | None                                 |
| share_network_id                      | None                                 |
| share_proto                           | NFS                                  |
| share_server_id                       | None                                 |
| share_type                            | 801bd773-ca14-459b-8742-76c551b40008 |
| share_type_name                       | powerstore                           |
| size                                  | 1                                    |
| snapshot_id                           | None                                 |
| snapshot_support                      | True                                 |
| source_share_group_snapshot_member_id | None                                 |
| status                                | creating                             |
| task_state                            | None                                 |
| user_id                               | c106b653262a483898a56b78a07c478c     |
| volume_type                           | powerstore                           |
+---------------------------------------+--------------------------------------+

sh-5.1$ openstack share create --name testshare2 --share-type powerstore cifs 1
+---------------------------------------+--------------------------------------+
| Field                                 | Value                                |
+---------------------------------------+--------------------------------------+
| access_rules_status                   | active                               |
| availability_zone                     | None                                 |
| create_share_from_snapshot_support    | True                                 |
| created_at                            | 2026-01-13T08:08:01.825146           |
| description                           | None                                 |
| has_replicas                          | False                                |
| host                                  |                                      |
| id                                    | ae0dbce1-fd72-4246-b64a-d805adf22719 |
| is_public                             | False                                |
| is_soft_deleted                       | False                                |
| metadata                              | {}                                   |
| mount_snapshot_support                | False                                |
| name                                  | testshare2                           |
| progress                              | None                                 |
| project_id                            | 9a15a3e6d6254c3d9f88d086b6b0d40a     |
| replication_type                      | None                                 |
| revert_to_snapshot_support            | True                                 |
| scheduled_to_be_deleted_at            | None                                 |
| share_group_id                        | None                                 |
| share_network_id                      | None                                 |
| share_proto                           | CIFS                                 |
| share_server_id                       | None                                 |
| share_type                            | 801bd773-ca14-459b-8742-76c551b40008 |
| share_type_name                       | powerstore                           |
| size                                  | 1                                    |
| snapshot_id                           | None                                 |
| snapshot_support                      | True                                 |
| source_share_group_snapshot_member_id | None                                 |
| status                                | creating                             |
| task_state                            | None                                 |
| user_id                               | c106b653262a483898a56b78a07c478c     |
| volume_type                           | powerstore                           |
+---------------------------------------+--------------------------------------+
```
Confirm the shares are created successfully.
```
sh-5.1$ openstack share list

+--------------------------------------+-----------+------+-------------+-----------+-----------+-----------------+-----------------------------------+-------------------+
| ID                                   | Name      | Size | Share Proto | Status    | Is Public | Share Type Name | Host                              | Availability Zone |
+--------------------------------------+-----------+------+-------------+-----------+-----------+-----------------+-----------------------------------+-------------------+
| 737c5f25-b678-4731-88ff-647fd203db3e |testshare  |    1 | CIFS        | available | False     | powerstore      | hostgroup@powerstore#powerstore   | nova              |
| ae0dbce1-fd72-4246-b64a-d805adf22719 |testshare2 |    1 | NFS         | available | False     | powerstore      | hostgroup@powerstore#powerstore   | nova              |
+--------------------------------------+-----------+------+-------------+-----------+-----------+-----------------+-----------------------------------+-------------------+
```
**NOTICE**: Same share type (powerstore), can be used for both CIFS and NFS protocol.
