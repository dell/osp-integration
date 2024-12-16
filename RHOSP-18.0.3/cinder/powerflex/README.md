# Dell EMC PowerFlex Backend Deployment Guide for Red Hat OpenStack Platform 18.0

Deployment tools for Dell EMC PowerFlex (formerly VxFlex OS/ScaleIO) support in RedHat OpenStack Platform 18.0

## Overview

This instruction provide detailed steps on how to enable PowerFlex.

**NOTICE**: this README represents only the **basic** steps necessary to enable PowerFlex driver. It does not contain steps on how update the overcloud or other components of the system applicable to your particular installation.

For more information please refer to [Product Documentation for Red Hat OpenStack Platform 18.0](https://docs.redhat.com/en/documentation/red_hat_openstack_services_on_openshift/18.0/).

## Prerequisites

- Red Hat OpenStack Platform 18.0 with RHEL 9.6.
- PowerFlex 4.x cluster.
- PowerFlex Storage Data Client (SDC) for RHEL9.6.

## Steps

### Prepare the environment for PowerFlex cinder backend
To enable the OpenStack workloads to consume PowwerFlex as backend follow the below instructions:

* Create a Secret CR file powerflex-secret.yaml
  
```yaml  
apiVersion: v1
kind: Secret
metadata:
  labels:
    component: cinder-volume
    service: cinder
  name: cinder-volume-powerflex-secrets
stringData:
  powerflex-secrets.conf: |
    [powerflex]
    san_ip=PF_Manager_IP
    san_login=PF_Manager_login
    san_password=PF_Manager_password
type: Opaque
```

* Apply the CR file by
```yaml
oc create -f powerflex-secret.yaml
```

* Verify the secret is created
  ```yaml
  oc describe secret/cinder-volume-powerflex-secrets
  ```
For full detailed instruction of all options please refer to [PowerFlex Backend Configuration](https://docs.openstack.org/cinder/wallaby/configuration/block-storage/drivers/dell-emc-powerflex-driver.html).

**Environment sample**

With a director deployment, PowerFlex backend can be deployed using the integrated heat environment file. This file is located in the following path of the Undercloud node:
`/usr/share/openstack-tripleo-heat-templates/environments/cinder-dellemc-powerflex-config.yaml`.

Create the `custom-dellemc-powerflex-config.yaml` file in your `~/templates/` directory.

Edit the file with your favorite editor and fill in the following settings to fit your environment. 

**NOTE**: this method replaces the previous one which consisted of editing the heat environment file. This is nolonger considered as a best practice and the preferred approach is to store all overrides into a separate file

Afterwards, open the copy (`~/templates/cinder-dellemc-powerflex-config.yaml`) and edit it as you see fit. The following shows a sample content of the file. The files will list optional params that the user can choose to override if they don't like the default value.

```yaml
parameter_defaults:
  CinderEnablePowerFlexBackend: true
  CinderPowerFlexBackendName: 'tripleo_dellemc_powerflex'
  CinderPowerFlexSanIp: '<PowerFlex SAN IP>'
  CinderPowerFlexSanLogin: '<PowerFlex SAN Login>'
  CinderPowerFlexSanPassword: '<PowerFlex SAN Password>'
  CinderPowerFlexStoragePools: '<PowerFlex Domain:PowerFlex Storage Pool>'
```

**NOTE**: All other values will be inherited from /usr/share/openstack-tripleo-heat-templates/cinder-dellemc-powerflex-config.yaml, including the resource_registry entry.
### Prepare custom volume mappings for connector configuration 

PowerFlex SDC needs to be installed on the following nodes:
* Controller nodes 
* Compute nodes

Create the directory where the connector file will reside.

```bash
$ mkdir -p /opt/emc/scaleio/openstack
```

Edit the `custom-dellemc-powerflex-config.yaml` again and add the following parameters to the `parameter_defaults` section

```yaml
  NovaComputeOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  CinderVolumeOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  CinderBackupOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
  GlanceApiOptVolumes:
    - /opt/emc/scaleio/openstack:/opt/emc/scaleio/openstack
```
### Deploy the configured backends

When you have created the file `cinder-dellemc-powerflex-config.yaml` file with appropriate backends, deploy the backend configuration by running the openstack overcloud deploy command using the templates option. If you passed any extra environment files when you created the overcloud, pass them again here using the -e option. 
 
```bash
(undercloud) $ openstack overcloud deploy --templates \
-e /home/stack/templates/overcloud_images.yaml \
-e /usr/share/openstack-tripleo-heat-templates/cinder-dellemc-powerflex-config.yaml
-e <other templates>
.....
-e /home/stack/templates/custom-dellemc-powerflex-config.yaml
```

### Verify configured changes

After the deployment finishes successfully, `/etc/cinder/cinder.conf` in the Cinder container should reflect changes made above.

```ini
[DEFAULT]
enabled_backends = tripleo_dellemc_powerflex

[tripleo_dellemc_powerflex]
volume_driver = cinder.volume.drivers.dell_emc.powerflex.driver.PowerFlexDriver
volume_backend_name = tripleo_dellemc_powerflex
san_ip = POWERFLEX_GATEWAY_IP
powerflex_storage_pools = Domain1:Pool1,Domain2:Pool2
san_login = <login>
san_password = <password>
san_thin_provision = false
...
```
## Post deployment tasks

### Install SDC

Install the PowerFlex Storage Data Client (SDC) on all nodes after deploying the overcloud.

### Configure the connector

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
**NOTE**: In order to apply the changes, you will need to reboot each node on which the SDC kernel module (scini) runs.

### Test the configured Backend
Finally, create a PowerFlex volume type and test if you can successfully create and attach volumes of that type.

Run the following command to check whether the cinder-volume service is up. 
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume service list
+---------------+-------------------------------------+------+---------+-------+-----------------------------+
| Binary        | Host                                | Zone | Status  | State | Updated At                  |
+---------------+-------------------------------------+------+---------+-------+-----------------------------+
... [Truncated]
| cinder-volume | hostgroup@tripleo_dellemc_powerflex | nova | enabled | up    | 2024-01-12T02:36:02.000000  |
... [Truncated]
```
Create a volume type mapped to the deployed backend.
```
[stack@rhosp-undercloud ~]$ source ~/overcloudrc
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type create powerflex1
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume type set --property volume_backend_name=tripleo_dellemc_powerflex powerflex1
```
Create a volume using the type created above without error to ensure the availability of the backend.
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume create --type powerflex1 --size 8 powerflex_volume1
```
Confirm the volume was created successfully
```
(overcloud) [stack@rhosp-undercloud ~]$ openstack volume list
+--------------------------------------+-------------------+-----------+------+-------------+
| ID                                   | Name              | Status    | Size | Attached to |
+--------------------------------------+-------------------+-----------+------+-------------+
| 35808e76-c4cd-4ff6-8829-f16c76ebad37 | powerflex_volume1 | available | 8    |             |
+--------------------------------------+-------------------+-----------+------+-------------+
```
