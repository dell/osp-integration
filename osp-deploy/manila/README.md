
# Dell EMC Manila Backend Deployment Guide for Red Hat OpenStack Platform 16

## Overview

This document describes how to deploy the Dell EMC Shared File Systems services in a Red Hat OpenStack Platform Overcloud.
This assumes that the RHOSP installion is through RHOSP director toolset which is based primarily on the upstream TripleO project.  

The following Dell EMC storage drivers that are fully integrated with director and can be deployed using tripleo heat templates 
* [VMAX Driver](https://docs.openstack.org/manila/rocky/configuration/shared-file-systems/drivers/dell-emc-vmax-driver.html) 
* [Unity Driver](https://docs.openstack.org/manila/rocky/configuration/shared-file-systems/drivers/dell-emc-unity-driver.html) 
* [VNX Driver](https://docs.openstack.org/manila/rocky/configuration/shared-file-systems/drivers/dell-emc-vnx-driver.html) 
 

## Prerequisites
- Dell EMC Shared File Systems Backend configured as file system repository.
- Configuration settings and credentials.

## Deployment Steps

### Prepare the Environment File
The environment file is a OSP director environment file. The environment file contains the settings for each back end you want to define. Using the environment file will ensure that the back end settings persist through future Overcloud updates and upgrades.  

Create the environment file that will orchestrate the back end settings. Use the sample file provided below for your specific backend.  


**1. VMAX Driver**

For full detailed instruction of all options please refer to [VMAX Shared File Systems Configuration](https://docs.openstack.org/manila/rocky/configuration/shared-file-systems/drivers/dell-emc-vmax-driver.html).

**Environment sample**

With a director deployment, VMAX manila backend can be deployed using the integrated heat environment file. This file is located in the following path of the Undercloud node:
/usr/share/openstack-tripleo-heat-templates/environments/manila-vmax-config.yaml.yaml

Copy this file to a local path where you can edit and invoke it later. For example, to copy it to ~/templates/:

```bash
$ cp /usr/share/openstack-tripleo-heat-templates/environments/manila-vmax-config.yaml ~/templates/
```
Afterwards, open the copy (~/templates/manila-vmax-config.yaml) and edit it as you see fit. The following shows a sample content of the file. The files will list optional params that the user can choose to override if they don't like the default value.

Note that the ```resource_registry``` entry in the heat environment file must be an absolute path when you make a copy.

```yaml
# This environment file enables Manila with the VMAX backend.
resource_registry:
  OS::TripleO::Services::ManilaApi: ../deployment/manila/manila-api-container-puppet.yaml
  OS::TripleO::Services::ManilaScheduler: ../deployment/manila/manila-scheduler-container-puppet.yaml
  # Only manila-share is pacemaker managed:
  OS::TripleO::Services::ManilaShare: ../deployment/manila/manila-share-pacemaker-puppet.yaml
  OS::TripleO::Services::ManilaBackendVMAX: ../deployment/manila/manila-backend-vmax.yaml

parameter_defaults:
  ManilaVMAXBackendName: tripleo_manila_vmax
  ManilaVMAXDriverHandlesShareServers: true
  ManilaVMAXNasLogin: ',user>'
  ManilaVMAXNasPassword: '<password>'
  ManilaVMAXNasServer: '<ip address>'
  ManilaVMAXServerContainer: '<Data Mover name>'
  ManilaVMAXShareDataPools: '<Comma separated pool names>'
  ManilaVMAXEthernetPorts: '<Comma separated ports list>'
  
```


**2. Unity Driver**

For full detailed instruction of options please refer to [Unity Shared File Systems  Configuration](https://docs.openstack.org/manila/rocky/configuration/shared-file-systems/drivers/dell-emc-unity-driver.html)

Please refer to this [custom deployment guide for the Unity Driver](https://github.com/emc-openstack/osp-deploy/tree/rhosp16/manila)



**2. VNX Driver**

For full detailed instruction of options please refer to [VNX Shared File Systems  Configuration](https://docs.openstack.org/manila/rocky/configuration/shared-file-systems/drivers/dell-emc-vnx-driver.html)

Please refer to this [custom deployment guide for the VNX driver](https://github.com/emc-openstack/osp-deploy/tree/rhosp16/manila)


### Deploy the configured backends

When you have created the file dellemc-backend-env.yaml file with appropriate backends, deploy the backend configuration by running the openstack overcloud deploy command using the templates option. If you passed any extra environment files when you created the overcloud, pass them again here using the -e option. 
 
```bash
(undercloud) $ openstack overcloud deploy --templates \
-e /home/stack/templates/overcloud_images.yaml \
-e <other templates>
.....
-e /home/stack/templates/dellemc-backend-env.yaml  \
```

### Verify the configured changes

When the director completes the overcloud deployment, check that the manila share services are up using the openstack cli command. You can also verify that the manila.conf in the manila container and it should reflect changes made above.
``` bash
$openstack share service list
```
### Testing the configured Backend
After you deploy the back ends to the overcloud, create a share-type per backend and test if you can successfully create and attach shares of that type.







