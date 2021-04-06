# Dell EMC PowerFlex Backend Deployment Guide for Red Hat OpenStack Platform 16.1

Deployment tools for Dell EMC PowerFlex (formerly VxFlex OS/ScaleIO) support in RedHat OpenStack Platform 16.1.

## Overview

This instruction provide detailed steps on how to enable PowerFlex.

**NOTICE**: this README represents only the **basic** steps necessary to enable PowerFlex driver. It does not contain steps on how update the overcloud or other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Platform 16.1](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.1/).

## Prerequisites

- Red Hat OpenStack Platform 16.1 with RHEL 8.2.
- PowerFlex 3.5
- PowerFlex Gateway has to be installed and accessible in the network.
- PowerFlex Storage Data Client (SDC) has to be installed on all OpenStack nodes.
- Configuration settings and credentials of Gateway.

## Steps

### Prepare the Environment File for PowerFlex cinder backend
The environment file is a OSP director environment file. The environment file contains the settings for each back end you want to define. Using the environment file will ensure that the back end settings persist through future Overcloud updates and upgrades.  

Create the environment file that will orchestrate the back end settings. Use the sample file provided below for your specific backend.  

**NOTICE**: **LVM driver** is enabled by default in TripleO, set the ```CinderEnableIscsiBackend``` to false in one of your environment files to turn it off.
```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

For full detailed instruction of all options please refer to [PowerFlex Backend Configuration](https://docs.openstack.org/cinder/victoria/configuration/block-storage/drivers/dell-emc-powerflex-driver.html).

**Environment sample**

With a director deployment, PowerFlex backend can be deployed using the integrated heat environment file. This file is located in the following path of the Undercloud node:
`/usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-powerflex-config.yaml`.

Copy this file to a local path where you can edit and invoke it later. For example, to copy it to `~/templates/`:

```bash
$ cp /usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-powerflex-config.yaml ~/templates/
```
Afterwards, open the copy (`~/templates/cinder-dellemc-powerflex-config.yaml`) and edit it as you see fit. The following shows a sample content of the file. The files will list optional params that the user can choose to override if they don't like the default value.

Note that the ```resource_registry``` entry in the heat environment file must be an absolute path when you make a copy.

```yaml
# A Heat environment file which can be used to enable a
# a Cinder Dell EMC PowerFlex backend, configured via puppet
resource_registry:
  OS::TripleO::Services::CinderBackendPowerFlex: ../deployment/cinder/cinder-backend-dellemc-powerflex-puppet.yaml

parameter_defaults:
  CinderEnablePowerFlexBackend: true
  CinderPowerFlexBackendName: 'tripleo_dellemc_powerflex'
  CinderPowerFlexSanIp: ''
  CinderPowerFlexSanLogin: ''
  CinderPowerFlexSanPassword: ''
  CinderPowerFlexStoragePools: 'domain1:pool1'
  CinderPowerFlexAllowMigrationDuringRebuild: false
  CinderPowerFlexAllowNonPaddedVolumes: false
  CinderPowerFlexMaxOverSubscriptionRatio: 7.0
  CinderPowerFlexRestServerPort: 443
  CinderPowerFlexServerApiVersion: ''
  CinderPowerFlexUnmapVolumeBeforeDeletion: false
  CinderPowerFlexSanThinProvision: true
  CinderPowerFlexDriverSSLCertVerify: false
  CinderPowerFlexDriverSSLCertPath: ''
```

### Prepare custom volume mappings for connector configuration 

On each node where PowerFlex SDC will be installed, create the directory where the connector file will reside.

```bash
$ mkdir -p /opt/emc/scaleio/openstack
```

Create or edit `/home/stack/templates/custom-dellemc-volume-mappings.yaml`.

```yaml
parameter_defaults:
  NovaComputeOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  CinderVolumeOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  GlanceApiOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
```
### Workarounds

Workaround for SElinux issue, see attached [SElinux_workaround_for_SDC_RHEL8.2.pdf](https://github.com/dell/osp-integration/edit/master/osp-deploy/cinder/powerflex/SElinux_workaround_for_SDC_RHEL8.2.pdf).


### Deploy the configured backends

When you have created the file `cinder-dellemc-powerflex-config.yaml` file with appropriate backends, deploy the backend configuration by running the openstack overcloud deploy command using the templates option. If you passed any extra environment files when you created the overcloud, pass them again here using the -e option. 
 
```bash
(undercloud) $ openstack overcloud deploy --templates \
-e /home/stack/templates/overcloud_images.yaml \
-e <other templates>
.....
-e /home/stack/templates/cinder-dellemc-powerflex-config.yaml  \
-e /home/stack/templates/custom-dellemc-volume-mappings.yaml \
```

### Verify configured changes

After the deployment finishes successfully, `/etc/cinder/cinder.conf` in the Cinder container should reflect changes made above.

```ini
[DEFAULT]
enabled_backends = tripleo_dellemc_powerflex

[tripleo_dellemc_powerflex]
volume_driver = cinder.volume.drivers.dell_emc.powerflex.driver.PowerFlexDriver
volume_backend_name = tripleo_dellemc_powerflex
san_ip = GATEWAY_IP
powerflex_storage_pools = Domain1:Pool1,Domain2:Pool2
san_login = <login>
san_password = <password>
san_thin_provision = false
...
```

### Configure connector

Before using attach/detach volume operations PowerFlex connector must be properly configured. On each node where PowerFlex SDC is installed do the following:

Create `/opt/emc/scaleio/openstack/connector.conf` if it does not exist.

```bash
$ touch /opt/emc/scaleio/openstack/connector.conf
```

For each PowerFlex section in the cinder.conf create the same section in the `/opt/emc/scaleio/openstack/connector.conf` and populate it with passwords. 

Example:

```
[tripleo_dellemc_powerflex]
san_password = powerflex_password

[tripleo_dellemc_powerflex-new]
san_password = powerflex_password
```

### SDC installation

Install the Storage Data Client (SDC) on all nodes after deploying the overcloud

### Testing the configured Backend
Finally create a volume-type and test if you can successfully create and attach volumes of that type.

## Upgrading from RHOSP13 to RHOSP16.1

### Upgrading PowerFlex

First of all, upgrade PowerFlex system using official instructions. Upgrading SDC is not necessary as it has to be upgraded after Overcloud nodes OS upgrade.
For more information on how to upgrade PowerFlex system please contact [PowerFlex support team](https://www.dell.com/support/contents/en-us/category/contact-information).

### Upgrading RHOSP

For instructions of how to upgrade RHOSP13 to RHOSP16.1 please refer to [In-place upgrades from Red Hat OpenStack Platform 13 to 16.1](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.1/html/framework_for_upgrades_13_to_16.1).

### Additional steps that have to be done to get working environment after upgrade

If you use the environment file with configuration from previous installation (RHOSP13) some changes have to be done:

* No custom containers for cinder/nova/glance services are needed in RHOSP16.1. They must be removed from environment file.
  However, custom volume mappings are still necessary.

* **cinder.volume.drivers.dell_emc.scaleio.driver.ScaleIODriver** was deprecated and removed. sio_* options have to be renamed.
  Do not change **volume_backend_name** as it is used in Cinder Volume Type object.

This file has to be included in openstack `overcloud upgrade prepare` and `openstack overcloud upgrade converge` commands.

Example of configuration file changes:

```diff
parameter_defaults:  
- DockerCinderVolumeImage: 192.168.24.1:8787/dellemc/openstack-cinder-volume-dellemc-vxflexos
- DockerCinderBackupImage: 192.168.24.1:8787/dellemc/openstack-cinder-volume-dellemc-vxflexos
- DockerNovaComputeImage: 192.168.24.1:8787/dellemc/openstack-nova-compute-dellemc-vxflexos
- DockerGlanceApiImage: 192.168.24.1:8787/dellemc/openstack-glance-api-dellemc-vxflexos
- DockerGlanceApiConfigImage: 192.168.24.1:8787/dellemc/openstack-glance-api-dellemc-vxflexos
- DockerInsecureRegistryAddress:
-   - 192.168.24.1:8787
  NovaComputeOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  CinderVolumeOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  GlanceApiOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  GlanceBackend: cinder
  ControllerExtraConfig:
    cinder::config::cinder_config:
      scaleio/volume_driver:
-       value: cinder.volume.drivers.dell_emc.scaleio.driver.ScaleIODriver
+       value: cinder.volume.drivers.dell_emc.powerflex.driver.PowerFlexDriver
      scaleio/volume_backend_name:
        value: scaleio
      scaleio/san_ip:
        value: <PowerFlex GATEWAY IP>
      scaleio/san_login:
        value: <PowerFlex_USER>
      scaleio/san_password:
        value: <PowerFlex_PASSWD>
-     scaleio/sio_storage_pools:
+     scaleio/powerflex_storage_pools:
        value: <Comma-separated list of protection domain:storage pool name>
-     sio_allow_non_padded_volumes:
+     powerflex_allow_non_padded_volumes:
        value: True
    cinder_user_enabled_backends: ['scaleio']
```

**NOTICE**: The OS of the Overcloud node is upgraded by issuing `openstack overcloud upgrade run --stack <STACK NAME> --tags system_upgrade --limit <NODE TO UPGRADE>`. 
After OS upgrade of each Overcloud node where PowerFlex SDC is installed SDC package has to be reinstalled using
appropriate package for current OS version. 