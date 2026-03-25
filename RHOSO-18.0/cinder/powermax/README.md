# Dell PowerMax Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

## Overview

These instructions provide detailed steps on how to enable the Dell PowerMax backend with RHOSO 18.0.

**Note**: This document describes only the minimum steps required to enable Cinder with a Dell PowerMax backend. It does not include configuration steps for other OpenStack or OpenShift components that may be required in your specific environment.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

* A functional Red Hat OpenStack Services on OpenShift 18.0 environment.
* A Dell PowerMax array running a supported PowerMaxOS version.
* Network connectivity between the OpenStack control plane and the PowerMax array.

## Steps

### Configure Dell PowerMax storage details
This section first describes how to configure a PowerMax Fibre Channel (FC) backend. iSCSI configuration will be covered later.
Finally, we'll discover how to configure a multi-backend using both PowerMax FC and iSCSI with an example.

For detailed instructions and supported configuration options, please refer to [Dell PowerMax Backend Configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-powermax-driver.html).

#### PowerMax FC based backend configuration

**Important**: To isolate credentials between backend types, Red Hat recommends creating one secret per backend. This prevents unnecessary reconfiguration or restarts of unrelated cinder-volume services when a secret is updated.
 
In this section, we'll create one secret per backend type but depending on your configuration, you might have to create only one of them.

* Create a Secret CR file named **cinder-volume-powermax-fc-secrets.yaml** (or a name suitable for your environment).
```
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-powermax-fc-secrets
stringData:
  powermax-secrets: |
    [powermaxFC]
    san_ip = <DellPowerMaxIP>
    san_login = <DellPowerMaxAdminUser>
    san_password = <DellPowerMaxAdminPasswd>
type: Opaque
```

* Create the secret.
```
oc create -f cinder-volume-powermax-fc-secrets.yaml
secret/cinder-volume-powermax-fc-secrets created
```

* Verify the secret is created.
```
oc get secrets/cinder-volume-powermax-fc-secrets -o jsonpath={.data.powermax-secrets} | base64 -d
[powermaxFC]
san_ip = <DellPowerMaxIP>
san_login = <DellPowerMaxAdminUser>
san_password = <DellPowerMaxAdminPasswd>
```

#### PowerMax iSCSI based backend configuration
* Create a Secret CR file named **cinder-volume-powermax-iscsi-secrets.yaml** or choose a name that suits your needs.
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-powermax-iscsi-secrets
stringData:
  powermax-secrets: |
    [powermaxISCSI]
    san_ip = <DellPowerMaxIP>
    san_login = <DellPowerMaxAdminUser>
    san_password = <DellPowerMaxAdminPasswd>
type: Opaque
```

* Create the secret.
```
oc create -f cinder-volume-powermax-iscsi-secrets.yaml
secret/cinder-volume-powermax-iscsi-secrets created
```

* Verify the secret is created.
```
oc get secrets/cinder-volume-powermax-iscsi-secrets -o jsonpath={.data.powermax-secrets} | base64 -d
[powermaxISCSI]
san_ip = <DellPowerMaxIP>
san_login = <DellPowerMaxAdminUser>
san_password = <DellPowerMaxAdminPasswd>
```

### Configure Cinder to consume the Dell PowerMax backend
This section describes how to add the PowerMax backend configuration to the OpenStack control plane custom resource (CR).

**NOTICE: As already discussed, you may have to adapt this section depending on your environment, especially if you use FC and iSCSI or only one backend type. In multi-backend configurations, each backend must use a unique volume_backend_name.**

* Edit the control plane CR file (_openstack_control_plane.yaml_) and add the Dell PowerMax configuration under **spec.cinder.cinderVolumes** section.

> PowerMax FC based backend configuration
```yaml
...
      cinderVolumes:
        powermax:
          customServiceConfig: |
            [powermaxFC]
            volume_backend_name = powermax_fc
            volume_driver   = cinder.volume.drivers.dell_emc.powermax.fc.PowerMaxFCDriver
            storage_protocol = FC
            suppress_requests_ssl_warnings = True
            san_api_port = 8443
            powermax_srp = <DellPowerMax_StorageResourcePoolName>
            powermax_array = <DellPowerMax_SerialNumber>
            powermax_port_groups = [<DellPowerMax_PortGroup>]
            powermax_service_level = <DellPowerMax_ServiceLevel>
            image_volume_cache_enabled = True
            driver_ssl_cert_verify = False
          customServiceConfigSecrets:
          - cinder-volume-powermax-fc-secrets
          networkAttachments:
          - storage
          - storageMgmt
          replicas: 1
          resources: {}
...
```

> PowerMax iSCSI based backend configuration
```yaml
...
      cinderVolumes:
        powermax:
          customServiceConfig: |
            [powermaxISCSI]
            volume_backend_name = powermax_iscsi
            volume_driver   = cinder.volume.drivers.dell_emc.powermax.iscsi.PowerMaxISCSIDriver
            storage_protocol = iSCSI
            suppress_requests_ssl_warnings = True
            san_api_port = 8443
            powermax_srp = <DellPowerMax_StorageResourcePoolName>
            powermax_array = <DellPowerMax_SerialNumber>
            powermax_port_groups = [<DellPowerMax_PortGroup>]
            powermax_service_level = <DellPowerMax_ServiceLevel>
            image_volume_cache_enabled = True
          customServiceConfigSecrets:
          - cinder-volume-powermax-iscsi-secrets
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
This section provides an example of configuring two PowerMax backends, using FC and iSCSI. It assumes that you have created one secret per backend type.

**NOTICE: You can add other backends as well but this documentation won't cover it.**

* Edit the control plane CR file (_openstack_control_plane.yaml_) and add the Dell PowerMax configuration under **spec.cinder.cinderVolumes** section.

```yaml
...
      cinderVolumes:
        powermaxFC:
          customServiceConfig: |
            [powermaxFC]
            volume_backend_name = powermax_fc
            volume_driver   = cinder.volume.drivers.dell_emc.powermax.fc.PowerMaxFCDriver
            storage_protocol = FC
            suppress_requests_ssl_warnings = True
            san_api_port = 8443
            powermax_srp = <DellPowerMax_StorageResourcePoolName>
            powermax_array = <DellPowerMax_SerialNumber>
            powermax_port_groups = [<DellPowerMax_PortGroup>]
            powermax_service_level = <DellPowerMax_ServiceLevel>
            image_volume_cache_enabled = True
            driver_ssl_cert_verify = False
          customServiceConfigSecrets:
          - cinder-volume-powermax-fc-secrets
          networkAttachments:
          - storage
          - storageMgmt
          replicas: 1
          resources: {}
       powermaxISCSI:
          customServiceConfig: |
            [powermaxISCSI]
            volume_backend_name = powermax_iscsi
            volume_driver   = cinder.volume.drivers.dell_emc.powermax.iscsi.PowerMaxISCSIDriver
            storage_protocol = iSCSI
            suppress_requests_ssl_warnings = True
            powermax_srp = <DellPowerMax_StorageResourcePoolName>
            powermax_array = <DellPowerMax_SerialNumber>
            powermax_port_groups = [<DellPowerMax_PortGroup>]
            powermax_service_level = <DellPowerMax_ServiceLevel>
            image_volume_cache_enabled = True
            driver_ssl_cert_verify = False
          customServiceConfigSecrets:
          - cinder-volume-powermax-iscsi-secrets
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
In this section, you validate the Cinder configuration by completing the following steps:
* Make sure the cinder-volume service is up and running.
* Create a specific volume type which will be used by Cinder.
* Create a volume on the Dell PowerMax using that volume type.
* Open a session on the **openstackclient** pod.
```
oc rsh openstackclient
```
* Confirm the cinder-volume service for Dell PowerMax as a backend is up and running.
```
sh-5.1$ openstack volume service list
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                         | Zone | Status  | State | Updated At                 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                           | nova | enabled | up    | 2026-03-24T06:12:27.000000 |
| cinder-volume    | cinder-volume-powermax-0@powermax_fc         | nova | enabled | up    | 2026-03-24T06:12:30.000000 |
| cinder-backup    | cinder-backup-0                              | nova | enabled | up    | 2026-03-24T06:12:27.000000 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
```
* Create a volume type named powermax and map it to the Dell PowerMax FC backend.
```
sh-5.1$ openstack volume type create --property volume_backend_name=powermax_fc powermax
```

Confirm the volume type exists and is successfully configured.
```
sh-5.1$ openstack volume type show powermax
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| access_project_ids | None                                 |
| description        | None                                 |
| id                 | f6aaab51-276d-4643-b02c-199df75a0aab |
| is_public          | True                                 |
| name               | powermax                             |
| properties         | volume_backend_name='powermax_fc'    |
| qos_specs_id       | None                                 |
+--------------------+--------------------------------------+
```

* Create a volume using the type created above without error to verify the availability of the backend.

```
sh-5.1$ openstack volume create --type powermax --size 1 vol
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2026-03-24T08:04:34.933724           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 4bb87397-6eff-406b-8e60-cd003366b0b1 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | vol                                  |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | powermax                             |
| updated_at          | None                                 |
| user_id             | 9356914d779d443f8cc09ebdf62ef7c9     |
+---------------------+--------------------------------------+
```
Confirm the volume was created successfully.

sh-5.1$ openstack volume list
+--------------------------------------+-----------+-----------+------+-------------+
| ID                                   | Name      | Status    | Size | Attached to |
+--------------------------------------+-----------+-----------+------+-------------+
| 4bb87397-6eff-406b-8e60-cd003366b0b1 | vol       | available |    1 |             |
+--------------------------------------+-----------+-----------+------+-------------+
sh-5.1$


#### Multi-backend validation
To conclude our multi-backend configuration example, we'll execute the same steps against two separate PowerMax backend (_powermax_fc_ and _powermax_iscsi_).

* Open a session on the **openstackclient** pod.
```
oc rsh openstackclient
```

* Confirm the cinder-volume service for all Dell PowerMax backends are up and running
```
sh-5.1$ openstack volume service list
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                         | Zone | Status  | State | Updated At                 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                           | nova | enabled | up    | 2026-01-16T15:34:39.000000 |
| cinder-volume    | cinder-volume-powermax-0@powermax_fc         | nova | enabled | up    | 2026-01-16T15:34:37.000000 |
| cinder-volume    | cinder-volume-powermax-0@powermax_iscsi      | nova | enabled | up    | 2026-01-16T15:34:37.000000 |
| cinder-backup    | cinder-backup-0                              | nova | enabled | up    | 2026-01-16T15:34:39.000000 |
+------------------+----------------------------------------------+------+---------+-------+----------------------------+
```

* Create a volume type named powermaxFC and powermaxISCSI and map it to the respective backends.
```
sh-5.1$ openstack volume type create --property volume_backend_name=powermax_fc powermaxFC

sh-5.1$ openstack volume type create --property volume_backend_name=powermax_iscsi powermaxISCSI
```

Confirm both volume types exist and are successfully configured.
```
sh-5.1$ openstack volume type show powermaxFC
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| access_project_ids | None                                 |
| description        | None                                 |
| id                 | 90ff51a1-7c7c-4167-ba41-5374d5ba43a7 |
| is_public          | True                                 |
| name               | powermaxFC                           |
| properties         | volume_backend_name='powermax_fc'    |
| qos_specs_id       | None                                 |
+--------------------+--------------------------------------+
```

```
sh-5.1$ openstack volume type show powermaxISCSI
+--------------------+---------------------------------------+
| Field              | Value                                 |
+--------------------+---------------------------------------+
| access_project_ids | None                                  |
| description        | None                                  |
| id                 | c82f4eb7-4056-49e6-9e6b-287d80cadec2  |
| is_public          | True                                  |
| name               | powermaxISCSI                         |
| properties         | volume_backend_name='powermax_iscsi'  |
| qos_specs_id       | None                                  |
+--------------------+---------------------------------------+
```

Create a volume using the volume-types created above and ensure there is no error to confirm the availability of the backend.
```
sh-5.1$ openstack volume create --type powermaxFC --size 1 vol-FC
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2026-03-24T06:43:51.479643           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 312e7502-e1be-49c8-845e-c0d9d3d0853d |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | vol-FC                               |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | powermaxFC                           |
| updated_at          | None                                 |
| user_id             | 9356914d779d443f8cc09ebdf62ef7c9     |
+---------------------+--------------------------------------+
```

```
sh-5.1$ openstack volume create --type powermaxISCSI --size 1 vol-ISCSI
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2026-03-24T06:53:00.813092           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 69f1e4b8-41ea-4452-b281-08f6187ee059 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | vol-ISCSI                            |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | powermaxISCSI                        |
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
| 69f1e4b8-41ea-4452-b281-08f6187ee059 | vol-ISCSI | available |    1 |             |
| 312e7502-e1be-49c8-845e-c0d9d3d0853d | vol-FC    | available |    1 |             |
+--------------------------------------+-----------+-----------+------+-------------+
```