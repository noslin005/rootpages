# OpenStack

* [Introduction](#introduction)
* [Overview](#overview)
* [Configurations](#configurations)
    * [Common](#configurations---common)
        * [Database](#configurations---common---database)
    * [Keystone](#configurations---keystone)
        * [Token Provider](#configurations---keystone---token-provider)
        * [API v3](#configurations---keystone---api-v3)
    * [Nova](#configurations---nova)
        * [Hypervisors](#configurations---nova---hypervisors)
        * [CPU Pinning](#configurations---nova---cpu-pinning)
    * [Neutron](#configurations---neutron)
        * [DNS](#configurations---neutron---dns)
        * [Metadata](#configurations---neutron---metadata)
        * [Load-Balancing-as-a-Service](#configurations---neutron---load-balancing-as-a-service)
        * [Quality of Service](#configurations---neutron---quality-of-service)
* [Command Line Interface Utilities](#command-line-interface-utilities)
* [Automation](#automation)
    * [Heat](#automation---heat)
        * [Resources](#automation---heat---resources)
        * [Parameters](#automation---heat---parameters)
    * [Vagrant](#automation---vagrant)
* [Testing](#testing)
    * [Tempest](#testing---tempest)
* [Performance](#performance)


# Introduction

This guide is aimed to help guide System Administrators through OpenStack. It is assumed that the cloud is using these industry standard software:

* OpenStack Mitaka
* CentOS 7 (Linux)

Most things mentioned here should be able to be applied to other similar environments.


# Overview

OpenStack has a large range of services. The essential ones required for a basic cloud are:

* Keystone
  * Authentication
* Nova
  * Compute
* Neutron
  * Networking


# Configurations

This section will focus on important settings for each service's configuration files.


## Configurations - Common

These are general configuration options that apply to most OpenStack configuration files.


### Configurations - Common - Database

Different database backends can be used by the API services on the controller nodes.

* MariaDB/MySQL. Requires "PyMySQL." Starting with Liberty, this is prefered over using "mysql://" as the latest OpenStack libraries are written for PyMySQL connections (not to be confused with "MySQL-python"). [1]
```
[ database ] connection = mysql+pymysql://<USER>:<PASSWORD>@<MYSQL_HOST>/<DATABASE>
```
* PostgreSQL. Requires "psycopg2." [2]
```
[ database ] connection = postgresql://<USER>:<PASSWORD>@<MYSQL_HOST>/<DATABASE>
```
* SQLite.
```
[ database ] connection = sqlite:///<DATABASE>.sqlite
```

Sources:

1. "DevStack switching from MySQL-python to PyMySQL." OpenStack nimeyo. Jun 9, 2015. Accessed October 15, 2016. https://openstack.nimeyo.com/48230/openstack-all-devstack-switching-from-mysql-python-pymysql
2. "Using PostgreSQL with OpenStack." FREE AND OPEN SOURCE SOFTWARE KNOWLEDGE BASE. June 06, 2014. Accessed October 15, 2016. https://fosskb.in/2014/06/06/using-postgresql-with-openstack/


## Configurations - Keystone


### Configurations - Keystone - API v3

In Mitaka, the Keystone v2.0 API has been deprecated. It will be removed entirely from OpenStack in the "Q" release. [1] It is possible to run both v2.0 and v3 at the same time but it's desirable to move towards the v3 standard. For disabling v2.0 entirely, Keystone's API paste configuration needs to have these lines removed (or commented out) and then the web server should be restarted.

* /etc/keystone/keystone-paste.ini
    * [pipeline:public_api] pipeline = cors sizelimit url_normalize request_id admin_token_auth build_auth_context token_auth json_body ec2_extension public_service
    * [pipeline:admin_api] pipeline = cors sizelimit url_normalize request_id admin_token_auth build_auth_context token_auth json_body ec2_extension s3_extension admin_service
    * [composite:main] /v2.0 = public_api
    * [composite:admin] /v2.0 = admin_api

[2]

Sources:

1. "Mitaka Series Release Notes." Accessed October 16, 2016. http://docs.openstack.org/releasenotes/keystone/mitaka.html
2. "Setting up an RDO deployment to be Identity V3 Only." Young Logic. May 8, 2015. Accessed October 16, 2016. https://adam.younglogic.com/2015/05/rdo-v3-only/


### Configurations - Keystone - Token Provider

The token provider is used to create and delete tokens for authentication. Different providers can be used as the backend. These are configured in the /etc/keystone/keystone.conf file.

#### Scenario #1 - UUID (default)

* [ token ] provider = uuid

#### Scenario #2 - PKI

* [ token ] provider = pki
* Create the certificates. A new directory "/etc/keystone/ssl/" will be used to store these files.
```
# keystone-manage pki_setup --keystone-user keystone --keystone-group keystone
```

#### Scenario #3 - Fernet (fastest token creation)

* [ token ] provider = fernet
* [ fernet_tokens ] key_repository = /etc/keystone/fernet-keys/
* Create the Fernet keys:
```
# mkdir /etc/keystone/fernet-keys/
# chmod 750 /etc/keystone/fernet-keys/
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

[2]

Sources:

1. "Configuring Keystone." OpenStack Documentation. Accessed October 16, 2016. http://docs.openstack.org/developer/keystone/configuration.html
2. "OpenStack Keystone Fernet tokens." Dolph Mathews. Accessed August 27th, 2016. http://dolphm.com/openstack-keystone-fernet-tokens/


## Configurations - Nova

* /etc/nova/nova.conf
    * [ libvirt ] inject_key = false
	    * Do not inject SSH keys via Nova. This should be handled by the Nova's metadata service. This will either be "openstack-nova-api" or "openstack-nova-metadata-api" depending on your setup.
    * [ DEFAULT ] enabled_apis = osapi_compute,metadata
	    * Enable support for the Nova API, and Nova's metadata API. If "metedata" is specified here, then the "openstack-nova-api" handles the metadata and not "openstack-nova-metadata-api."
    * [ api_database ] connection = connection=mysql://nova:password@10.1.1.1/nova_api
    * [ database ] connection = mysql://nova:password@10.1.1.1/nova
	    * For the controller nodes, specify the connection SQL connection string. In this example it uses MySQL, the MySQL user "nova" with a password called "password", it connects to the IP address "10.1.1.1" and it is using the database "nova."


### Configurations - Nova - Hypervisors

Nova supports a wide range of virtualization technologies. Full hardware virtulization, paravirutlization, or containers can be used. Even Windows' Hyper-V is supported. This is configured in the /etc/nova/nova.conf file. [1]

#### Scenario #1 - KVM

* [DEFAULT] compute_driver = libvirt.LibvirtDriver
* [libvirt] virt_type = kvm

[2]

#### Scenario #2 - Xen

* [DEFAULT] compute_driver = libvirt.LibvirtDriver
* [libvirt] virt_type = xen

[3]

#### Scenario #3 - LXC

* [DEFAULT] compute_driver = libvirt.LibvirtDriver
* [libvirt] virt_type = lxc

[4]

#### Scenario #4 - Docker

Docker is not available by default in OpenStack. First it must be installed before configuring it in Nova.
```
# git clone https://github.com/openstack/nova-docker.git
# cd nova-docker/
# pip install -r requirements.txt
# python setup.py install
```

* [DEFAULT] compute_driver=novadocker.virt.docker.DockerDriver

[5]

Sources:

1. "Hypervisors." OpenStack Configuration Reference - Liberty. Accessed August 28th, 2016. http://docs.openstack.org/liberty/config-reference/content/section_compute-hypervisors.html
2. "KVM." OpenStack Configuration Reference - Liberty. Accessed August 28th, 2016. http://docs.openstack.org/liberty/config-reference/content/kvm.html
3. "Xen." OpenStack Configuration Reference - Liberty. Accessed August 28th, 2016. http://docs.openstack.org/liberty/config-reference/content/xen_libvirt.html
4. "LXC." OpenStack Configuration Reference - Liberty. Accessed August 28th, 2016. http://docs.openstack.org/liberty/config-reference/content/lxc.html
5. "nova-docker." GitHub.com. December, 2015. Accessed August 28th, 2016. https://github.com/openstack/nova-docker


### Configurations - Nova - CPU Pinning

Verify that the processor(s) has hardware support for non-uniform memory access (NUMA). If it does, NUMA may still need to be turned on in the BIOS. NUMA nodes are the physical processors. These processors are then mapped to specific sectors of RAM. [1]

```
# lscpu | grep NUMA
NUMA node(s):          2
NUMA node0 CPU(s):     0-9,20-29
NUMA node1 CPU(s):     10-19,30-39
```
```
# numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 20 21 22 23 24 25 26 27 28 29
node 0 size: 49046 MB
node 0 free: 31090 MB
node 1 cpus: 10 11 12 13 14 15 16 17 18 19 30 31 32 33 34 35 36 37 38 39
node 1 size: 49152 MB
node 1 free: 31066 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```
```
# virsh nodeinfo | grep NUMA
NUMA cell(s):        2
```

[1][3]


#### Configuration

* Append the two NUMA filters.
```
# vim /etc/nova/nova.conf
[ DEFAULT ] scheduler_default_filters = RetryFilter,AvailabilityZoneFilter,RamFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImageProp
ertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,NUMATopologyFilter,AggregateInstanceExtraSpecsFilter
```

* Restart the Nova scheduler service on the controller node(s).
```
# systemctl restart openstack-nova-scheduler
```

* Set the aggregate/availability zone to allow pinning.
```
# nova aggregate-set-metadata <AGGREGATE_ZONE> pinned=true
```

* Modify a flavor to provide dedicated CPU pinning.
```
# nova flavor-key <FLAVOR_ID> set hw:cpu_policy=dedicated
```

[1][2][3]

Sources:

1. http://redhatstackblog.redhat.com/2015/05/05/cpu-pinning-and-numa-topology-awareness-in-openstack-compute/
2. http://docs.openstack.org/admin-guide/compute-flavors.html
3. http://www.stratoscale.com/blog/openstack/cpu-pinning-and-numa-awareness/


## Configurations - Neutron


### Configurations - Neutron - DNS

By default, Neutron does not provide any DNS resolvers. This means that DNS will not work. It is possible to either provide a default list of name servers or configure Neutron to refer to the relevant /etc/resolv.conf file on the server.

#### Scenario #1 - Define default resolvers (recommended)

* /etc/neutron/dhcp_agent.ini
    * [ DEFAULT ] dnsmasq_dns_servers = 8.8.8.8,8.8.4.4

#### Scenario #2 - Leave resolvers to be configured by the subnet details

* Nothing needs to be configured.

#### Scenario #3 - Do not provide resolvers

* /etc/neutron/dhcp_agent.ini
    * [ DEFAULT ] dnsmasq_local_resolv = True

[1]

Source:

1. "Name resolution for instances." OpenStack Documentation. August 09, 2016. Accessed August 13th, 2016. http://docs.openstack.org/mitaka/networking-guide/config-dns-resolution.html


### Configurations - Neutron - Metadata

The metadata service provides useful information about the instance from the IP address 169.254.169.254/32. This service is also used to communicate with "cloud-init" on the instance to configure SSH keys and other post-boot tasks.

Assuming authentication is already configured, set these options for the OpenStack environment. These are the basics needed before the metadata service can be used correctly. Then you can choose to use DHCP namespaces (layer 2) or router namespaces (layer 3) for delievering/recieving requests.

* /etc/neutron/metadata_agent.ini
    * [ DEFAULT ] nova_metadata_ip = CONTROLLER_IP
    * [ DEFAULT ] metadata_proxy_shared_secret = SECRET_KEY
* /etc/nova/nova.conf
    * [ DEFAULT ] enabled_apis = osapi_compute,metadata
    * [ neutron ] service_metadata_proxy = True
    * [ neutron ] metadata_proxy_shared_secret = SECRET_KEY

#### Scenario #1 - DHCP Namespace (recommended for DVR)

* /etc/neutron/dhcp_agent.ini
    * [ DEFAULT ] force_metadata = True
    * [ DEFAULT ] enable_isolated_metadata = True
    * [ DEFAULT ] enable_metadata_network = True
* /etc/neutron
    * [ DEFAULT ] enable_metadata_proxy = False

#### Scenario #2 - Router Namespace

* /etc/neutron/dhcp_agent.ini
    * [ DEFAULT ] force_metadata = False
    * [ DEFAULT ] enable_isolated_metadata = True
    * [ DEFAULT ] enable_metadata_network = False
* /etc/neutron/l3_agent.ini
    * [ DEFAULT ] enable_metadata_proxy = True

[1]

Source:

1. "Introduction of Metadata Service in OpenStack." VietStack. September 09, 2014. Accessed August 13th, 2016. https://vietstack.wordpress.com/2014/09/27/introduction-of-metadata-service-in-openstack/


### Configurations - Neutron - Load-Balancing-as-a-Service

Load-Balancing-as-a-Service version 2 (LBaaSv2) has been stable since Liberty. It can be configured with either the HAProxy or Octavia back-end.

* /etc/neutron/neutron.conf
    * [ DEFAULT ] service_plugins = `<PLUGINS>`, neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
        * Append the LBaaSv2 service plugin.
*   /etc/neutron/lbaas_agent.ini
    * [ DEFAULT ] interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
        * This is for Neutron with the OpenVSwitch backend only.
    * [ DEFAULT ] interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
        * This is for Neutron with the Linux Bridge backend only.

#### Scenario #1 - HAProxy (recommended for it's maturity)

* /etc/neutron/neutron_lbaas.conf
    * [ service_providers ] service_provider = LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
* /etc/neutron/lbaas_agent.ini
    * [ DEFAULT ] device_driver = neutron_lbaas.drivers.haproxy.namespace_driver.HaproxyNSDriver
    * [ haproxy ] user_group = haproxy
        * Specify the group that HAProxy runs as. In RHEL, it's "haproxy."

#### Scenario #2 - Octavia

* /etc/neutron/neutron_lbaas.conf
    * [ service_providers ] service_provider = LOADBALANCERV2:Octavia:neutron_lbaas.drivers.octavia.driver.OctaviaDriver:default

[1]

Source:

1. http://docs.openstack.org/draft/networking-guide/config-lbaas.html


### Configurations - Neutron - Quality of Service

The Quality of Service (QoS) plugin can be used to rate limit the amount of bandwidth that is allowed through a network port.

* /etc/neutron/neutron.conf
    * [ DEFAULT ] service_plugins = neutron.services.qos.qos_plugin.QoSPlugin
        * Append the QoS plugin to the list of service_plugins.
* /etc/neutron/plugins/ml2/openvswitch_agent.ini
    * [ml2] extension_drivers = qos
        * Append the QoS driver with the modular layer 2 plugin provider. Alternatively to OpenVSwitch, LinuxBridge and SR-IOV are also supported for quality of service.
* /etc/neutron/plugins/ml2/ml2_conf.ini
    * [agent] extensions = qos
        * Lastly, append the QoS extension to the moduler layer 2 configuration.

[1]

Source:

1. "Quality of Service (QoS)." OpenStack Documentation. October 10, 2016. Accessed October 16, 2016. http://docs.openstack.org/draft/networking-guide/config-qos.html


# Command Line Interface Utilities

The OpenStack CLI resources used to be handled by seperate commands. These have all been modified and are managed by the universal "openstack" command. The various options and arguments are explained in Root Pages' OpenStack section [Linux Commands excel sheet](https://raw.githubusercontent.com/ekultails/rootpages/master/linux_commands.xlsx).

For the CLI utilies to work, the environment variables need to be set for the project and user. This way the commands can automatically authenticate.

* Add the credentials to a text file This is generally ends with the ".sh" extension to signify it's a shell file. A few default variables are filled in below.
  * Keystone v2.0
```
# unset any variables used
unset OS_PROJECT_ID
unset OS_PROJECT_NAME
unset OS_PROJECT_DOMAIN_ID
unset OS_PROJECT_DOMAIN_NAME
unset OS_USER_ID
unset OS_USER_NAME
unset OS_USER_DOMAIN_ID
unset OS_USER_DOMAIN_NAME
unset OS_REGION_ID
unset OS_REGION_NAME
# fill in the project, user, and endpoint details
export PROJECT_ID=
export PROJECT_NAME=
export OS_USERNAME=
export OS_PASSWORD=
export OS_REGION_NAME="RegionOne"
export OS_AUTH_URL="http://controller1:5000/v2.0"
export OS_AUTH_VERSION="2.0"
```
  * Keystone v3
```
# unset any variables used
unset OS_PROJECT_ID
unset OS_PROJECT_NAME
unset OS_PROJECT_DOMAIN_ID
unset OS_PROJECT_DOMAIN_NAME
unset OS_USER_ID
unset OS_USER_NAME
unset OS_USER_DOMAIN_ID
unset OS_USER_DOMAIN_NAME
unset OS_REGION_ID
unset OS_REGION_NAME
# fill in the project, user, and endpoint details
export OS_PROJECT_ID=
export OS_PROJECT_NAME=
export OS_PROJECT_DOMAIN_NAME="default"
export OS_USERID=
export OS_USERNAME=
export OS_PASSWORD=
export OS_USER_DOMAIN_NAME="default"
export OS_REGION_NAME="RegionOne"
export OS_AUTH_URL="http://controller1:5000/v3"
export OS_AUTH_VERSION="3"
```

* Source the credential file to load it into the shell environment:
```
$ source <USER_CREDENTIALS_FILE>.sh
```

* View the available command line options.
```
$ openstack help
```
```
$ openstack help <OPTION>
```

[1]

Source:

1. "OpenStack Command Line." OpenStack Documentation. Accessed October 16, 2016. http://docs.openstack.org/developer/python-openstackclient/man/openstack.html


# Automation

Automating deployments can be accomplished in a few ways. The built-in OpenStack way is via Orhcestration as a Service (OaaS), typically handled by Heat. It is also possible to use Ansible or Vagrant to automate OpenStack deploys.


## Automation - Heat

Heat is used to orchestrate the deployment of multiple OpenStack components at once. It can also install and configure software on newly built instances.


## Automation - Heat - Resources

Heat templates are made of multiple resources. All of the different resource types are listed here [http://docs.openstack.org/developer/heat/template_guide/openstack.html](http://docs.openstack.org/developer/heat/template_guide/openstack.html). Resources use properties to create a component. If no name is specified (for example, a network name), a random string will be used. Most properties also accept either an exact ID of a resource or a reference to a dynamically generated resource (which will provide it's ID once it has been created).

This section will go over examples of the more common modules.

Syntax:
```
<DESCRIPTIVE_OBJECT_NAME>:
    type: <HEAT_RESOURCE_TYPE>
    properties:
        <PROPERTY_1>: <VALUE_1>
        <PROPERTY_2>: <VALUE_2>
```

For referencing created resources (for example, creating a subnet in a created network) the "get_resource" function should be used.
```
{ get_resource: <OBJECT_NAME> }
```

* Create a network, assigned to the "internal_network" object.
```
internal_network: {type: 'OS::Neutron::Net'}
```

* Create a subnet for the created network. Required properties: network name or ID.
```
internal_subnet:
    type: OS::Neutron::Subnet
    properties:
      ip_version: 4
      cidr: 10.0.0.0/24
      dns_nameservers: [8.8.4.4, 8.8.8.8]
      network_id: {get_resource: internal_network}
```

* Create a port. This object can be used during the instance creation. Required properties: network name or ID.
```
subnet_port:
    type: OS::Neutron::Port
    properties:
        fixed_ips:
            - subnet_id: {get_resource: internal_subnet}
        network: {get_resource: internal_network}
```

* Create a router associated with the public "ext-net" network.
```
external_router:
    type: OS::Neutron::Router
    properties:
        external_gateway_info: [ network: ext-net ]
```

* Attach a port from the network to the router.
```
external_router_interface:
    type: OS::Neutron::RouterInterface
    properties:
        router: {get_resource: external_router}
        subnet: {get_resource: internal_subnet}
```

* Create a key pair called "HeatKeyPair." Required property: name.
```
ssh_keys:
    type: OS::Nova::KeyPair
    properties:
        name: HeatKeyPair
        public_key: HeatKeyPair
        save_private_key: true
```

* Create an instance using the "m1.small" flavor, "centos7" image, assign the subnet port creaetd by "subnet_port" and use the "default" security group.
```
instance_creation:
  type: OS::Nova::Server
  properties:
    flavor: m1.small
    image: centos7
    networks:
        - port: {get_resource: subnet_port}
    security_groups: {default}
```

* Allocate an IP from the "ext-net" floating IP pool.
```
floating_ip:
    type: OS::Neutron::FloatingIP
    properties: {floating_network: ext-net}
```

* Allocate a a floating IP to the created instance from a "instance_creation" function. Alternatively, a specific instance's ID can be defined here.
```
floating_ip_association:
    type: OS::Nova::FloatingIPAssociation
    properties:
	    floating_ip: {get_resource: floating_ip}
		server_id: {get_resource: instance_creation}
```

Source:

1. "OpenStack Orchestration In Depth, Part I: Introduction to Heat." Accessed September 24, 2016. November 7, 2014. https://developer.rackspace.com/blog/openstack-orchestration-in-depth-part-1-introduction-to-heat/


## Automation - Heat - Parameters

Parameters allow users to input custom variables for Heat templates.

Options:
* type = The input type. This can be a string, number, JSON, a comma seperated list or a boolean.
* label = String. The text presented to the end-user for the fillable entry.
* description = String. Detailed information about the parameter.
* default = A default value for the parameter.
* constraints = A parameter has to match a specified constrant. Any number of constraints can be used from the available ones below.
  * length = How long a string can be.
  * range = How big a number can be.
  * allowed_values = Allow only one of these specific values to be used.
  * allowed_pattern = Allow only a value matching a regular expression.
  * custom_constraint = A full list of custom service constraints can be found at [http://docs.openstack.org/developer/heat/template_guide/hot_spec.html#custom-constraint](#http://docs.openstack.org/developer/heat/template_guide/hot_spec.html#custom-constraint).
* hidden = Boolean. Specify if the text entered should be hidden or not. Default: false.
* immutable = Boolean. Specify whether this variable can be changed. Default: false.

Syntax:
```
parameters:
	<CUSTOM_NAME>:
    	type: string
        label: <LABEL>
        description: <DESCRIPTION>
        default: <DEFAULT_VALUE>
        constraints:
        	- length: { min: <MINIMUM_NUMBER>, max: <MAXIMUM_NUMBER> }
        	- range: { min: <MINIMUM_NUMBER>, max: <MAXIMUM_NUMBER> }
        	- allowed_values: [ <VALUE1>, <VALUE2>, <VALUE3> ]
        	- allowed_pattern: <REGULAR_EXPRESSION>
        	- custom_constrant: <CONSTRAINT>
		hidden: <BOOLEAN>
        immutable: <BOOLEAN>
```

For referencing this parameter elsewhere in the Heat template, use this syntax for the variable:
```
{ get_param: <CUSTOM_NAME> }
```

[1]

Source:

1. "Heat Orchestration Template (HOT) specification." OpenStack Developer Documentation. October 21, 2016. Accessed October 22, 2016. http://docs.openstack.org/developer/heat/template_guide/hot_spec.html


## Automation - Vagrant

Vagrant is a tool to automate the deployment of virtual machines. A "Vagrantfile" file is used to initalize the instance. An example is provided below. Note that Vagrant currently does not support Keystone's v3 API.

```
require 'vagrant-openstack-provider'

Vagrant.configure('2') do |config|

  config.vm.box       = 'vagrant-openstack'
  config.ssh.username = 'cloud-user'

  config.vm.provider :openstack do |os|
    os.openstack_auth_url = 'http://controller1/v2.0/tokens'
    os.username           = 'openstackUser'
    os.password           = 'openstackPassword'
    os.tenant_name        = 'myTenant'
    os.flavor             = 'm1.small'
    os.image              = 'centos'
    os.networks           = "vagrant-net"
    os.floating_ip_pool   = 'publicNetwork'
    os.keypair_name       = "private_key"
  end
end
```

Once those settings are configured for the end user's cloud environment, it can be created by running:

```
$ vagrant up --provider=openstack
```

[1]

Source:

1. "Vagrant OpenStack Cloud Provider." GitHub - ggiamarchi. Accessed September 24, 2016. April 30, 2016. https://github.com/ggiamarchi/vagrant-openstack-provider


# Testing


## Testing - Tempest

Tempest is used to query all of the different APIs in use. This helps to validate the functionality of OpenStack. This software is a rolling release aimed towards verifying the latest OpenStack release in development but it should also work for older versions as well.

The sample configuration flie "/etc/tempest/tempest.conf.sample" should be copied to "/etc/tempest/tempest.conf" and then modified. If it is not available then the latest configuration file can be downloaded from one of thes sources:
* http://docs.openstack.org/developer/tempest/sampleconf.html
* http://docs.openstack.org/developer/tempest/_static/tempest.conf.sample


* Provide credentials to a user with the "admin" role.
```
[auth]
admin_username
admin_password
admin_project_name
admin_domain_name
default_credentials_domain_name = Default
```

* Specify the Keystone version to use. Valid options are "v2" and "v3."
```
[identity]
auth_version
```

* Provide the admin Keystone endpoint for v2 (uri) or v3 (uri_v3).
```
[identity]
uri
uri_v3
```

* Two different size flavor IDs should be given.
```
[compute]
flavor_ref
flavor_ref_alt
```

* Two different image IDs should be given.
```
[compute]
image_ref
image_ref_alt
```

* Define what services should be tested for the specific cloud.
```
[service_available]
cinder = true
neutron = true
glance = true
swift = false
nova = true
heat = false
sahara = false
ironic = false
```

[1]

Source:

1. "Tempest Configuration Guide." Sep 14th, 2016. http://docs.openstack.org/developer/tempest/configuration.html


# Performance

A few general tips for getting the fastest OpenStack performance.

* KeyStone
  * Switch to Fernet keys.
    * Creation of tokens is significantly faster.
    * Refer to [Configurations - Keystone - Token Provider](#configurations---keystone---token-provider).
* Neutron
  * Use distributed virtual routing (DVR).
    * This offloads a lot of networking resources onto the compute nodes.
* General
  * Utilize /etc/hosts.
    * Ensure that all of your domain names (including the public IP) are listed in the /etc/hosts. This avoids a performance hit from DNS lookups. Alternatively, consider setting up a recursive DNS server on the controller nodes.
  * Use memcache.
    * This is generally configured by an option called "memcache_servers" in the configuration files for most services. Consider using "CouchBase" for its ease of clustering and redudancy support.