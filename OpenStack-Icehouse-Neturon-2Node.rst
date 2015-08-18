####
OpenStack Icehouse 설치 가이드(2 개 N/W 인터페이스를 갖는 2개 노드 설치)
####

.. contents::
  

Basic Architecture and Network Configuration
==========================================

In this installation guide, we cover the step-by-step process of installing Openstack Icehouse on Ubuntu 14.04.  We consider a multi-node architecture with Openstack Networking (Neutron) that requires two node types: 

+ **Controller & Network Node** that runs management services (keystone, Horizon…) needed for OpenStack to function and that runs networking services and is responsible for virtual network provisioning  and for connecting virtual machines to external networks.

+ **Compute Node** that runs the virtual machine instances in OpenStack. 

We have deployed a single compute node (see the Figure below) but you can simply add more compute nodes to our multi-node installation, if needed.  


.. image:: https://github.com/GunSik2/OpenStack/blob/master/images/arch1.jpg

For OpenStack Multi-Node setup you need to create two networks:

+ **Management & Public Network** (192.168.100.0/24): A network segment used for administration, not accessible to the public Internet. This network is connected to the controller nodes so users can access the OpenStack interfaces, and connected to the network nodes to provide VMs with publicly routable traffic functionality.

+ **VM Traffic Network** (10.10.1.0/24): This network is used as internal network for traffic between virtual machines in OpenStack, and between the virtual machines and the network nodes that provide L3 routes out to the public network.

In the next subsections, we describe in details how to set up, configure and test the network architecture. We want to make sure everything is ok before install ;)

So, let’s prepare the nodes for OpenStack installation!


Configure Controller node
-------------------------

The controller node has two Network Interfaces: eth0 is internal (used for connectivity for OpenStack nodes) and eth1 is external.

* Change to super user mode::

    sudo su

* Set the hostname::

    vi /etc/hostname
    controller


* Edit /etc/hosts::

    vi /etc/hosts
        
    #controller & network
    10.0.1.21       controller
    192.168.100.21  pcontroller
        
    # compute1  
    10.0.0.31       compute1


* Edit network settings to configure the interfaces eth0 and eth1::

    vi /etc/network/interfaces
      
    # The management & public network interface
      auto eth0
      iface eth0 inet static
      address 192.168.100.21
      netmask 255.255.255.0
      gateway 192.168.100.1
      dns-nameservers 8.8.8.8
    
    # VM traffic interface
      auto eth1
      iface eth1 inet static
      address 10.0.1.21
      netmask 255.255.255.0

* Restart network::

    ifdown eth0 && ifup eth0
    ifdown eth1 && ifup eth1


Configure Compute node
----------------------
The network node has two network Interfaces: eth0 for management use and eth1 for connectivity between VMs.

* Change to super user mode::

    sudo su

* Set the hostname::

    vi /etc/hostname
    compute1


* Edit /etc/hosts::

    vi /etc/hosts
    
    # compute1
    10.0.1.31       compute1
  
    #controller & network
    10.0.1.11       controller
    192.168.100.21  pcontroller

* Edit network settings to configure the interfaces eth0 and eth1::

    vi /etc/network/interfaces
  
    # The management network interface    
      auto eth0
      iface eth0 inet static
      address 192.168.100.31
      netmask 255.255.255.0
  
    # VM traffic interface     
      auto eth1
      iface eth1 inet static
      address 10.0.1.31
      netmask 255.255.255.0


* Restart network::
  
    ifdown eth0 && ifup eth0
      
    ifdown eth1 && ifup eth1


Verify connectivity
-------------------

We recommend that you verify network connectivity to the internet and among the nodes before proceeding further.

    
* From the controller node::

    # ping a site on the internet:
    ping openstack.org

    # ping the management interface on the compute node:
    ping compute1

* From the compute node::

    # ping a site on the internet:
    ping openstack.org

    # ping the management interface on the controller node:
    ping controller


Install 
=======

Now everything is ok :) So let's go ahead and install it !


Controller Node
---------------

Here we will install the basic services (keystone, glance, nova,neutron and horizon) and also the supporting services 
such as MySql database, message broker (RabbitMQ), and NTP. 

Install the supporting services (Package, NTP, MySQL and RabbitMQ)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install the Ubuntu Cloud Archive for Icehouse::

    apt-get install python-software-properties
    add-apt-repository cloud-archive:icehouse

* Update and Upgrade your System::
   
    apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade

* Install NTP service (Network Time Protocol)::

    apt-get install -y ntp

* Install MySQL::

    apt-get install -y mysql-server python-mysqldb

* Set the bind-address key to the management IP address of the controller node::

    vi /etc/mysql/my.cnf
    bind-address = 10.0.1.21

* Under the [mysqld] section, set the following keys to enable InnoDB, UTF-8 character set, and UTF-8 collation by default::

    vi /etc/mysql/my.cnf
    [mysqld]
    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    init-connect = 'SET NAMES utf8'
    character-set-server = utf8

* Restart the MySQL service::

    service mysql restart

* Delete the anonymous users that are created when the database is first started::

    mysql_install_db
    mysql_secure_installation

* Install RabbitMQ (Message Queue) ::

    apt-get install -y rabbitmq-server
    rabbitmqctl change_password guest RABBIT_PASS


Install the Identity Service (Keystone)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install Identity Service

   * Install keystone packages::
   
       apt-get install -y keystone python-keystoneclient
   
   * Create a MySQL database for keystone::
   
       mysql -u root -p
   
       CREATE DATABASE keystone;
       GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
       GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
   
       exit;
   
   * Remove Keystone SQLite database::
   
       rm /var/lib/keystone/keystone.db
   
   * Edit /etc/keystone/keystone.conf::
   
        vi /etc/keystone/keystone.conf
     
       [database]
       # replace connection = sqlite:////var/lib/keystone/keystone.db by
       connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone
       
       [DEFAULT]
       admin_token=ADMIN_TOKEN 
       log_dir=/var/log/keystone
     
   
   * Restart the identity service then synchronize the database::
   
       service keystone restart
       keystone-manage db_sync
   
   * Check synchronization::
           
       mysql -u keystone -p 
       show databases;
       show TABLES;


* Define users, tenants, and roles

   * Create an administrative user::
   
       export OS_SERVICE_TOKEN=ADMIN_TOKEN 
       export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
       
       keystone user-create --name=admin --pass=admin_pass --email=admin@domain.com
       keystone role-create --name=admin
       keystone tenant-create --name=admin --description="Admin Tenant"
       keystone user-role-add --user=admin --tenant=admin --role=admin
       keystone user-role-add --user=admin --role=_member_ --tenant=admin
   
   * Create a normal user::
   
       keystone user-create --name=demo --pass=demo_pass --email=demo@domain.com
       keystone tenant-create --name=demo --description="Demo Tenant"
       keystone user-role-add --user=demo --role=_member_ --tenant=demo

   * Create a service tenant::
   
       keystone tenant-create --name=service --description="Service Tenant"
   

* Define services and API endpoints
   
   * Create a service entry for the Identity Service::
   
       keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
   
   * Specify an API endpoint for the Identity Service::
   
       keystone endpoint-create \
       --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
       --publicurl=http://pcontroller:5000/v2.0 \
       --internalurl=http://controller:5000/v2.0 \
       --adminurl=http://controller:35357/v2.0

* Verify the Identity Service installation
   
   * Create a simple credential file::

       vi admin_creds
       #Paste the following: 
       export OS_TENANT_NAME=admin
       export OS_USERNAME=admin
       export OS_PASSWORD=admin_pass
       export OS_AUTH_URL="http://pcontroller:5000/v2.0/"

       vi demo_creds
       #Paste the following: 
       export OS_USERNAME=demo
       export OS_PASSWORD=demo_pass
       export OS_TENANT_NAME=demo
       export OS_AUTH_URL=http://controller:35357/v2.0

   * clear the values in the OS_SERVICE_TOKEN and OS_SERVICE_ENDPOINT environment variables::
   
     unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT

   * Request a authentication token::
   
     keystone --os-username=admin --os-password=admin_pass --os-auth-url=http://controller:35357/v2.0 token-get

   * Load credential admin file::
   
     source admin_creds
     keystone token-get

   * Load credential file::
   
     source admin_creds
     keystone user-list
     keystone user-role-list --user admin --tenant admin


Install the Image Service (Glance)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Image Service Components::
    - glance-api: Accepts Image API calls for image discovery, retrieval, and storage.
    - glance-registry: Stores, processes, and retrieves metadata about images. Metadata includes items such as size and type
    - Database: Stores image metadata. You can choose your database depending on your preference.
    - Storage repository (for image files): The Image Service supports a variety of repositories including normal file systems, Object Storage, RADOS block devices, HTTP, and Amazon S3

* Install the Image Service

   * Install Glance packages::
   
       apt-get install -y glance python-glanceclient
   
   * Create a MySQL database for Glance::
   
       mysql -u root -p
       CREATE DATABASE glance;
       GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
       GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
       exit;
   
   * Configure service user and role::
   
       keystone user-create --name=glance --pass=service_pass --email=glance@domain.com
       keystone user-role-add --user=glance --tenant=service --role=admin
   
   * Register the service and create the endpoint::
   
       keystone service-create --name=glance --type=image --description="OpenStack Image Service"
       keystone endpoint-create \
       --service-id=$(keystone service-list | awk '/ image / {print $2}') \
       --publicurl=http://pcontroller:9292 \
       --internalurl=http://controller:9292 \
       --adminurl=http://controller:9292
   
   * Update /etc/glance/glance-api.conf::
   
       vi /etc/glance/glance-api.conf
       
       [database]
       # replace sqlite_db = /var/lib/glance/glance.sqlite with
       connection = mysql://glance:GLANCE_DBPASS@controller/glance
       
       [keystone_authtoken]
       auth_uri = http://controller:5000
       auth_host = controller
       auth_port = 35357
       auth_protocol = http
       admin_tenant_name = service
       admin_user = glance
       admin_password = service_pass
       
       [paste_deploy]
       flavor = keystone
   
   
   * Update /etc/glance/glance-registry.conf::
       
       vi /etc/glance/glance-registry.conf
       
       [database]
       # replace sqlite_db = /var/lib/glance/glance.sqlite with:
       connection = mysql://glance:GLANCE_DBPASS@controller/glance
       
       [keystone_authtoken]
       auth_uri = http://controller:5000
       auth_host = controller
       auth_port = 35357
       auth_protocol = http
       admin_tenant_name = service
       admin_user = glance
       admin_password = service_pass
       
       [paste_deploy]
       flavor = keystone
   
   * Remove sqlite database::
   
       rm /var/lib/glance/glance.sqlite
   
   * Create the database tables for the glance database::
   
       glance-manage db_sync

   * Restart the glance-api and glance-registry services::
   
       service glance-registry restart
       service glance-api restart; 
   
* Verify the Image Service installation

   * Test Glance, upload the cirros cloud image::

       source admin_creds
       glance image-create --name="cirros-0.3.2-x86_64" --disk-format=qcow2 \
       --container-format=bare --is-public=true \
       --copy-from http://download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img
 
   * List Images::

       glance image-list


Install the compute Service (Nova)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Compute service components:

  * API:
     - nova-api service. Accepts and responds to end user compute API calls. 
     - nova-api-metadata service. Accepts metadata requests from instances. 
  * Compute core:
     - nova-compute process. A worker daemon that creates and terminates virtual machine instances through hypervisor APIs.
     - nova-scheduler process. Conceptually the simplest piece of code in Compute. 
     - nova-conductor module. Mediates interactions between nova-compute and the database.
  * Networking for VMs:
     - nova-network worker daemon. Similar to nova-compute, it accepts networking tasks from the queue and performs tasks to manipulate the network, such as setting up bridging interfaces or changing iptables rules.
     - nova-dhcpbridge script. Tracks IP address leases and records them in the database by using the dnsmasq dhcp-script facility.
  * Console interface
     - nova-consoleauth daemon. Authorizes tokens for users that console proxies provide.
     - nova-novncproxy daemon. Provides a proxy for accessing running instances through a VNC connection. 
     - nova-xvpnvncproxy daemon. A proxy for accessing running instances through a VNC connection. 
     - nova-cert daemon. Manages x509 certificates.
  * Image management 
     - nova-objectstore daemon. Provides an S3 interface for registering images with the Image Service.
     - euca2ools client. A set of command-line interpreter commands for managing cloud resources.
  * Command-line clients and other interfaces
     - nova client. Enables users to submit commands as a tenant administrator or end user.
     - nova-manage client. Enables cloud administrators to submit commands.
  * Other components
     - The queue. A central hub for passing messages between daemons. Usually implemented with RabbitMQ
     - SQL database. Stores most build-time and runtime states for a cloud infrastructure.

* Install nova packages for the controller node::

    apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth \
    nova-novncproxy nova-scheduler python-novaclient


* Create a Mysql database for Nova::

    mysql -u root -p

    CREATE DATABASE nova;
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
    
    exit;

* Configure service user and role::

    keystone user-create --name=nova --pass=service_pass --email=nova@domain.com
    keystone user-role-add --user=nova --tenant=service --role=admin

* Register the service and create the endpoint::
    
    keystone service-create --name=nova --type=compute --description="OpenStack Compute"
    keystone endpoint-create \
    --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
    --publicurl=http://pcontroller:8774/v2/%\(tenant_id\)s \
    --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
    --adminurl=http://controller:8774/v2/%\(tenant_id\)s


* Edit the /etc/nova/nova.conf::
    
    vi /etc/nova/nova.conf

    [database]
    connection = mysql://nova:NOVA_DBPASS@controller/nova
    
    [DEFAULT]
    rpc_backend = rabbit
    rabbit_host = controller
    my_ip = 10.0.1.21
    vncserver_listen = 10.0.1.21
    vncserver_proxyclient_address = 10.0.1.21
    auth_strategy = keystone
    
    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = nova
    admin_password = service_pass


* Remove Nova SQLite database::

    rm /var/lib/nova/nova.sqlite


* Synchronize your database::

    nova-manage db sync

* Restart nova-* services::

    service nova-api restart
    service nova-cert restart
    service nova-conductor restart
    service nova-consoleauth restart
    service nova-novncproxy restart
    service nova-scheduler restart


* Check Nova is running. The :-) icons indicate that everything is ok !::
   
    nova-manage service list

* To verify your configuration, list available images::

    source admin_creds
    nova image-list
 
   
Install the network Service (Neutron)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install the Neutron server and the OpenVSwitch packages::

    apt-get install -y neutron-server neutron-plugin-ml2

* Create a MySql database for Neutron::

    mysql -u root -p
  
    CREATE DATABASE neutron;
    GRANT ALL PRIVILEGES ON neutron.* TO neutron@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
    GRANT ALL PRIVILEGES ON neutron.* TO neutron@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
    
    exit;

* Configure service user and role::

    keystone user-create --name=neutron --pass=service_pass --email=neutron@domain.com
    keystone user-role-add --user=neutron --tenant=service --role=admin

* Register the service and create the endpoint::

    keystone service-create --name=neutron --type=network --description="OpenStack Networking"
    
    keystone endpoint-create \
    --service-id=$(keystone service-list | awk '/ network / {print $2}') \
    --publicurl=http://pcontroller:9696 \
    --internalurl=http://controller:9696 \
    --adminurl=http://controller:9696 


* Update /etc/neutron/neutron.conf::
      
    vi /etc/neutron/neutron.conf
    
    [database]
    # replace connection = sqlite:////var/lib/neutron/neutron.sqlite with
    connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron
    
    [DEFAULT]
    # replace  core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin with
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    
    auth_strategy = keystone
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = controller
    
    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller:8774/v2
    nova_admin_username = nova
    # Replace the SERVICE_TENANT_ID with the output of this command (keystone tenant-list | awk '/ service / { print $2 }')
    nova_admin_tenant_id = SERVICE_TENANT_ID
    nova_admin_password = service_pass
    nova_admin_auth_url = http://controller:35357/v2.0
    
    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass


* Configure the Modular Layer 2 (ML2) plug-in::

    vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    
    [ml2_type_gre]
    tunnel_id_ranges = 1:1000
    
    [securitygroup]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_security_group = True


* Reconfigure Compute to use Networking::

    vi /etc/nova/nova.conf
    
    [DEFAULT]
    network_api_class=nova.network.neutronv2.api.API
    neutron_url=http://controller:9696
    neutron_auth_strategy=keystone
    neutron_admin_tenant_name=service
    neutron_admin_username=neutron
    neutron_admin_password=service_pass
    neutron_admin_auth_url=http://controller:35357/v2.0
    libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
    linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver=nova.virt.firewall.NoopFirewallDriver
    security_group_api=neutron

* Restart the Compute services::
    
    service nova-api restart
    service nova-scheduler restart
    service nova-conductor restart

* Restart the Networking service::

    service neutron-server restart


Install the dashboard Service (Horizon)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install the required packages::

    apt-get install -y apache2 memcached libapache2-mod-wsgi openstack-dashboard

* You can remove the openstack-dashboard-ubuntu-theme package::

    apt-get remove -y --purge openstack-dashboard-ubuntu-theme

* Edit /etc/openstack-dashboard/local_settings.py::
    
    vi /etc/openstack-dashboard/local_settings.py
    ALLOWED_HOSTS = ['localhost', 'pcontroller']
    OPENSTACK_HOST = "controller"

* Reload Apache and memcached::

    service apache2 restart 
    service memcached restart

* Note::

    If you have this error: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. 
    Set the 'ServerName' directive  globally to suppress this message”

    Solution: Edit /etc/apache2/apache2.conf

    vi /etc/apache2/apache2.conf
    Add the following new line end of file:
    ServerName localhost

* Reload Apache and memcached::

    service apache2 restart; service memcached restart


* Check OpenStack Dashboard at http://192.168.100.21/horizon. login admin/admin_pass


Install the block stroage Service (Cinder)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Block storage consists of the following three components:
    - cinder-api
    - cinder-scheduler
    - cinder-volume
  The first two services are installed on controller node and the last on the service node for storage.
  If the controller node has storage, the last service can also be installed on the controller.
  The document separtes the two nodes.
  
(1) Configure a Block Storage service controller

   * Install the cinder services::
   
       apt-get -y install cinder-api cinder-scheduler

   * Create a MySql database for Cinder::
   
       mysql -u root -p
       CREATE DATABASE cinder;
       GRANT ALL ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_PASS';
       GRANT ALL ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_PASS';
       exit;
       
       rm /var/lib/cinder/cinder.sqlite
       cinder-manage db sync
   
   * Register the service and create the endpoint::

       source admin_creds
       keystone user-create --name=cinder --pass=CINDER_PASS --email=cinder@email.com
       keystone user-role-add --user=cinder --tenant=service --role=admin
       keystone service-create --name=cinder --type=volume --description="OpenStack Block Storage"
       keystone endpoint-create \
         --service-id=$(keystone service-list | awk '/ volume / {print $2}') \
         --publicurl=http://pcontroller:8776/v1/%\(tenant_id\)s \
         --internalurl=http://controller:8776/v1/%\(tenant_id\)s \
         --adminurl=http://controller:8776/v1/%\(tenant_id\)s
         
       keystone service-create --name=cinderv2 --type=volumev2 --description="OpenStack Block Storage v2"
       keystone endpoint-create \
         --service-id=$(keystone service-list | awk '/ volumev2 / {print $2}') \
         --publicurl=http://pcontroller:8776/v2/%\(tenant_id\)s \
         --internalurl=http://controller:8776/v2/%\(tenant_id\)s \
         --adminurl=http://controller:8776/v2/%\(tenant_id\)s

   * Configure the cinder services::

       sudo vi /etc/cinder/cinder.conf
       [DEFAULT]
       rpc_backend = cinder.openstack.common.rpc.impl_kombu
       rabbit_host = controller
       rabbit_port = 5672
       …
       [keystone_authtoken]
       auth_uri = http://controller:5000
       auth_host = controller
       auth_port = 35357
       auth_protocol = http
       admin_tenant_name = service
       admin_user = cinder
       admin_password = CINDER_PASS
       [database]
       connection = mysql://cinder:CINDER_PASS@controller/cinder

   * Restart cinder services::
   
       service cinder-scheduler restart
       service cinder-api restart

(2) Configure a Block Storage service node

   * Configure the block storage (assumes a second disk /dev/sdb3 that is used for  LVM physical and logical volumes)::
   
       apt-get install lvm2
       pvcreate /dev/sdb3
       vgcreate cinder-volumes /dev/sdb3
       
       vi /etc/lvm/lvm.conf
       devices {
       ...
       filter = [ "a/sda1/", "a/sdb3/", "r/.*/"]
       ...
       }

   * Install the cinder serivces::
   
       apt-get install cinder-volume
       vi /etc/cinder/cinder.conf
       [keystone_authtoken]
       auth_uri = http://controller:5000
       auth_host = controller
       auth_port = 35357
       auth_protocol = http
       admin_tenant_name = service
       admin_user = cinder
       admin_password = CINDER_PASS
       ...
       [DEFAULT]
       rpc_backend = rabbit
       rabbit_host = controller
       rabbit_port = 5672
       ...
       my_ip = 10.0.1.21         
       glance_host = controller
       ...
       [database]
       connection = mysql://cinder:CINDER_DBPASS@controller/cinder

   * Restart cinder volume services::

       service cinder-volume restart  
       service tgt restart

(3) Verify the Block Storage installation

   * Test the cinder services::
   
       source demo_creds
       cinder create --display-name myVolume 1
       cinder list


Install Network moduels
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now, let's move to second step!

The network node runs the Networking plug-in and different agents (see the Figure below).


* Install other services::

    apt-get install -y vlan bridge-utils

* Edit /etc/sysctl.conf to contain the following::

    vi /etc/sysctl.conf
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0


* Implement the changes::

    sysctl -p

* Install the Networking components::

    apt-get install -y neutron-plugin-ml2 neutron-plugin-openvswitch-agent dnsmasq neutron-l3-agent neutron-dhcp-agent


* Edit the /etc/neutron/l3_agent.ini::

    vi /etc/neutron/l3_agent.ini
    
    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    use_namespaces = True

* Edit the /etc/neutron/dhcp_agent.ini::

    vi /etc/neutron/dhcp_agent.ini
    
    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = True
    # This is for resolving mtu problem. You can set jumo frame instread of setting this.
    # jumbo frame set: ifconfig eth0 mtu 9000
    dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
    
    vi /etc/neutron/dnsmasq-neutron.conf
    dhcp-option-force=26,1454


* Edit the /etc/neutron/metadata_agent.ini::

    vi /etc/neutron/metadata_agent.ini
    
    [DEFAULT]
    auth_url = http://controller:5000/v2.0
    auth_region = regionOne
    
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    nova_metadata_ip = controller
    metadata_proxy_shared_secret = helloOpenStack

* Note: On the controller node::
    vi /etc/nova/nova.conf

    [DEFAULT]
    service_neutron_metadata_proxy = true
    neutron_metadata_proxy_shared_secret = helloOpenStack
    
    service nova-api restart


* Edit the /etc/neutron/plugins/ml2/ml2_conf.ini::

    vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    
    [ml2_type_gre]
    tunnel_id_ranges = 1:1000
    
    [ovs]
    local_ip = 10.0.1.21
    tunnel_type = gre
    enable_tunneling = True
    
    [securitygroup]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_security_group = True

* Restart openVSwitch::

    service openvswitch-switch restart

* Create the bridges::

    #br-int will be used for VM integration
    ovs-vsctl add-br br-int

    #br-ex is used to make to VM accessible from the internet
    ovs-vsctl add-br br-ex


* Add the eth0 to the br-ex::

    #Internet connectivity will be lost after this step but this won't affect OpenStack's work
    ovs-vsctl add-port br-ex eth0

* Edit /etc/network/interfaces::

    vi /etc/network/interfaces
    #  comment out the following part and add the next part.
    # The management & public network interface
    #  auto eth0
    #  iface eth0 inet static
    #  address 192.168.100.21
    #  netmask 255.255.255.0
    #  gateway 192.168.100.1
    #  dns-nameservers 8.8.8.8
    
    # The public network interface
    auto eth0
    iface eth0 inet manual
    up ifconfig $IFACE 0.0.0.0 up
    up ip link set $IFACE promisc on
    down ip link set $IFACE promisc off
    down ifconfig $IFACE down
  
    auto br-ex
    iface br-ex inet static
    address 192.168.100.21
    netmask 255.255.255.0
    gateway 192.168.100.1
    dns-nameservers 8.8.8.8

* Restart network::

    ifdown eth0 && ifup eth0
    ifdown br-ex && ifup br-ex


* Restart all neutron services::

    service neutron-plugin-openvswitch-agent restart
    service neutron-dhcp-agent restart
    service neutron-l3-agent restart
    service neutron-metadata-agent restart
    service dnsmasq restart

* Check status::

    service neutron-plugin-openvswitch-agent status
    service neutron-dhcp-agent status
    service neutron-l3-agent status
    service neutron-metadata-agent status
    service dnsmasq status

* Create a simple credential file::

    vi admin_creds
    #Paste the following:
    export OS_TENANT_NAME=admin
    export OS_USERNAME=admin
    export OS_PASSWORD=admin_pass
    export OS_AUTH_URL="http://pcontroller:5000/v2.0/"

* Check Neutron agents::

    source admin_creds
    neutron agent-list



Compute Node
------------

Finally, let's install the services on the compute node!
It uses KVM as hypervisor and runs nova-compute, the Networking plug-in and layer 2 agent.  

Install Compute
^^^^^^^^^^^^^^^^

1) Install basic services

   * Install the Ubuntu Cloud Archive for Icehouse::
   
       apt-get install python-software-properties
       add-apt-repository cloud-archive:icehouse
   
   * Update and Upgrade your System::
    
       apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade
   
   
   * Install ntp service::
       
       apt-get install -y ntp
   
   * Set the compute node to follow up your conroller node::
   
      sed -i 's/server ntp.ubuntu.com/server controller/g' /etc/ntp.conf
   
   * Restart NTP service::
   
       service ntp restart
   
   * Install MySQL Python library::
   
       apt-get install python-mysqldb
      
   * Check that your hardware supports virtualization::
   
       apt-get install -y cpu-checker
       kvm-ok

   * Install and configure kvm::
   
       apt-get install -y kvm libvirt-bin pm-utils

2) Install Compute

   * Install the Compute packages::
   
       apt-get install -y nova-compute-kvm python-guestfs
   
   * Make the current kernel readable::
   
       dpkg-statoverride  --update --add root root 0644 /boot/vmlinuz-$(uname -r)
   
   * Enable this override for all future kernel updates, create the file /etc/kernel/postinst.d/statoverride containing::
   
       vi /etc/kernel/postinst.d/statoverride
       #!/bin/sh
       version="$1"
       # passing the kernel version is required
       [ -z "${version}" ] && exit 0
       dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-${version}
   
   * Make the file executable::
   
       chmod +x /etc/kernel/postinst.d/statoverride
   
   
   * Modify the /etc/nova/nova.conf like this::
   
       vi /etc/nova/nova.conf
       [DEFAULT]
       auth_strategy = keystone
       vif_plugging_is_fatal=false
       vif_plugging_timeout=0
   
       rpc_backend = rabbit
       rabbit_host = controller
       glance_host = controller
   
       my_ip = 10.0.1.31
       vnc_enabled = True
       vncserver_listen = 0.0.0.0
       vncserver_proxyclient_address = 10.0.1.31
       novncproxy_base_url = http://pcontroller:6080/vnc_auto.html
       
       [database]
       connection = mysql://nova:NOVA_DBPASS@controller/nova
       
       [keystone_authtoken]
       auth_uri = http://controller:5000
       auth_host = controller
       auth_port = 35357
       auth_protocol = http
       admin_tenant_name = service
       admin_user = nova
       admin_password = service_pass
       
   * Notice that if this command (egrep -c '(vmx|svm)' /proc/cpuinfo) returns a value of zero, 
     your system does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM in nova.conf.:: 
       [libvirt]
       ...
       virt_type = qemu
   
   * Delete /var/lib/nova/nova.sqlite file::
       
       rm /var/lib/nova/nova.sqlite
   
   * Restart nova-compute services::
   
       service nova-compute restart


Install Network
^^^^^^^^^^^^^^^^

* Edit /etc/sysctl.conf to contain the following::

    vi /etc/sysctl.conf
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0

* Implement the changes::

    sysctl -p

* Install the Networking components::
    
    apt-get install -y neutron-common neutron-plugin-ml2 neutron-plugin-openvswitch-agent


* Update /etc/neutron/neutron.conf::

    vi /etc/neutron/neutron.conf
    
    [DEFAULT]
    auth_strategy = keystone
    # replace  core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin with
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    
    rpc_backend = neutron.openstack.common.rpc.impl_kombu
    rabbit_host = controller
    
    [keystone_authtoken]
    auth_uri = http://controller:5000
    auth_host = controller
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    
   [database]
   connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron


* Configure the Modular Layer 2 (ML2) plug-in::
    
    vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    
    [ml2_type_gre]
    tunnel_id_ranges = 1:1000
    
    [ovs]
    local_ip = 10.0.1.31
    tunnel_type = gre
    enable_tunneling = True
    
    [securitygroup]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_security_group = True

* Restart the OVS service::

    service openvswitch-switch restart

* Create the bridges::

    #br-int will be used for VM integration
    ovs-vsctl add-br br-int
    

* Edit /etc/nova/nova.conf::

    vi /etc/nova/nova.conf
    
    [DEFAULT]
    network_api_class = nova.network.neutronv2.api.API
    neutron_url = http://controller:9696
    neutron_auth_strategy = keystone
    neutron_admin_tenant_name = service
    neutron_admin_username = neutron
    neutron_admin_password = service_pass
    neutron_admin_auth_url = http://controller:35357/v2.0
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    security_group_api = neutron


* Restart nova-compute services::

    service nova-compute restart

* Restart the Open vSwitch (OVS) agent::

    service neutron-plugin-openvswitch-agent restart

* Check Nova is running. The :-) icons indicate that everything is ok !::

    nova-manage service list
    


Trouble Shooting on Operation 
------------
* Case1: 'Too many connections'
    Problem
    - cat /var/log/nova/nova-api.log
    ERROR nova.api.openstack [req-22e47296-ae37-4a66-9d6f-953c21efb8c2 b5b2b5c7c2a740599857cf31cc3b43e3 27d4039052fd42b09ee477e0b40fe713] Caught error: (OperationalError) (1040, 'Too many connections') None None

    Solution
    - /etc/mysql/my.cnf
      max_connections        = 500
    - service mysql restart
    
    
    
Reference
=========
The content is the summarization of OpenStack Installation Guide for Ubuntu 14.04 (LTS) and the applicaiton case to two nodes.
- http://docs.openstack.org/icehouse/install-guide/install/apt/content/
- https://fosskb.wordpress.com/2014/06/10/managing-openstack-internaldataexternal-network-in-one-interface/

Contacts
========

GunSik Choi : cgshome at gmail.com

