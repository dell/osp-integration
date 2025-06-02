# Dell PowerMax Backend Deployment Guide for Red Hat OpenStack Platform 17.1

## Overview

This instruction provide detailed steps on how to enable PowerMax storage in Red Hat OpenStack Platform Overcloud. This assumes that the RHOSP installion is through RHOSP director toolset which is based primarily on the upstream TripleO project.
[PowerMax iSCSI and FC drivers](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-powermax-driver.html).

**NOTICE**: This README represents only the **basic** steps necessary to enable PowerMax driver. It does not contain steps on how update the overcloud or other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Platform 17.1](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/).

## Prerequisites

- Red Hat OpenStack Platform 17.1 with RHEL 9.2.
- Dell Powermax Storage Backend configured as storage repository.
- Configuration settings and credentials

## Deployment Steps

### Prepare the Environment File for PowerMax cinder backend
The environment file is a OSP director environment file. The environment file contains the settings for each back end you want to define. Using the environment file will ensure that the back end settings persist through future Overcloud updates and upgrades.  

Create the environment file that will orchestrate the back end settings. Use the sample file provided below the backend reference.  

**NOTICE**: **LVM driver** is enabled by default in TripleO, set the ```CinderEnableIscsiBackend``` to false in one of your environment file to turn it off.
```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

For full detailed instruction of options please refer to [PowerMax Backend Configuration](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/dell-emc-powermax-driver.html#configuration-options).

**Environment sample**
With a director deployment, PowerMax backend can be deployed using the integrated heat environment file. This file is located in the following path of the Undercloud node:
`/usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-powermax-config.yaml`.

Create the `custom-dellemc-powermax-config.yaml` file in your `~/templates/` directory. Open the file and edit with your favorite editor to fit your environment. The following shows sample content of the file. The files will list optional parameters that the user can choose to override if they don't need the default value.

```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
  CinderEnablePowermaxBackend: true
  CinderPowermaxBackendName: 'tripleo_dellemc_powermax'
  CinderPowermaxMultiConfig: {}
  CinderPowermaxAvailabilityZone: ''
  CinderPowermaxSanIp: 'PowerMax SAN IP'
  CinderPowermaxSanLogin: 'PowerMax SAN login'
  CinderPowermaxSanPassword: 'PowerMax SAN password'
  CinderPowermaxArray: 'Array name'
  CinderPowermaxSrp: ''
  CinderPowermaxPortGroups: 'Port group name'
  CinderPowermaxStorageProtocol: 'iSCSI/FC'
```

**NOTE**: All other values will be inherited from `/usr/share/openstack-tripleo-heat-templates/cinder-dellemc-powerflex-config.yaml`, including the resource_registry entry.

### Deploy the configured backend

Deploy the backend configuration by running the openstack overcloud deploy command using the appropriate templates. If you passed any extra environment files when you created the overcloud created the overcloud, pass them again here using the -e option.

```bash
(undercloud) $ openstack overcloud deploy --templates \
-e /home/stack/templates/overcloud_images.yaml \
-e /usr/share/openstack-tripleo-heat-templates/cinder-dellemc-powermax-config.yaml
-e <other templates>
.....
-e /home/stack/templates/custom-dellemc-powermax-config.yaml
```

## Multiple Backend Deployment
To configure multiple Dell Cinder backends, define an environment file ~/templates/cinder-dellemc-multibackend-config.yaml as follows:

```yaml
parameter_defaults:
  CinderEnablePowermaxBackend: true
  CinderPowermaxBackendName:
   - tripleo_dellemc_powermax_iscsi
   - tripleo_dellemc_powermax_fc
  CinderPowermaxMultiConfig:
   tripleo_dellemc_powermax_iscsi:
     CinderPowermaxStorageProtocol: 'iSCSI'
     CinderPowermaxPortGroups: '[openstack17-ISCSI_PG]'
     CinderPowermaxSanIp: 'powermax_iscsi_SAN_IP'
     CinderPowermaxSanLogin: 'powermax_iscsi_SAN_username'
     CinderPowermaxSanPassword: 'powermax_iscsi_SAN_password'
     CinderPowermaxArray: 'Array name'
     CinderPowermaxSrp: 'SRP_1'
   tripleo_dellemc_powermax_fc:
     CinderPowermaxStorageProtocol: 'FC'
     CinderPowermaxPortGroups: '[openstack17_FC_PG]'
     CinderPowermaxSanIp: 'powermax_fc_SAN_IP'
     CinderPowermaxSanLogin: 'powermax_fc_SAN_username'
     CinderPowermaxSanPassword: 'powermax_fc_SAN_password'
     CinderPowermaxArray: 'Array name'
     CinderPowermaxSrp: 'SRP_1'
```
Multiple backends can be configured at a time during deployment. Add the appropriate templates and environment file to the the overcloud deploy command above if necessary.

## Post deployment tasks

### Verify configured backend

After the deployment finishes successfully, open the /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf file with your favorite editor and verify that the powermax backend has been configured correctly. Depending on your environment, it may differ from the output below:

```ini
[DEFAULT]
.. [output truncated]
enabled_backends = tripleo_dellemc_powermax_iscsi,tripleo_dellemc_powermax_fc
.. [output truncated]

[tripleo_dellemc_powermax_iscsi]
backend_host=hostgroup
volume_backend_name=tripleo_dellemc_powermax_iscsi
volume_driver=cinder.volume.drivers.dell_emc.powermax.iscsi.PowerMaxISCSIDriver
san_ip=10.10.10.226
san_login=admin
san_password=P@$$w0rd
powermax_array=array123
powermax_srp=SRP_1
powermax_port_groups=[openstack17-ISCSI_PG]
[tripleo_dellemc_powermax_fc]
backend_host=hostgroup
volume_backend_name=tripleo_dellemc_powermax_fc
volume_driver=cinder.volume.drivers.dell_emc.powermax.fc.PowerMaxFCDriver
san_ip=10.10.10.226
san_login=admin
san_password=P@$$w0rd
powermax_array=array123
powermax_srp=SRP_1
powermax_port_groups=[openstack17_FC_PG]
...
```

Run the following command to check whether the cinder-volume service is up.
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume service list

+------------------+-------------------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                                      | Zone | Status  | State | Updated At                 |
+------------------+-------------------------------------------+------+---------+-------+----------------------------+
... [Truncated]                                
| cinder-volume    | hostgroup@tripleo_dellemc_powermax_iscsi  | nova | enabled | up    | 2025-05-20T17:38:08.000000 |
| cinder-volume    | hostgroup@tripleo_dellemc_powermax_fc     | nova | enabled | up    | 2025-05-20T17:43:50.000000 |
| cinder-backup    | osp166ctl0                                | nova | enabled | up    | 2025-05-20T17:43:48.000000 |
... [Truncated]                               
```

### Testing the Configured Backend

Finally, create a PowerMax volume type and test if you can successfully create and attach volumes of that type.

**For PowerMax iSCSI backend**
Create a volume type mapped to the deployed backend.
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type create PowerMax_iSCSI
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property volume_backend_name=tripleo_dellemc_powermax_iscsi PowerMax_iSCSI
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property pool_name=Diamond+SRP_1+<array_name>
```
Create a volume using the type created above without error to ensure the availability of the backend.
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume create --type PowerMax_iSCSI --size 1 powermax_iscsi_volume1
```
Confirm the volume was created successfully
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume list
+--------------------------------------+------------------------+-----------+------+-------------+
| ID                                   | Name                   | Status    | Size | Attached to |
+--------------------------------------+------------------------+-----------+------+-------------+
| 251d8bc8-4a67-4370-9356-929b984ea4dd | powermax_iscsi_volume1 | available |    1 |             |
+--------------------------------------+------------------------+-----------+------+-------------+
```

**For PowerMax FC backend**
Create a volume type mapped to the deployed backend.
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type create PowerMax_FC
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property volume_backend_name=tripleo_dellemc_powermax_fc PowerMax_FC
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property pool_name=Diamond+SRP_1+<array_name>
```
Create a volume using the type created above without error to ensure the availability of the backend.
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume create --type PowerMax_FC --size 1 powermax_fc_volume1
```
Confirm the volume was created successfully
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume list
+--------------------------------------+------------------------+-----------+------+-------------+
| ID                                   | Name                   | Status    | Size | Attached to |
+--------------------------------------+------------------------+-----------+------+-------------+
| 251d8bc8-4a67-4370-9356-929b984ea4dd | powermax_fc_volume1    | available |    1 |             |
+--------------------------------------+------------------------+-----------+------+-------------+
```

**NOTE**: You can repeat the above operations for each backend that you have configured in case you are using multi-backends setup

**References**
[Red Hat OpenStack Platform Overcloud Deploying Custom Block Storage Backend](https://docs.redhat.com/en/documentation/red_hat_openstack_platform/17.1/html/deploying_a_custom_block_storage_back_end/index).
