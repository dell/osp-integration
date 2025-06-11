# Dell Unity Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

## Overview

These instruction provide detailed steps on how to enable Dell Unity backend with RHOSO 18.

**NOTICE**: this README represents only the **basic** required steps to enable Cinder to use a Dell Unity array as a backend. It does not contain steps of other components in the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- Red Hat OpenStack Services on OpenShift 18.0.
- Dell Unity storage platform running supported Dell Unity OE.

## Steps

### Configure Dell Unity storage details
This section is broken down into two sub-sections. In the first, we'll cover how to configure a Unity FC based backend while we'll talk about Unity iSCSI based backend in the latter.
Finally, we'll discover how to configure a multi-backend using Unity FC and Unity iSCSI as an example.

#### Unity FC based backend configuration

**NOTICE: In order to isolate FC and iSCSI secrets data, it's recommended to dedicate one secret per backend type instead of using one common secret and get a reconfiguration for all backends sharing the same secret as a side-effect.**
 
In this section, we'll create one secret per backend type but depending on your configuration, you might have to create only one of them.

For detailed instructions and supported configuration options, please refer to [Dell Unity Backend Configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-unity-driver.html).

* Create a Secret CR file named **cinder-volume-unity-fc-secrets.yaml** or choose a name that suits your needs.
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-unity-fc-secrets
stringData:
  unity-secrets: |
    [unityFC]
    san_ip = <DellUnityIP>
    san_login = <DellUnityAdminUser>
    san_password = <DellUnityAdminPasswd>
type: Opaque
```

* Create the secret.
```
oc create -f cinder-volume-unity-fc-secrets.yaml
secret/cinder-volume-unity-fc-secrets created
```

* Verify the secret is created.
```
oc get secrets/cinder-volume-unity-fc-secrets -o jsonpath={.data.unity-secrets} | base64 -d
[unityFC]
san_ip = <DellUnityIP>
san_login = <DellUnityAdminUser>
san_password = <DellUnityAdminPasswd>
```

#### Unity iSCSI based backend configuration
* Create a Secret CR file named **cinder-volume-unity-iscsi-secrets.yaml** or choose a name that suits your needs.
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-unity-iscsi-secrets
stringData:
  unity-secrets: |
    [unityISCSI]
    san_ip = <DellUnityIP>
    san_login = <DellUnityAdminUser>
    san_password = <DellUnityAdminPasswd>
type: Opaque
```

* Create the secret.
```
oc create -f cinder-volume-unity-iscsi-secrets.yaml
secret/cinder-volume-unity-iscsi-secrets created
```

* Verify the secret is created.
```
oc get secrets/cinder-volume-unity-iscsi-secrets -o jsonpath={.data.unity-secrets} | base64 -d
[unityISCSI]
san_ip = <DellUnityIP>
san_login = <DellUnityAdminUser>
san_password = <DellUnityAdminPasswd>
```

### Configure Cinder to consume the Dell Unity backend
In this section, we'll add Dell Unity configuration to the OpenStack control plane. 

**NOTICE: As already discussed, you may have to adapt this section depending on your environment, especially if you use FC and iSCSI or only one backend type.**

* Edit the control plane CR file (_openstack_control_plane.yaml_) and add the Dell Unity configuration under **spec.cinder.cinderVolumes** section.

> Unity FC based backend configuration
```yaml
...
      cinderVolumes:
        unity:
          customServiceConfig: |
            [unityFC]
            volume_backend_name = unity
            volume_driver   = cinder.volume.drivers.dell_emc.unity.Driver
            storage_protocol = FC
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-unity-fc-secrets
          networkAttachments:
          - storage
          - storageMgmt
          replicas: 1
          resources: {}
...
```

> Unity iSCSI based backend configuration
```yaml
...
      cinderVolumes:
        unity:
          customServiceConfig: |
            [unityISCSI]
            volume_backend_name = unity
            volume_driver   = cinder.volume.drivers.dell_emc.unity.Driver
            storage_protocol = iSCSI
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-unity-iscsi-secrets
          networkAttachments:
          - storage
          - storageMhmt
          replicas: 1
          resources: {}
...
```

* Apply your changes to the control-plane.
```
oc apply -f openstack_control_plane.yaml
```
### Alter the container image used by cinder-volume
**NOTICE: Because Dell Unity has a library dependency named storops, this section describes how to specify a custom container image when running cinder-volume.**

* Edit the OpenStack versions CR file (_openstack_versions.yaml_) and specify the container image name in the **spec.customContainerImages.cinderVolumeImages** section.
```yaml
...
spec:
  customContainerImages:
    cinderVolumeImages:
      unity: registry.connect.redhat.com/dell-emc/openstack-cinder-volume-dellemc-unity:18.0.0
 ...
```

* Apply your changes to the control-plane.
  ```
  oc apply -f openstack_versions.yaml
  ```

* Confirm the pod is running and uses the custom container image provided in the previous section.
```
oc get pods/cinder-volume-unity-0 -o jsonpath='{.spec.containers[0].image}{"\n"}'
registry.connect.redhat.com/dell-emc/openstack-cinder-volume-dellemc-unity:18.0
```

### Configure multi-backend 
This section provides an example of configuring two unity backend, using FC and iSCSI. It assumes that you have created one secret per backend type like depicted in [Configure Dell Unity storage details](https://github.com/dell/osp-integration/tree/master/RHOSO-18.0/cinder/unity#configure-dell-unity-storage-details)

**NOTICE: You can add other backend as well but this documentation won't cover it.**

* Edit the control plane CR file (_openstack_control_plane.yaml_) and add the Dell Unity configuration under **spec.cinder.cinderVolumes** section.

```yaml
...
      cinderVolumes:
        unityiSCSI:
          customServiceConfig: |
            [unityISCSI]
            volume_backend_name = unityISCSI
            volume_driver   = cinder.volume.drivers.dell_emc.unity.Driver
            storage_protocol = iSCSI
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-unity-iscsi-secrets
          networkAttachments:
          - storage
          - storageMhmt
          replicas: 1
          resources: {}
        unityFC:
          customServiceConfig: |
            [unityFC]
            volume_backend_name = unityFC
            volume_driver   = cinder.volume.drivers.dell_emc.unity.Driver
            storage_protocol = FC
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-unity-fc-secrets
          networkAttachments:
          - storage
          - storageMhmt
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
* Create a volume on the Dell Unity using that volume type.

* Open a session on the **openstackclient** pod.
```
oc rsh openstackclient
```

* Confirm the cinder-volume using Dell Unity as a backend is up and running
```
[admin@rhadmin ~]$ oc rsh openstackclient
sh-5.1$ openstack volume service list
+------------------+-------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                | Zone | Status  | State | Updated At                 |
+------------------+-------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                  | nova | enabled | up    | 2025-06-04T12:30:32.000000 |
| cinder-volume    | cinder-volume-unity-0@unity         | nova | enabled | up    | 2025-05-21T10:17:11.000000 |
| cinder-backup    | cinder-backup-0                     | nova | enabled | up    | 2025-06-04T12:30:34.000000 |
+------------------+-------------------------------------+------+---------+-------+----------------------------+
```

* Create a volume type named unity and map it to the Dell Unity backend.
```
sh-5.1$ openstack volume type create --property volume_backend_name=unity unity
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| description | None                                 |
| id          | 5dfa8b77-3445-4a33-b14f-186fe1a00c51 |
| is_public   | True                                 |
| name        | unity                                |
| properties  | volume_backend_name='unity'          |
+-------------+--------------------------------------+
```

* Confirm the volume type exists and is successfully configured.
```
sh-5.1$ openstack volume type show unity
+--------------------+------------------------------------------------------------+
| Field              | Value                                                      |
+--------------------+------------------------------------------------------------+
| access_project_ids | None                                                       |
| description        |                                                            |
| id                 | 5dfa8b77-3445-4a33-b14f-186fe1a00c51                       |
| is_public          | True                                                       |
| name               | unity                                                      |
| properties         | volume_backend_name='unity'                                |
| qos_specs_id       | None                                                       |
+--------------------+------------------------------------------------------------+
```

Create a volume using the type created above without error to ensure the availability of the backend.
```
sh-5.1$ openstack volume create --type unity --size 1 volunity
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2025-06-04T12:37:32.981447           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | f026a707-544c-4c97-ac65-e340f816894e |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | volunity                             |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | unity                                |
| updated_at          | None                                 |
| user_id             | 1ce9b1415f13480bb951bc470062d5d0     |
+---------------------+--------------------------------------+
```

Confirm the volume was created successfully
```
sh-5.1$ openstack volume list
+--------------------------------------+----------+-----------+------+-------------+
| ID                                   | Name     | Status    | Size | Attached to |
+--------------------------------------+----------+-----------+------+-------------+
| f026a707-544c-4c97-ac65-e340f816894e | volunity | available |    1 |             |
+--------------------------------------+----------+-----------+------+-------------+
```

#### Multi-backend validation
To conclude our multi-backend configuration example, we'll execute the same steps against two separate Unity backend (_unityFC_ and _unityISCSI_.

* Open a session on the **openstackclient** pod.
```
oc rsh openstackclient
```

* Confirm the cinder-volume for all Dell Unity backend are up and running
```
[admin@rhadmin ~]$ oc rsh openstackclient
sh-5.1$ openstack volume service list
+------------------+-------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                | Zone | Status  | State | Updated At                 |
+------------------+-------------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | cinder-scheduler-0                  | nova | enabled | up    | 2025-06-04T12:30:32.000000 |
| cinder-volume    | cinder-volume-unity-0@unityFC       | nova | enabled | up    | 2025-05-21T10:17:11.000000 |
| cinder-volume    | cinder-volume-unity-0@unityISCSI    | nova | enabled | up    | 2025-05-21T10:17:11.000000 |
| cinder-backup    | cinder-backup-0                     | nova | enabled | up    | 2025-06-04T12:30:34.000000 |
+------------------+-------------------------------------+------+---------+-------+----------------------------+
```

* Create a volume type named unityFC and map it to the Dell Unity FC backend.
```
sh-5.1$ openstack volume type create --property volume_backend_name=unityFC unityFC
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| description | None                                 |
| id          | 5dfa8c98-3556-4a33-b14f-29bde4534521 |
| is_public   | True                                 |
| name        | unityFC                              |
| properties  | volume_backend_name='unityFC'        |
+-------------+--------------------------------------+
```

* Create a volume type named unityISCSI and map it to the Dell Unity iSCSI backend.
```
sh-5.1$ openstack volume type create --property volume_backend_name=unityISCSI unityISCSI
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| description | None                                 |
| id          | 5dfa8b77-3445-4a33-b14f-186fe1a00c51 |
| is_public   | True                                 |
| name        | unityISCSI                           |
| properties  | volume_backend_name='unityISCSI'     |
+-------------+--------------------------------------+
```

* Confirm both volume types exist and are successfully configured.
```
sh-5.1$ openstack volume type show unityFC
+--------------------+------------------------------------------------------------+
| Field              | Value                                                      |
+--------------------+------------------------------------------------------------+
| access_project_ids | None                                                       |
| description        |                                                            |
| id                 | 5dfa8c98-3556-4a33-b14f-29bde4534521                       |
| is_public          | True                                                       |
| name               | unityFC                                                    |
| properties         | volume_backend_name='unityFC'                              |
| qos_specs_id       | None                                                       |
+--------------------+------------------------------------------------------------+
```

```
sh-5.1$ openstack volume type show unityFC
+--------------------+------------------------------------------------------------+
| Field              | Value                                                      |
+--------------------+------------------------------------------------------------+
| access_project_ids | None                                                       |
| description        |                                                            |
| id                 | 5dfa8b77-3445-4a33-b14f-186fe1a00c51                       |
| is_public          | True                                                       |
| name               | unityISCSI                                                 |
| properties         | volume_backend_name='unityISCSI                            |
| qos_specs_id       | None                                                       |
+--------------------+------------------------------------------------------------+
```

Create a volume using the types created above without error to ensure the availability of all backends.
```
sh-5.1$ openstack volume create --type unityFC --size 1 volunityFC
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2025-06-11T14:05:32.981447           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | f026a707-544c-4c97-ac65-e340f816894e |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | volunityFC                           |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | unityFC                              |
| updated_at          | None                                 |
| user_id             | 1ce9b1415f13480bb951bc470062d5d0     |
+---------------------+--------------------------------------+
```

```
sh-5.1$ openstack volume create --type unityISCSI --size 1 volunityISCSI
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2025-06-11T14:07:32.052124           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | e137b808-633d-4a85-bd69-f450f927905e |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | volunityISCSI                        |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | unityISCSI                           |
| updated_at          | None                                 |
| user_id             | 1ce9b1415f13480bb951bc470062d5d0     |
+---------------------+--------------------------------------+
```

Confirm the volumes were created successfully
```
sh-5.1$ openstack volume list
+--------------------------------------+---------------+-----------+------+-------------+
| ID                                   | Name          | Status    | Size | Attached to |
+--------------------------------------+---------------+-----------+------+-------------+
| f026a707-544c-4c97-ac65-e340f816894e | volunityFC    | available |    1 |             |
| e137b808-633d-4a85-bd69-f450f927905e | volunityISCSI | available |    1 |             |
+--------------------------------------+---------------+-----------+------+-------------+
```

