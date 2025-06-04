# Dell Unity Backend Deployment Guide for Red Hat OpenStack Services on OpenShift 18.0

## Overview

These instruction provide detailed steps on how to enable Dell Unity backend with RHOSO 18.

**NOTICE**: this README represents only the **basic** required steps to enable Cinder to use a Dell Unity array as a backend. It does not contain steps of other components in the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Services on OpenShift 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- Red Hat OpenStack Services on OpenShift 18.0 with RHEL 9.4.
- Red Hat OpenShift 4.16
- Dell Unity storage platform running supported Dell Unity OE.

## Steps

### Configure Dell Unity storage details
In this section, we'll create a secret which hold sensitive data for the Dell Unity backend that we'll configure in the next step.

* Create a Secret CR file named **cinder-volume-unity-secrets.yaml** or choose a name that suits your needs.
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-unity-secrets
stringData:
  unity-secrets: |
    [unity]
    san_ip = <DellUnityIP>
    san_login = <DellUnityAdminUser>
    san_password = <DellUnityAdminPasswd>
type: Opaque
```

* Create the secret.
```
oc create -f cinder-volume-unity-secrets.yaml
secret/cinder-volume-unity-secrets created
```

* Verify the secret is created.
```
oc get secrets/cinder-volume-unity-secrets -o jsonpath={.data.unity-secrets} | base64 -d
[unity]
san_ip = <DellUnityIP>
san_login = <DellUnityAdminUser>
san_password = <DellUnityAdminPasswd>
```
For detailed instructions and supported configuration options, please refer to [Dell Unity Backend Configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-unity-driver.html).

### Configure Cinder to consume the Dell Unity backend
In this section, we'll add Dell Unity configuration to the OpenStack control plane.

* Edit the control plane and add the Dell Unity configuration under **spec.cinder.cinderVolumes** section.

**NOTICE: Depending on your environment, you may have to choose between FC or iSCSI as the storage_protocol**

```yaml
oc edit openstackcontrolplanes.core.openstack.org openstack-galera-network-isolation
...
      cinderVolumes:
        unity:
          customServiceConfig: |
            [unity]
            volume_backend_name = unity
            volume_driver   = cinder.volume.drivers.dell_emc.unity.Driver
            storage_protocol = iSCSI
            image_volume_cache_enabled  = true
            use_multipath_for_image_xfer = true
            suppress_requests_ssl_warnings = True
          customServiceConfigSecrets:
          - cinder-volume-unity-secrets
          networkAttachments:
          - storage
          replicas: 1
          resources: {}
...
```
### Alter the container image used by cinder-volume
**NOTICE: Because Dell Unity has a library dependency named storops, this section describes how to specify a custom container image when running cinder-volume.**

* Edit the current control plane configuration and specify the container image name in the **spec.customContainerImages.cinderVolumeImages** section.
```yaml
oc edit openstackversions.core.openstack.org openstack-galera-network-isolation
...
spec:
  customContainerImages:
    cinderVolumeImages:
      unity: quay.io/jproque-dell/openstack-cinder-volume-dellemc-unity:latest
  targetVersion: 18.0.6-20250403.1
...
```

* Confirm the pod is running and uses the custom container image provided in the previous section.
```
oc get pods/cinder-volume-unity-0 -o jsonpath='{.spec.containers[0].image}{"\n"}'
quay.io/jproque-dell/openstack-cinder-volume-dellemc-unity:latest
```

### Test the configured Backend
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
