# Dell PowerStore Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

## Overview

These instructions provide detailed steps on how to enable the Dell PowerStore backend with RHOSO 18.0.

**NOTICE**: This README represents only the **basic** necessary steps to enable Cinder to use a Dell PowerStore array as a backend. It does not contain steps of other components in the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- A working Red Hat OpenStack Services on OpenShift 18.0 environment.
- Dell PowerStore running a supported PowerStoreOS version.

## Steps

### Configure Dell PowerStore storage details
This section is broken down into two sub-sections. First of all, we'll cover how to configure a PowerStore FC based backend while we'll talk about PowerStore iSCSI based backend in the latter.
Finally, we'll discover how to configure a multi-backend using both PowerStore FC and iSCSI with an example.

For detailed instructions and supported configuration options, please refer to [Dell PowerStore Backend Configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-powerstore-driver.html).

#### PowerStore FC based backend configuration

**NOTICE: To isolate FC and iSCSI credentials, it is recommended to use one secret per backend type. This prevents unnecessary reconfiguration or restarts of unrelated cinder-volume services when a secret is updated.**
 
In this section, we'll create one secret per backend type but depending on your configuration, you might have to create only one of them.

* Create a Secret CR file named **cinder-volume-powerstore-fc-secrets.yaml** (or a name suitable for your environment).
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-powerstore-fc-secrets
stringData:
  powerstore-secrets: |
    [powerstoreFC]
    san_ip = <DellPowerStoreIP>
    san_login = <DellPowerStoreAdminUser>
    san_password = <DellPowerStoreAdminPasswd>
type: Opaque
```

* Create the secret.
```
oc create -f cinder-volume-powerstore-fc-secrets.yaml
secret/cinder-volume-powerstore-fc-secrets created
```

* Verify the secret is created.
```
oc get secrets/cinder-volume-powerstore-fc-secrets -o jsonpath={.data.powerstore-secrets} | base64 -d
[powerstoreFC]
san_ip = <DellPowerStoreIP>
san_login = <DellPowerStoreAdminUser>
san_password = <DellPowerStoreAdminPasswd>
```

#### PowerStore iSCSI based backend configuration
* Create a Secret CR file named **cinder-volume-powerstore-iscsi-secrets.yaml** or choose a name that suits your needs.
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-powerstore-iscsi-secrets
stringData:
  powerstore-secrets: |
    [powerstoreISCSI]
    san_ip = <DellPowerStoreIP>
    san_login = <DellPowerStoreAdminUser>
    san_password = <DellPowerStoreAdminPasswd>
type: Opaque
```

* Create the secret.
```
oc create -f cinder-volume-powerstore-iscsi-secrets.yaml
secret/cinder-volume-powerstore-iscsi-secrets created
```

* Verify the secret is created.
```
oc get secrets/cinder-volume-powerstore-iscsi-secrets -o jsonpath={.data.powerstore-secrets} | base64 -d
[powerstoreISCSI]
san_ip = <DellPowerStoreIP>
san_login = <DellPowerStoreAdminUser>
san_password = <DellPowerStoreAdminPasswd>
```

### Configure Cinder to consume the Dell PowerStore backend
In this section, PowerStore configuration is added to the OpenStack control plane CR. 

**NOTICE: As already discussed, you may have to adapt this section depending on your environment, especially if you use FC and iSCSI or only one backend type. In multi-backend configurations, each backend must use a unique
volume_backend_name.**

* Edit the control plane CR file (_openstack_control_plane.yaml_) and add the Dell PowerStore configuration under **spec.cinder.cinderVolumes** section.

> PowerStore FC based backend configuration
```yaml
...
      cinderVolumes:
        powerstore:
          customServiceConfig: |
            [powerstoreFC]
            volume_backend_name = powerstore
            volume_driver   = cinder.volume.drivers.dell_emc.powerstore.driver.PowerStoreDriver
            storage_protocol = FC
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-powerstore-fc-secrets
          networkAttachments:
          - storage
          - storageMgmt
          replicas: 1
          resources: {}
...
```

> PowerStore iSCSI based backend configuration
```yaml
...
      cinderVolumes:
        powerstore:
          customServiceConfig: |
            [powerstoreISCSI]
            volume_backend_name = powerstore
            volume_driver   = cinder.volume.drivers.dell_emc.powerstore.driver.PowerStoreDriver
            storage_protocol = iSCSI
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-powerstore-iscsi-secrets
          networkAttachments:
          - storage
          - storageMgmt
          replicas: 1
          resources: {}
...
```

* Save and exit the file. Apply the CR. 
```
oc apply -f openstack_control_plane.yaml
```

### Configure multi-backend 
This section provides an example of configuring two PowerStore backends, using FC and iSCSI. It assumes that you have created one secret per backend type.

**NOTICE: You can add other backends as well but this documentation won't cover it.**

* Edit the control plane CR file (_openstack_control_plane.yaml_) and add the Dell PowerStore configuration under **spec.cinder.cinderVolumes** section.

```yaml
...
      cinderVolumes:
        powerstoreISCSI:
          customServiceConfig: |
            [powerstoreISCSI]
            volume_backend_name = powerstoreISCSI
            volume_driver   = cinder.volume.drivers.dell_emc.powerstore.driver.PowerStoreDriver
            storage_protocol = iSCSI
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-powerstore-iscsi-secrets
          networkAttachments:
          - storage
          - storageMgmt
          replicas: 1
          resources: {}
        powerstoreFC:
          customServiceConfig: |
            [powerstoreFC]
            volume_backend_name = powerstoreFC
            volume_driver   = cinder.volume.drivers.dell_emc.powerstore.driver.PowerStoreDriver
            storage_protocol = FC
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-powerstore-fc-secrets
          networkAttachments:
          - storage
          - storageMgmt
          replicas: 1
          resources: {}
        <OtherBackend>:
...
```

### Test the configured Backend

#### Single backend validation
In this section, we'll validate the cinder configuration by following the steps below:
* Make sure the cinder-volume service is up and running.
* Create a specific volume type which will be used by Cinder.
* Create a volume on the Dell PowerStore using that volume type.


* Open a session on the **openstackclient** pod.
```
oc rsh openstackclient
```

* Confirm the cinder-volume service for Dell PowerStore as a backend is up and running.
```
sh-5.1$ openstack volume service list
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                         | Zone | Status  | State | Updated At                 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                           | nova | enabled | up    | 2026-01-16T15:34:39.000000 |
| cinder-volume    | cinder-volume-powerstore-0@powerstore        | nova | enabled | up    | 2026-01-16T15:34:37.000000 |
| cinder-backup    | cinder-backup-0                              | nova | enabled | up    | 2026-01-16T15:34:39.000000 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
```

* Create a volume type named powerstore and map it to the Dell PowerStore backend.
```
sh-5.1$ openstack volume type create --property volume_backend_name=powerstore powerstore
```

Confirm the volume type exists and is successfully configured.
```
sh-5.1$ openstack volume type show powerstore
+--------------------+------------------------------------------------------------+
| Field              | Value                                                      |
+--------------------+------------------------------------------------------------+
| access_project_ids | None                                                       |
| description        |                                                            |
| id                 | a5c06b96-77e6-40de-90d9-330567a89fae                       |
| is_public          | True                                                       |
| name               | powerstore                                                 |
| properties         | volume_backend_name='powerstore'                           |
| qos_specs_id       | None                                                       |
+--------------------+------------------------------------------------------------+
```

* Create a volume using the type created above without error to ensure the availability of the backend.
```
sh-5.1$ openstack volume create --type powerstore --size 1 vol
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2026-01-23T06:33:35.346017           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 8aaa6f1b-b8b9-468b-8491-23d31cfb7084 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | vol                                  |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | powerstore                           |
| updated_at          | None                                 |
| user_id             | 9356914d779d443f8cc09ebdf62ef7c9     |
+---------------------+--------------------------------------+
```

Confirm the volume was created successfully.
```
sh-5.1$ openstack volume list
+--------------------------------------+------+-----------+------+-------------+
| ID                                   | Name | Status    | Size | Attached to |
+--------------------------------------+------+-----------+------+-------------+
| 8aaa6f1b-b8b9-468b-8491-23d31cfb7084 | vol  | available |    1 |             |
+--------------------------------------+------+-----------+------+-------------+
```

#### Multi-backend validation
To conclude our multi-backend configuration example, we'll execute the same steps against two separate PowerStore backend (_powerstoreFC_ and _powerstoreISCSI_).

* Open a session on the **openstackclient** pod.
```
oc rsh openstackclient
```

* Confirm the cinder-volume service for all Dell PowerStore backend are up and running
```
sh-5.1$ openstack volume service list
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                         | Zone | Status  | State | Updated At                 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                           | nova | enabled | up    | 2026-01-16T15:34:39.000000 |
| cinder-volume    | cinder-volume-powerstore-0@powerstoreISCSI   | nova | enabled | up    | 2026-01-16T15:34:37.000000 |
| cinder-volume    | cinder-volume-powerstore-0@powerstoreFC      | nova | enabled | up    | 2026-01-16T15:34:37.000000 |
| cinder-backup    | cinder-backup-0                              | nova | enabled | up    | 2026-01-16T15:34:39.000000 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
```

* Create a volume type named powerstoreFC and powerstoreISCSI and map it to the respective backends.
```
sh-5.1$ openstack volume type create --property volume_backend_name=powerstoreFC powerstoreFC

sh-5.1$ openstack volume type create --property volume_backend_name=powerstoreISCSI powerstoreISCSI
```

* Confirm both volume types exist and are successfully configured.
```
sh-5.1$ openstack volume type show powerstoreFC
+--------------------+------------------------------------------------------------+
| Field              | Value                                                      |
+--------------------+------------------------------------------------------------+
| access_project_ids | None                                                       |
| description        |                                                            |
| id                 | fde329b2-be30-473b-aef1-5a5059a552dd                       |
| is_public          | True                                                       |
| name               | powerstoreFC                                               |
| properties         | volume_backend_name='powerstoreFC'                         |
| qos_specs_id       | None                                                       |
+--------------------+------------------------------------------------------------+
```

```
sh-5.1$ openstack volume type show powerstoreISCSI
+--------------------+-----------------------------------------+
| Field              | Value                                   |
+--------------------+-----------------------------------------+
| access_project_ids | None                                    |
| description        |                                         |
| id                 | 581a4cc1-6ba3-40e9-a46d-dc736758630d    |
| is_public          | True                                    |
| name               | powerstoreISCSI                         |
| properties         | volume_backend_name='powerstoreISCSI'   |
| qos_specs_id       | None                                    |
+--------------------+-----------------------------------------+

```
* Validate a volume type.
Create a volume using the volume-types created above and ensure there is no error to confirm the availability of the backend.
```
sh-5.1$ openstack volume create --type powerstoreFC --size 1 vol-FC
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2026-01-23T08:24:05.129270           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 3260be2f-d547-469e-ab63-785646bdd09e |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | vol-FC                               |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | powerstoreFC                         |
| updated_at          | None                                 |
| user_id             | 9356914d779d443f8cc09ebdf62ef7c9     |
+---------------------+--------------------------------------+
```

```
sh-5.1$ openstack volume create --type powerstoreISCSI --size 1 vol-ISCSI
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2026-01-23T08:38:13.205530           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 19a09d9a-1bd8-4c08-97e0-344a9da38caa |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | vol-ISCSI                            |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | powerstoreISCSI                      |
| updated_at          | None                                 |
| user_id             | 9356914d779d443f8cc09ebdf62ef7c9     |
+---------------------+--------------------------------------+
```

Confirm the volumes were created successfully.
```
sh-5.1$ openstack volume list
+--------------------------------------+-----------+-----------+------+-------------+
| ID                                   | Name      | Status    | Size | Attached to |
+--------------------------------------+-----------+-----------+------+-------------+
| 19a09d9a-1bd8-4c08-97e0-344a9da38caa | vol-ISCSI | available |    1 |             |
| 3260be2f-d547-469e-ab63-785646bdd09e | vol-FC    | available |    1 |             |
+--------------------------------------+-----------+-----------+------+-------------+
```
