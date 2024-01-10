# Dell EMC PowerFlex/VxFlex OS Backend Deployment Guide for Red Hat OpenStack Platform 16.1

Deployment tools for Dell EMC PowerFlex/VxFlex OS (formerly ScaleIO) support in RedHat OpenStack Platform 16.1.

## Overview

This instruction provide detailed steps on how to enable PowerFlex/VxFlex OS Driver.

**NOTICE**: this README represents only the **basic** steps necessary to enable PowerFlex/VxFlex OS driver. It does not contain steps on how update the overcloud or other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Platform 16.1](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.1/).

## Prerequisites

- Red Hat OpenStack Platform 16.1 with RHEL 8.2.
- PowerFlex 3.5
- PowerFlex/VxFlex OS Gateway has to be installed and accessible in the network.
- PowerFlex/VxFlex OS Storage Data Client (SDC) has to be installed on all OpenStack nodes.
- Configuration settings and credentials of Gateway.

## Steps

### Prepare the Environment File for PowerFlex/VxFlex OS cinder backend
The environment file is a OSP director environment file. The environment file contains the settings for each back end you want to define. Using the environment file will ensure that the back end settings persist through future Overcloud updates and upgrades.  

Create the environment file that will orchestrate the back end settings. Use the sample file provided below for your specific backend.  

Note: **LVM driver** is enabled by default in TripleO, you want to set the ```CinderEnableIscsiBackend``` to false in one of your environment file to turn it off.
```yaml
parameter_defaults:
  CinderEnableIscsiBackend: false
```

For full detailed instruction of all options please refer to [PowerFlex/VxFlex OS Backend Configuration](https://docs.openstack.org/cinder/train/configuration/block-storage/drivers/dell-emc-vxflex-driver.html)

**Environment sample**

With a director deployment, PowerFlex/VxFlex OS backend can be deployed using the integrated heat environment file. This file is located in the following path of the Undercloud node:
/usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-vxflexos-config.yaml

Copy this file to a local path where you can edit and invoke it later. For example, to copy it to ~/templates/:

```bash
$ cp /usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-vxflexos-config.yaml ~/templates/
```
Afterwards, open the copy (~/templates/cinder-dellemc-vxflexos-config.yaml) and edit it as you see fit. The following shows a sample content of the file. The files will list optional params that the user can choose to override if they don't like the default value.

Note that the ```resource_registry``` entry in the heat environment file must be an absolute path when you make a copy.

```yaml
# A Heat environment file which can be used to enable a
# a Cinder Dell EMC VxFlexOS backend, configured via puppet
resource_registry:
  OS::TripleO::Services::CinderBackendVxFlexOS: ../deployment/cinder/cinder-backend-vxflexos-puppet.yaml

parameter_defaults:
  CinderEnableVxFlexOSBackend: true
  CinderVxFlexOSBackendName: 'tripleo_dellemc_vxflexos'
  CinderVxFlexOSSanIp: ''
  CinderVxFlexOSSanLogin: ''
  CinderVxFlexOSSanPassword: ''
  CinderVxFlexOSStoragePools: 'domain1:pool1'
  CinderVxFlexOSAllowMigrationDuringRebuild: false
  CinderVxFlexOSAllowNonPaddedVolumes: false
  CinderVxFlexOSMaxOverSubscriptionRatio: 7.0
  CinderVxFlexOSRestServerPort: 443
  CinderVxFlexOSServerApiVersion: ''
  CinderVxFlexOSUnmapVolumeBeforeDeletion: false
  CinderVxFlexOSSanThinProvision: true
  CinderVxFlexOSDriverSSLCertVerify: false
  CinderVxFlexOSDriverSSLCertPath: '' 
```

### Prepare custom volume mappings for connector configuration 

On each node where VxFlex OS SDC will be installed, create the directory where the connector file will reside
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

Workaround for SElinux issue, see attached [SElinux_workaround_for_SDC_RHEL8.2.pdf](https://github.com/dell/osp-integration/edit/master/osp-deploy/cinder/powerflex/SElinux_workaround_for_SDC_RHEL8.2.pdf)


### Deploy the configured backends

When you have created the file dellemc-backend-env.yaml file with appropriate backends, deploy the backend configuration by running the openstack overcloud deploy command using the templates option. If you passed any extra environment files when you created the overcloud, pass them again here using the -e option. 
 
```bash
(undercloud) $ openstack overcloud deploy --templates \
-e /home/stack/templates/overcloud_images.yaml \
-e <other templates>
.....
-e /home/stack/templates/cinder-dellemc-vxflexos-config.yaml  \
-e /home/stack/templates/custom-dellemc-volume-mappings.yaml \
```

### Verify configured changes

After the deployment finishes successfully, `/etc/cinder/cinder.conf` in the Cinder container should reflect changes made above.

```ini
[DEFAULT]
enabled_backends = vxflexos

[vxflexos]
volume_driver = cinder.volume.drivers.dell_emc.vxflexos.driver.VxFlexOSDriver
volume_backend_name = vxflexos
san_ip = GATEWAY_IP
vxflexos_storage_pools = Domain1:Pool1,Domain2:Pool2
san_login = <login>
san_password = <password>
san_thin_provision = false
...
```

### Configure connector
Before using attach/detach volume operations VxFlex OS connector must be properly configured. On each node where VxFlex OS SDC is installed do the following:

Create /opt/emc/scaleio/openstack/connector.conf if it does not exist.

```bash
$ touch /opt/emc/powerflex/openstack/connector.conf
```

For each VxFlex OS section in the cinder.conf create the same section in the /opt/emc/scaleio/openstack/connector.conf and populate it with passwords. Example:

```
[vxflexos]
san_password = powerflex_password

[vxflexos-new]
san_password = powerflex_password
```

### SDC installation

Install the Storage Data Client (SDC) on all nodes after deploying the overcloud

### Testing the configured Backend
Finally create a volume-type and test if you can successfully create and attach volumes of that type.

