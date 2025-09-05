
# Dell Cinder Backend Deployment Guide for Red Hat OpenStack Platform 17

## Overview

This document describes how to deploy the Dell Block Storage services in a Red Hat OpenStack Platform Overcloud.
This assumes that the RHOSP installation is done through RHOSP director toolset which is based primarily on the upstream TripleO project.  



The following Dell storage drivers are fully integrated with director and can be deployed using tripleo heat templates 
* [Unity iSCSI and FC drivers](https://docs.openstack.org/cinder/wallaby/configuration/block-storage/drivers/dell-emc-unity-driver.html) - Please refer to this [custom deployment guide for the Unity Driver](https://github.com/emc-openstack/osp-deploy/tree/rhosp17.1/cinder)
* [PowerFlex drivers](https://docs.openstack.org/cinder/wallaby/configuration/block-storage/drivers/dell-emc-powerflex-driver.html) - Please refer to this [Guide for the PowerFlex driver](https://github.com/dell/osp-integration/tree/master/RHOSP-17.1/cinder/powerflex/README.md)
* [PowerMax drivers](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-powermax-driver.html) - Please refer to this [Guide for the PowerMax driver](https://github.com/dell/osp-integration/blob/master/RHOSP-17.1/cinder/powermax/README.md)
* [PowerStore iSCSI and FC drivers](https://docs.openstack.org/cinder/wallaby/configuration/block-storage/drivers/dell-emc-powerstore-driver.html) - see below

**Note:** The following drivers are no longer certified with RHOSP 17.1 and will be deprecated starting from 2023.1 Antelope release of OpenStack.
* XtremIO iSCSI and FC drivers
* SC Series FC and iSCSI drivers
* VNX iSCSI and FC drivers

## Prerequisites
- Dell Storage Backend configured as storage repository.
- Configuration settings and credentials.

## Deployment Steps

### Prepare the Environment File
The environment file is an OSP director environment file. It contains the settings for each back end that you want to define. This approach will ensure that the back end settings persist through future Overcloud updates and upgrades.  

Create the environment file that will orchestrate the back end settings. Use the sample file provided below for your specific backend.  

**Note:** **LVM driver** is enabled by default in TripleO, you want to set the ```CinderEnableIscsiBackend``` to false in one of your environment file to turn it off.

```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

#### PowerStore iSCSI and FC drivers    
  
For full detailed instruction of options please refer to [PowerStore Backend Configuration](https://docs.openstack.org/cinder/wallaby/configuration/block-storage/drivers/dell-emc-powerstore-driver.html#driver-configuration)

**Environment sample**

With a director deployment, PowerStore backend can be deployed using the integrated heat environment file. This file is located in the following path on the director:
`/usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-powerstore-config.yaml`.

Create the `custom-dellemc-powerstore-config.yaml` file in your `~/templates/` directory.

Edit the file with your favorite editor and fill in the following settings to fit your environment. 

**NOTE**: this method replaces the previous one which consisted of editing the heat environment file. This is no longer considered as a best practice and the preferred approach is to store all overrides into a separate file.

Afterwards, open the copy (`~/templates/cinder-dellemc-powerstore-config.yaml`) and edit it as you see fit. The following shows a sample content of the file. The file will list optional params that the users can choose to override if they don't like the default value.

```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
  CinderEnablePowerStoreBackend: true
  CinderPowerStoreBackendName: 'tripleo_dellemc_powerstore'
  CinderPowerStoreSanIp: 'PowerStore SAN IP'
  CinderPowerStoreSanLogin: 'PowerStore SAN login'
  CinderPowerStoreSanPassword: 'PowerStore SAN password'
  CinderPowerStorePorts: 'PowerStore SAN ports'
  CinderPowerStoreStorageProtocol: 'iSCSI|FC'
```

**NOTE**: All other values will be inherited from `/usr/share/openstack-tripleo-heat-templates/cinder-dellemc-powerstore-config.yaml`, including the resource_registry entry.

From there, you can add the template `~/templates/cinder-dellemc-powerstore-config.yaml` and the environment file `~/templates/custom-dellemc-powerstore-config.yaml` to the openstack overcloud deploy command

    
### Deploy the configured backends

Deploy the backend configuration by running the openstack overcloud deploy command using the appropriate templates. If you passed any extra environment files when you created the overcloud, pass them again here using the -e option.

#### For PowerMax backend
```bash
(undercloud) $ openstack overcloud deploy --templates \
-e /home/stack/templates/overcloud_images.yaml \
-e /usr/share/openstack-tripleo-heat-templates/cinder-dellemc-powermax-config.yaml \
-e <other templates>
.....
-e /home/stack/templates/custom-dellemc-powermax-config.yaml
```
#### For PowerStore backend 
```bash
(undercloud) $ openstack overcloud deploy --templates \
-e /home/stack/templates/overcloud_images.yaml \
-e /usr/share/openstack-tripleo-heat-templates/cinder-dellemc-powerstore-config.yaml \
-e <other templates>
.....
-e /home/stack/templates/custom-dellemc-powerstore-config.yaml
```

### Multiple Backend Deployment
To configure multiple Dell Cinder backends, define an environment file `~/templates/cinder-dellemc-multibackend-config.yaml` as follows:
```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
  CinderEnablePowerStoreBackend: true
  CinderPowerStoreBackendName:
    - tripleo_dellemc_powerstore1
    - tripleo_dellemc_powerstore2
  CinderPowerStoreMultiConfig:
    tripleo_dellemc_powerstore1:
      CinderPowerStoreSanIp: 'PowerStore1 SAN IP'
      CinderPowerStoreSanLogin: 'PowerStore1 SAN login'
      CinderPowerStoreSanPassword: 'PowerStore1 SAN password'
      CinderPowerStorePorts: 'PowerStore1 SAN ports'
      CinderPowerStoreStorageProtocol: 'iSCSI|FC'
    tripleo_dellemc_powerstore2:
      CinderPowerStoreSanIp: 'PowerStore2 SAN IP'
      CinderPowerStoreSanLogin: 'PowerStore2 SAN login'
      CinderPowerStoreSanPassword: 'PowerStore2 SAN password'
      CinderPowerStorePorts: 'PowerStore2 SAN ports'
      CinderPowerStoreStorageProtocol: 'iSCSI|FC'
```

Multiple backends can be configured at a time during deployment. Add the appropriate templates and environment file to the the `overcloud deploy` command above if necessary.

### Verify the configured changes
When the director completes the overcloud deployment, edit the `/var/lib/containers/cinder/etc/cinder/cinder.conf` file with your favorite editor and verify that the powerstore backend has been configured correctly. 
Depending on your environment, it may differ from the output below:
```
[DEFAULT]
.. [output truncated]

enabled_backends = tripleo_dellemc_powerstore, <other backends>

.. [output truncated]

[triple_dellemc_powerstore]
volume_driver = cinder.volume.drivers.dell_emc.powerstore.driver.PowerStoreDriver
volume_backend_name = tripleo_dellemc_powerstore
san_ip = 10.20.30.40
san_login = admin
san_password = password
storage_protocol = iSCSI
powerstore_ports = 10.20.30.41,10.20.30.42
```

Check that the cinder-volume service is up by using the `openstack volume service list` cli command.
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume service list
+---------------+--------------------------------------+------+---------+-------+-----------------------------+
| Binary        | Host                                 | Zone | Status  | State | Updated At                  |
+---------------+--------------------------------------+------+---------+-------+-----------------------------+
... [Truncated]
| cinder-volume | hostgroup@tripleo_dellemc_powerstore | nova | enabled | up    | 2024-01-12T02:36:02.000000  |
... [Truncated]
```
### Testing the configured Backend

After you deploy the back ends to the overcloud, create a volume-type per bac kend and test if you can successfully create and attach volumes of that type.

#### For PowerMax backend
Create a backend volume type for PowerMax would look like:
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type create powermax
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property volume_backend_name=tripleo_dellemc_powermax powermax
```
Create a volume using the type created above without error to ensure the availability of the backend.
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume create --type powermax --size 8 powermax_volume1
```
Confirm the volume was created successfully
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume list
+--------------------------------------+--------------------+-----------+------+-------------+
| ID                                   | Name               | Status    | Size | Attached to |
+--------------------------------------+--------------------+-----------+------+-------------+
| 4ab54252-f4de-4fc4-8721-a45bd89781d1 | powermax_volume1   | available | 8    |             |
+--------------------------------------+--------------------+-----------+------+-------------+
```


#### For PowerStore backend
Create a backend volume type for PowerStore would look like:
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type create powerstore
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property volume_backend_name=tripleo_dellemc_powerstore powerstore
```
Create a volume using the type created above without error to ensure the availability of the backend.
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume create --type powerstore --size 8 powerstore_volume1
```
Confirm the volume was created successfully
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume list
+--------------------------------------+--------------------+-----------+------+-------------+
| ID                                   | Name               | Status    | Size | Attached to |
+--------------------------------------+--------------------+-----------+------+-------------+
| 35808e76-c4cd-4ff6-8829-f16c76ebad37 | powerstore_volume1 | available | 8    |             |
+--------------------------------------+--------------------+-----------+------+-------------+
```
**NOTE**: You can repeat the above operations for each backend that you have configured in case you are using multi-backends setup
## References
* [Red Hat OpenStack Platform Overcloud Custom Block Storage Backend Guide](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.0/html/custom_block_storage_back_end_deployment_guide/index)

