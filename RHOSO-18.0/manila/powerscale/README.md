# Dell PowerScale Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

## Overview

These instructions provide detailed steps on how to enable Dell PowerScale (formerly known as Isilon) storage backend with RHOSO 18.0.

**NOTICE**: this README represents only the **basic** necessary steps to enable Manila to use a Dell Powerscale as backend. It does not contain steps of other components in the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- A working Red Hat OpenStack Services on OpenShift 18.0.14 environment.
- Dell PowerScale (9.13) storage cluster running a supported OneFS version.

## Steps

### Configure Dell PowerScale details
Manila uses a DriverBackedShare mechanism to manage share backends. You must create a Secret containing PowerScale credentials and then reference this secret in the Manila control-plane configuration.

* Create a Secret CR file manila-powerscale-secrets.yaml.
  
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: manila-share
    service: manila
  name: manila-powerscale-secrets
stringData:
  powerscale-secrets.conf: |
    [powerscale]
    emc_nas_server = <DellPowerScaleIP>
    emc_nas_login = <DellPowerScaleUser>
    emc_nas_password = <DellPowerScalePassword>
type: Opaque
```

* Apply the CR file.
```yaml
oc create -f manila-powerscale-secrets.yaml
secret/manila-powerscale-secrets created
```

* Verify the secret is created.
```yaml
oc get secret/manila-powerscale-secrets -o jsonpath={.data.powerscale-secrets} | base64 -d
[powerscale]
emc_nas_server = <DellPowerScaleIP>
emc_nas_login = <DellPowerScaleUser>
emc_nas_password = <DellPowerScalePassword>
```
For full detailed instructions of available options please refer to [PowerScale Backend Configuration](https://docs.openstack.org/manila/latest/configuration/shared-file-systems/drivers/dell-emc-powerscale-driver.html).

### Configure Manila to Use the Dell PowerScale Backend
Edit your OpenStackControlPlane CR file (openstack_control_plane.yaml).

* Add the following under spec.manila.manilaShares:
```
...
      manilaShares:
        powerscale:
          customServiceConfig: |
            [DEFAULT]
            debug = true
            enabled_share_backends = powerscale
            [powerscale]
            driver_handles_share_servers = False
            emc_share_backend = isilon
            share_backend_name = powerscale
            enabled_share_protocols = NFS,CIFS
            share_driver = manila.share.drivers.dell_emc.driver.EMCShareDriver
            emc_nas_root_dir = /ifs/manila
            emc_ssl_cert_verify = False
            emc_nas_server_port = 8080
          customServiceConfigSecrets:
          - manila-powerscale-secrets
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
oc get pods -n openstack | grep powerscale
manila-share-powerscale-0                                         2/2     Running     0             21h
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
| manila-scheduler | hostgroup             | nova | enabled | up    | 2025-12-04T17:06:48.034510 |
| manila-share     | hostgroup@powerscale  | nova | enabled | up    | 2025-12-04T17:06:46.526608 |
+------------------+-----------------------+------+---------+-------+----------------------------+
```

* Create a Share Type.

Finally, create a PowerScale share type and test if you can successfully create and attach shares of that type.
Create a share type name powerscale.
```
openstack share type create --driver-handles-share-servers false powerscale
```
Map it to backend.
```
openstack share type set --property share_backend_name=powerscale powerscale
```
Verify that the share type is exists and is successfully configured.
```
sh-5.1$ openstack share type show powerscale
+----------------------+-------------------------------------------+
| Field                | Value                                     |
+----------------------+-------------------------------------------+
| id                   | 6c7c23ea-926c-4158-918b-5fcef1c73cc4      |
| name                 | powerscale                                |
| visibility           | public                                    |
| required_extra_specs | driver_handles_share_servers : False      |
| optional_extra_specs | share_backend_name : powerscale           |
| description          | None                                      |
+----------------------+-------------------------------------------+
```
* Validate a Share Type.

Create a volume using the type created above and ensure there is no error to confirm the availability of the backend.
```
sh-5.1$ openstack share create --name testshare --share-type powerscale nfs 1
+---------------------------------------+--------------------------------------+
| Field                                 | Value                                |
+---------------------------------------+--------------------------------------+
| access_rules_status                   | active                               |
| availability_zone                     | None                                 |
| created_at                            | 2025-12-05T06:26:58.213059           |
| description                           | None                                 |
| has_replicas                          | False                                |
| id                                    | 849d064a-1586-4f4e-8c39-4597a5dbdc22 |
| name                                  | testshare                            |
| progress                              | None                                 |
| project_id                            | 9a15a3e6d6254c3d9f88d086b6b0d40a     |
| replication_type                      | None                                 |
| share_group_id                        | None                                 |
| share_network_id                      | None                                 |
| share_proto                           | NFS                                  |
| share_server_id                       | None                                 |
| share_type                            | 6c7c23ea-926c-4158-918b-5fcef1c73cc4 |
| share_type_name                       | powerscale                           |
| size                                  | 1                                    |
| snapshot_id                           | None                                 |
| source_share_group_snapshot_member_id | None                                 |
| status                                | creating                             |
| task_state                            | None                                 |
| user_id                               | c106b653262a483898a56b78a07c478c     |
| volume_type                           | powerscale                           |
+---------------------------------------+--------------------------------------+

sh-5.1$ openstack share create --name testshare2 --share-type powerscale cifs 1
+---------------------------------------+--------------------------------------+
| Field                                 | Value                                |
+---------------------------------------+--------------------------------------+
| access_rules_status                   | active                               |
| availability_zone                     | None                                 |
| created_at                            | 2025-12-05T10:05:46.737706           |
| description                           | None                                 |
| has_replicas                          | False                                |
| id                                    | a875ef4a-d8ef-47f7-862c-1c087cf2dafe |
| name                                  | testshare2                           |
| progress                              | None                                 |
| project_id                            | 9a15a3e6d6254c3d9f88d086b6b0d40a     |
| replication_type                      | None                                 |
| share_group_id                        | None                                 |
| share_network_id                      | None                                 |
| share_proto                           | CIFS                                 |
| share_server_id                       | None                                 |
| share_type                            | 6c7c23ea-926c-4158-918b-5fcef1c73cc4 |
| share_type_name                       | powerscale                           |
| size                                  | 1                                    |
| snapshot_id                           | None                                 |
| source_share_group_snapshot_member_id | None                                 |
| status                                | creating                             |
| task_state                            | None                                 |
| user_id                               | c106b653262a483898a56b78a07c478c     |
| volume_type                           | powerscale                           |
+---------------------------------------+--------------------------------------+
```
Confirm the share was created successfully.
```
sh-5.1$ openstack share list
+--------------------------------------+-----------+------+-------------+-----------+-----------+-----------------+-----------------------------------+-------------------+
| ID                                   | Name      | Size | Share Proto | Status    | Is Public | Share Type Name | Host                              | Availability Zone |
+--------------------------------------+-----------+------+-------------+-----------+-----------+-----------------+-----------------------------------+-------------------+
| a875ef4a-d8ef-47f7-862c-1c087cf2dafe |testshare  |    1 | CIFS        | available | False     | powerscale      | hostgroup@powerscale#powerscale   | nova              |
| 849d064a-1586-4f4e-8c39-4597a5dbdc22 |testshare2 |    1 | NFS         | available | False     | powerscale      | hostgroup@powerscale#powerscale   | nova              |
+--------------------------------------+-----------+------+-------------+-----------+-----------+-----------------+-----------------------------------+-------------------+
```
**NOTICE**: Same share type (powerscale), can be used for both CIFS and NFS protocol.
