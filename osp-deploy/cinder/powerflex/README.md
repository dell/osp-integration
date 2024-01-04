# Dell EMC PowerFlex Backend Deployment Guide for Red Hat OpenStack Platform 17.1

Deployment tools for Dell EMC PowerFlex (formerly VxFlex OS/ScaleIO) support in RedHat OpenStack Platform 17.1.

## Overview

This instruction provide detailed steps on how to enable PowerFlex.

**NOTICE**: this README represents only the **basic** steps necessary to enable PowerFlex driver. It does not contain steps on how update the overcloud or other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Platform 17.1](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/17.1/).

## Prerequisites

- Red Hat OpenStack Platform 17.1 with RHEL 9.2.
- PowerFlex 3.5 or 3.6
- PowerFlex Storage Data Client (SDC) has to be installed on all OpenStack nodes.

## Steps

### Prepare the Environment File for PowerFlex cinder backend
The environment file is a OSP director environment file. The environment file contains the settings for each back end you want to define. Using the environment file will ensure that the back end settings persist through future Overcloud updates and upgrades.  

Create the environment file that will orchestrate the back end settings. Use the sample file provided below for your specific backend.  

**NOTICE**: **LVM driver** is enabled by default in TripleO, set the ```CinderEnableIscsiBackend``` to false in one of your environment files to turn it off.
```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

For full detailed instruction of all options please refer to [PowerFlex Backend Configuration](https://docs.openstack.org/cinder/wallaby/configuration/block-storage/drivers/dell-emc-powerflex-driver.html).

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

PowerFlex SDC needs to be installed on the following nodes:
* Controller nodes 
* Compute nodes

Create the directory where the connector file will reside.

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
  CinderBackupOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  GlanceApiOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
```
### Workarounds

Workaround for SElinux issue, see attached [SElinux_workaround_for_SDC_RHEL8.2.pdf](https://github.com/dell/osp-integration/blob/master/osp-deploy/cinder/powerflex/SElinux_workaround_for_SDC_RHEL8.2.pdf).


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

### Install SDC

Install the Storage Data Client (SDC) on all nodes after deploying the overcloud

### Test the configured Backend
Finally create a volume-type and test if you can successfully create and attach volumes of that type.



