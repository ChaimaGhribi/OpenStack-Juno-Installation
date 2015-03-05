####
OpenStack Juno Installation - Multi Node
####

Welcome to OpenStack Juno installation manual !

This document is based on `the OpenStack Official Documentation <http://docs.openstack.org/juno/install-guide/install/apt/content/index.html>`_ for Juno. 

===============================

**Contributors:**

Chaima Ghribi: chaima.ghribi@it-sudparis.eu

Marouen Mechtri : marouen.mechtri@it-sudparis.eu

Djamal Zeghlache : djamal.zeghlache@it-sudparis.eu

==========================================

.. contents::
   

Basic Architecture and Network Configuration
==========================================

In this installation guide, we cover the step-by-step process of installing Openstack Juno on Ubuntu 14.04.  We consider a multi-node architecture with Openstack Networking (Neutron) that requires three node types: 

+ **Controller Node** that runs management services (keystone, Horizon…) needed for OpenStack to function.

+ **Network Node** that runs networking services and is responsible for virtual network provisioning  and for connecting virtual machines to external networks.

+ **Compute Node** that runs the virtual machine instances in OpenStack. 

We have deployed a single compute node but you can simply add more compute nodes to our multi-node installation, if needed.  



.. image:: https://raw.githubusercontent.com/ChaimaGhribi/OpenStack-Juno-Installation/master/images/network-topo.jpg

For OpenStack Multi-Node setup you need to create three networks:

+ **Management Network** (10.0.0.0/24): A network segment used for administration, not accessible to the public Internet.

+ **VM Traffic Network** (10.0.1.0/24): This network is used as internal network for traffic between virtual machines in OpenStack, and between the virtual machines and the network nodes that provide L3 routes out to the public network.

+ **Public Network**: This network is connected to the controller node so users can access the OpenStack interfaces, and connected to the network node to provide VMs with publicly routable traffic functionality.


In the next subsections, we describe in details how to set up, configure and test the network architecture. We want to make sure everything is ok before install ;)

So, let’s prepare the nodes for OpenStack installation!

Configure Controller node
-------------------------

The controller node has two Network Interfaces: eth0 (used for management network) and eth1 is external.

* Change to super user mode::

    sudo su

* Set the hostname::

    vi /etc/hostname
    controller


* Edit /etc/hosts::

    vi /etc/hosts
        
    #controller
    10.0.0.11       controller
        
    #network
    10.0.0.21       network
        
    # compute1  
    10.0.0.31       compute1
    
* Note::

    Remove or comment the line beginning with 127.0.1.1.

* Edit network settings to configure the interfaces eth0 and eth1::

    vi /etc/network/interfaces
      
    # The management network interface
    auto eth0
    iface eth0 inet static
        address 10.0.0.11
        netmask 255.255.255.0
        network 10.0.0.0

    # The public network interface
    auto eth1
    iface eth1 inet static
        address 192.168.100.11
        netmask 255.255.255.0
        network 192.168.100.0
        gateway 192.168.100.1
        dns-nameservers 8.8.8.8 8.8.4.4

* Restart network and if needed **reboot** the system to activate the changes::

    ifdown eth0 && ifup eth0
    
    ifdown eth1 && ifup eth1
    
Configure Network node
----------------------

The network node has three network Interfaces: eth0 for management use: eth1
for connectivity between VMs and eth2 for external connectivity.

* Change to super user mode::

    sudo su

* Set the hostname::

    vi /etc/hostname
    network


* Edit /etc/hosts::

    vi /etc/hosts

    #network
    10.0.0.21       network
    
    #controller
    10.0.0.11       controller
      
    # compute1   
    10.0.0.31       compute1

* Note::

    Remove or comment the line beginning with 127.0.1.1.

* Edit network settings to configure the interfaces eth0, eth1 and eth2::

    vi /etc/network/interfaces

    # The management network interface
    auto eth0
    iface eth0 inet static
        address 10.0.0.21
        netmask 255.255.255.0
        network 10.0.0.0

    
    # VM traffic interface
    auto eth1
    iface eth1 inet static
        address 10.0.1.21
        netmask 255.255.255.0
        network 10.0.1.0
    
    # The public network interface
    auto eth2
    iface eth2 inet static
        address 192.168.100.21
        netmask 255.255.255.0
        network 192.168.100.0
        gateway 192.168.100.1
        dns-nameservers 8.8.8.8 8.8.4.4



* Restart network and if needed **reboot** the system to activate the changes::

    ifdown eth0 && ifup eth0
    
    ifdown eth1 && ifup eth1
    
    ifdown eth2 && ifup eth2


Configure Compute node
----------------------

The network node has two network Interfaces: eth0 for management use and 
eth1 for connectivity between VMs.


* Change to super user mode::

    sudo su

* Set the hostname::

    vi /etc/hostname
    compute1


* Edit /etc/hosts::

    vi /etc/hosts
    
    # compute1
    10.0.0.31       compute1
  
    #controller
    10.0.0.11       controller
  
    #network
    10.0.0.21       network
    
* Note::

    Remove or comment the line beginning with 127.0.1.1.

* Edit network settings to configure the interfaces eth0 and eth1::

    vi /etc/network/interfaces
  
    # The management network interface    
    auto eth0
    iface eth0 inet static
        address 10.0.0.31
        netmask 255.255.255.0
        network 10.0.0.0
  
    # VM traffic interface     
    auto eth1
    iface eth1 inet static
        address 10.0.1.31
        netmask 255.255.255.0
        network 10.0.1.0

* In order to be able to reach public network for installing openstack packages, we recommend to add a new interface (e.g. eth2) connected to the public network or configure nat Service on the management network.


* Restart network and if needed **reboot** the system to activate the changes::
  
    ifdown eth0 && ifup eth0
      
    ifdown eth1 && ifup eth1


Verify connectivity
-------------------

We recommend that you verify network connectivity to the internet and among the nodes before proceeding further.

    
* From the controller node::

    # ping a site on the internet:
    ping openstack.org

    # ping the management interface on the network node:
    ping network

    # ping the management interface on the compute node:
    ping compute1

* From the network node::

    # ping a site on the internet:
    ping openstack.org

    # ping the management interface on the controller node:
    ping controller

    # ping the VM traffic interface on the compute node:
    ping 10.0.1.31
    
* From the compute node::

    # ping a site on the internet:
    ping openstack.org

    # ping the management interface on the controller node:
    ping controller

    # ping the VM traffic interface on the network node:
    ping 10.0.1.21

Install 
=======

Now everything is ok :) So let's go ahead and install it !


Controller Node
---------------

Let's start with the controller ! the cornerstone !

Here we've installed the basic services (keystone, glance, nova,neutron and horizon) and also the supporting services 
such as MySql database, message broker (RabbitMQ), and NTP. 

Additional installation guides for optional services (Heat, Ceilometer...) are also provided
`Heat installation guide <https://github.com/MarouenMechtri/Heat-Installation-for-OpenStack-Juno>`_, 
`Ceilometer installation guide <https://github.com/MarouenMechtri/Ceilometer-Installation-for-OpenStack-Juno>`_.

.. image:: https://raw.githubusercontent.com/ChaimaGhribi/OpenStack-Juno-Installation/master/images/controller.jpg
	
Install the supporting services (NTP, MySQL and RabbitMQ)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install NTP (Network Time Protocol) service::
   
    apt-get install -y ntp

* Edit the /etc/ntp.conf file::

    server 0.ubuntu.pool.ntp.org iburst
    server 1.ubuntu.pool.ntp.org iburst
    server 2.ubuntu.pool.ntp.org iburst
    server 3.ubuntu.pool.ntp.org iburst

    # Use Ubuntu's ntp server as a fallback.
    server ntp.ubuntu.com iburst

    restrict -4 default kod notrap nomodify 
    restrict -6 default kod notrap nomodify

* Restart the NTP service::

    service ntp restart

* Enable the OpenStack repository::

    apt-get install -y ubuntu-cloud-keyring
    echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu" \
    "trusty-updates/juno main" > /etc/apt/sources.list.d/cloudarchive-juno.list

* Upgrade the packages on your system::

    apt-get update -y && apt-get dist-upgrade -y

* Install MySQL::

    apt-get install -y mariadb-server python-mysqldb

* Edit /etc/mysql/my.cnf file::

    vi /etc/mysql/my.cnf
    [mysqld]
    bind-address = 10.0.0.11
    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    init-connect = 'SET NAMES utf8'
    character-set-server = utf8

* Restart the MySQL service::

    service mysql restart

* Secure the database service::

    mysql_secure_installation

* Install RabbitMQ (Message Queue)::

   apt-get install -y rabbitmq-server

* Change the password::
   
   rabbitmqctl change_password guest service_pass


Install the Identity Service (Keystone)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Create a MySQL database for keystone::

    mysql -u root -p

    CREATE DATABASE keystone;
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';

    exit;

* Install keystone packages::

    apt-get install -y keystone python-keystoneclient

    
* Remove Keystone SQLite database::

    rm -f /var/lib/keystone/keystone.db

* Edit /etc/keystone/keystone.conf::

     vi /etc/keystone/keystone.conf
  
    [database]
    replace connection = sqlite:////var/lib/keystone/keystone.db by
    connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone
    
    [DEFAULT]
    admin_token = ADMIN
    verbose = True

    [token]
    provider = keystone.token.providers.uuid.Provider
    driver = keystone.token.persistence.backends.sql.Token
    
    [revoke]
    driver = keystone.contrib.revoke.backends.sql.Revoke


* Populate the Identity service database::

    su -s /bin/sh -c "keystone-manage db_sync" keystone

* Restart the Identity service::

    service keystone restart

* We recommend that you use cron to configure a periodic task that purges expired tokens hourly::

    (crontab -l -u keystone 2>&1 | grep -q token_flush) || \
    echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' \
    >> /var/spool/cron/crontabs/keystone

* Check synchronization::
        
    mysql -u root -p keystone
    show TABLES;


* Define users, tenants, and roles::

    export OS_SERVICE_TOKEN=ADMIN
    export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
    
    #Create an administrative user
    keystone tenant-create --name admin --description "Admin Tenant"
    keystone user-create --name admin --pass admin_pass --email admin@domain.com
    keystone role-create --name admin
    keystone user-role-add --user admin --tenant admin --role admin

    
    #Create a normal user
    keystone tenant-create --name demo --description "Demo Tenant"
    keystone user-create --name demo --tenant demo --pass demo_pass --email demo@domain.com

    
    #Create a service tenant
    keystone tenant-create --name service --description "Service Tenant"


* Define services and API endpoints::
    
    keystone service-create --name keystone --type identity --description "OpenStack Identity"

    keystone endpoint-create \
    --service-id $(keystone service-list | awk '/ identity / {print $2}') \
    --publicurl http://controller:5000/v2.0 \
    --internalurl http://controller:5000/v2.0 \
    --adminurl http://controller:35357/v2.0 \
    --region regionOne

     
* Test Keystone::
    
    #clear the values in the OS_SERVICE_TOKEN and OS_SERVICE_ENDPOINT environment variables:        
    unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT

    #Request an authentication token:
    keystone --os-tenant-name admin --os-username admin --os-password admin_pass \
    --os-auth-url http://controller:35357/v2.0 token-get

    #List tenants: 
    keystone --os-tenant-name admin --os-username admin --os-password admin_pass \
    --os-auth-url http://controller:35357/v2.0 tenant-list
    
    #List users:
    keystone --os-tenant-name admin --os-username admin --os-password admin_pass \
    --os-auth-url http://controller:35357/v2.0 user-list
     
    #List roles:
    keystone --os-tenant-name admin --os-username admin --os-password admin_pass \
    --os-auth-url http://controller:35357/v2.0 role-list
     
    #Request an authentication token:
    keystone --os-tenant-name demo --os-username demo --os-password demo_pass \
    --os-auth-url http://controller:35357/v2.0 token-get 

    #Attempt to list users to verify that demo tenant cannot list users:
    keystone --os-tenant-name demo --os-username demo --os-password demo_pass \
    --os-auth-url http://controller:35357/v2.0 user-list
    You are not authorized to perform the requested action: admin_required (HTTP 403)

* Create a simple credential file::
        
    vi admin_creds
    #Paste the following:
    export OS_TENANT_NAME=admin
    export OS_USERNAME=admin
    export OS_PASSWORD=admin_pass
    export OS_AUTH_URL=http://controller:35357/v2.0

    vi demo_creds
    #Paste the following:
    export OS_TENANT_NAME=demo
    export OS_USERNAME=demo
    export OS_PASSWORD=demo_pass
    export OS_AUTH_URL=http://controller:5000/v2.0

* To load client environment scripts::

     source admin_creds  
     
Install the image Service (Glance)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


* Create a MySQL database for Glance::

    mysql -u root -p

    CREATE DATABASE glance;
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
    
    exit;

* Configure service user and role::

    source admin_creds

    keystone user-create --name glance --pass service_pass
    keystone user-role-add --user glance --tenant service --role admin

* Register the service and create the endpoint::

    keystone service-create --name glance --type image --description "OpenStack Image Service"
    keystone endpoint-create \
    --service-id $(keystone service-list | awk '/ image / {print $2}') \
    --publicurl http://controller:9292 \
    --internalurl http://controller:9292 \
    --adminurl http://controller:9292 \
    --region regionOne

* Install Glance packages::

    apt-get install -y glance python-glanceclient

* Update /etc/glance/glance-api.conf::

    vi /etc/glance/glance-api.conf
    
    [database]
    replace sqlite_db = /var/lib/glance/glance.sqlite with
    connection = mysql://glance:GLANCE_DBPASS@controller/glance
    
    [DEFAULT]
    notification_driver = noop
    verbose = True
    
    [keystone_authtoken]
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = glance
    admin_password = service_pass
    
    [paste_deploy]
    flavor = keystone
  
    [glance_store]
    default_store = file
    filesystem_store_datadir = /var/lib/glance/images/


* Update /etc/glance/glance-registry.conf::
    
    vi /etc/glance/glance-registry.conf
    
    [database]
    replace sqlite_db = /var/lib/glance/glance.sqlite with:
    connection = mysql://glance:GLANCE_DBPASS@controller/glance
    
    [DEFAULT]
    notification_driver = noop
    verbose = True

    [keystone_authtoken]
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = glance
    admin_password = service_pass

    
    [paste_deploy]
    flavor = keystone

* Populate the Image Service database::

    su -s /bin/sh -c "glance-manage db_sync" glance

* Restart the Image Service services::

    service glance-registry restart
    service glance-api restart

* Test Glance, upload the cirros cloud image::

    source admin_creds
    mkdir /tmp/images

    wget -P /tmp/images http://cdn.download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img

    glance image-create --name "cirros-0.3.3-x86_64" --file /tmp/images/cirros-0.3.3-x86_64-disk.img \
    --disk-format qcow2 --container-format bare --is-public True --progress

    rm -r /tmp/images
    
* List Images::

    glance image-list


Install the compute Service (Nova)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Create a Mysql database for Nova::

    mysql -u root -p

    CREATE DATABASE nova;
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
    
    exit;

* Configure service user and role::

    source admin_creds
    keystone user-create --name nova --pass service_pass
    keystone user-role-add --user nova --tenant service --role admin
    
    
* Register the service and create the endpoint::    
    
    keystone service-create --name nova --type compute --description "OpenStack Compute"
    keystone endpoint-create \
    --service-id $(keystone service-list | awk '/ compute / {print $2}') \
    --publicurl http://controller:8774/v2/%\(tenant_id\)s \
    --internalurl http://controller:8774/v2/%\(tenant_id\)s \
    --adminurl http://controller:8774/v2/%\(tenant_id\)s \
    --region regionOne
  
* Install nova packages::

    apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth \
    nova-novncproxy nova-scheduler python-novaclient
    
* Edit the /etc/nova/nova.conf::
    
    vi /etc/nova/nova.conf

    [database]
    connection = mysql://nova:NOVA_DBPASS@controller/nova
    
    [DEFAULT]
    rpc_backend = rabbit
    rabbit_host = controller
    rabbit_password = service_pass
    auth_strategy = keystone
    my_ip = 10.0.0.11
    vncserver_listen = 10.0.0.11
    vncserver_proxyclient_address = 10.0.0.11
    verbose = True
    
    
    [keystone_authtoken]
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = nova
    admin_password = service_pass

    [glance]
    host = controller

* Populate the Compute database::

   su -s /bin/sh -c "nova-manage db sync" nova

* Restart nova-* services::

    service nova-api restart
    service nova-cert restart
    service nova-consoleauth restart
    service nova-scheduler restart
    service nova-conductor restart
    service nova-novncproxy restart

* Remove the SQLite database file::

    rm -f /var/lib/nova/nova.sqlite

* Check Nova is running ::
    
    source admin_creds
    nova service-list

* To verify your configuration, list available images::

    source admin_creds
    nova image-list
    
Install the network Service (Neutron)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Create a MySql database for Neutron::

    mysql -u root -p
  
    CREATE DATABASE neutron;
    GRANT ALL PRIVILEGES ON neutron.* TO neutron@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
    GRANT ALL PRIVILEGES ON neutron.* TO neutron@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
    
    exit;

* Configure service user and role::

    source admin_creds
    keystone user-create --name neutron --pass service_pass
    keystone user-role-add --user neutron --tenant service --role admin
    
* Register the service and create the endpoint::

    keystone service-create --name neutron --type network --description "OpenStack Networking"
   
    keystone endpoint-create \
    --service-id $(keystone service-list | awk '/ network / {print $2}') \
    --publicurl http://controller:9696 \
    --adminurl http://controller:9696 \
    --internalurl http://controller:9696 \
    --region regionOne

* Install the Networking components::

    apt-get install -y neutron-server neutron-plugin-ml2 python-neutronclient
    
* Update /etc/neutron/neutron.conf::
      
    vi /etc/neutron/neutron.conf
    
    [database]
    replace connection = sqlite:////var/lib/neutron/neutron.sqlite with
    connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron
    
    [DEFAULT]
    rpc_backend = rabbit
    rabbit_host = controller
    rabbit_password = service_pass
    
    auth_strategy = keystone

    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True

    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller:8774/v2
    nova_admin_auth_url = http://controller:35357/v2.0
    nova_region_name = regionOne
    nova_admin_username = nova
    # Replace the SERVICE_TENANT_ID with the output of this command 
    # (keystone tenant-list | awk '/ service / { print $2 }')
    nova_admin_tenant_id = SERVICE_TENANT_ID
    nova_admin_password = service_pass
    
    verbose = True    
     
    [keystone_authtoken]
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass


* Configure the Modular Layer 2 (ML2) plug-in::

    vi /etc/neutron/plugins/ml2/ml2_conf.ini   
    
    [ml2]
    type_drivers = flat,gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    
    [ml2_type_gre]
    tunnel_id_ranges = 1:1000 
    
    [securitygroup]    
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* Configure Compute to use Networking::

    add in /etc/nova/nova.conf
        
    vi /etc/nova/nova.conf
    
    [DEFAULT]
    network_api_class = nova.network.neutronv2.api.API
    security_group_api = neutron
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver = nova.virt.firewall.NoopFirewallDriver

    [neutron]
    url = http://controller:9696
    auth_strategy = keystone
    admin_auth_url = http://controller:35357/v2.0
    admin_tenant_name = service
    admin_username = neutron
    admin_password = service_pass
    
* Populate the database::

    su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade juno" neutron
    

* Restart the Compute services::
    
    service nova-api restart
    service nova-scheduler restart
    service nova-conductor restart

* Restart the Networking service::

    service neutron-server restart


Install the dashboard Service (Horizon)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Install the required packages::

    apt-get install -y openstack-dashboard apache2 libapache2-mod-wsgi memcached python-memcache

* You can remove the openstack-dashboard-ubuntu-theme package::

    apt-get remove -y --purge openstack-dashboard-ubuntu-theme

* Edit /etc/openstack-dashboard/local_settings.py::
    
    vi /etc/openstack-dashboard/local_settings.py
    
    OPENSTACK_HOST = "controller"
    ALLOWED_HOSTS = ['*']
    
    CACHES = {
               'default': { 'BACKEND': 'django.core.cache.backends.memcached.
                             MemcachedCache', 'LOCATION': '127.0.0.1:11211',
                          }
    }
    
    
* Restart the web server and session storage service::

    service apache2 restart
    service memcached restart


* Check OpenStack Dashboard at http://controller/horizon  login admin/admin_pass


Network Node
------------

Now, let's move to second step!

The network node runs the Networking plug-in and different agents (see the Figure below).


.. image:: https://raw.githubusercontent.com/ChaimaGhribi/OpenStack-Juno-Installation/master/images/network.jpg
     	 :align: center

* Install NTP service::
   
   apt-get install -y ntp
   
* Edit the /etc/ntp.conf file::

   vi /etc/ntp.conf
   
   [replace]
   server 0.ubuntu.pool.ntp.org
   server 1.ubuntu.pool.ntp.org
   server 2.ubuntu.pool.ntp.org
   server 3.ubuntu.pool.ntp.org

   # Use Ubuntu's ntp server as a fallback.
   server ntp.ubuntu.com

   [with]
   server controller iburst


* Restart NTP service::

    service ntp restart
    
* Enable the OpenStack repository::

    apt-get install -y ubuntu-cloud-keyring
    echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu" \
    "trusty-updates/juno main" > /etc/apt/sources.list.d/cloudarchive-juno.list

 
* Upgrade the packages on your system::

    apt-get update -y && apt-get dist-upgrade -y   
    

* Edit /etc/sysctl.conf to contain the following::

    vi /etc/sysctl.conf
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0

* Implement the changes::

    sysctl -p

* Install the Networking components::

    apt-get install -y neutron-plugin-ml2 neutron-plugin-openvswitch-agent \
    neutron-l3-agent neutron-dhcp-agent

* Update /etc/neutron/neutron.conf::

    vi /etc/neutron/neutron.conf

    [DEFAULT]
    rpc_backend = rabbit
    rabbit_host = controller
    rabbit_password = service_pass
    
    auth_strategy = keystone
    
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    
    verbose = True
        
    [keystone_authtoken]
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    
    **[comment out this line]**
    connection = sqlite:////var/lib/neutron/neutron.sqlite

* Edit the /etc/neutron/plugins/ml2/ml2_conf.ini::

    vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = flat,gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    
    [ml2_type_flat]
    flat_networks = external
    
    [ml2_type_gre]
    tunnel_id_ranges = 1:1000
    
    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    
    [ovs]
    local_ip = 10.0.1.21
    enable_tunneling = True
    bridge_mappings = external:br-ex
   
    [agent]
    tunnel_types = gre

* Edit the /etc/neutron/l3_agent.ini::

    vi /etc/neutron/l3_agent.ini
    
    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    use_namespaces = True
    external_network_bridge = br-ex
    router_delete_namespaces = True
    
    verbose = True

* Edit the /etc/neutron/dhcp_agent.ini::

    vi /etc/neutron/dhcp_agent.ini
    
    [DEFAULT]
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = True
    dhcp_delete_namespaces = True
    
    verbose = True 
    
    dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
    
* Edit the /etc/neutron/dnsmasq-neutron.conf file::

     vi /etc/neutron/dnsmasq-neutron.conf
     
     dhcp-option-force=26,1454

* Kill any existing dnsmasq processes::
     
     pkill dnsmasq   
    
    
* Edit the /etc/neutron/metadata_agent.ini::

    vi /etc/neutron/metadata_agent.ini
    
    [DEFAULT]
    auth_url = http://controller:5000/v2.0
    auth_region = regionOne
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass
    
    nova_metadata_ip = controller
    
    metadata_proxy_shared_secret = METADATA_SECRET
    
    verbose = True

* Note: On the **controller node**, edit the /etc/nova/nova.conf file::
    
    vi /etc/nova/nova.conf

    [DEFAULT]
    service_metadata_proxy = True
    metadata_proxy_shared_secret = METADATA_SECRET
    
* Note: On the **controller node**, restart nova-api service::

    service nova-api restart


* Restart openVSwitch::

    service openvswitch-switch restart

* Add the external bridge::

    ovs-vsctl add-br br-ex


* Add the eth2 to the br-ex::

    ovs-vsctl add-port br-ex eth2

* Edit /etc/network/interfaces::

    vi /etc/network/interfaces
    
    # The public network interface
    auto eth2
    iface eth2 inet manual
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down
  
    auto br-ex
    iface br-ex inet static
        address 192.168.100.21
        netmask 255.255.255.0
        network 192.168.100.0
        gateway 192.168.100.1
        dns-nameservers 8.8.8.8 8.8.4.4

* Restart network::

    ifdown eth2 && ifup eth2

    ifdown br-ex && ifup br-ex
    
    
* Restart the Networking services::

    service neutron-plugin-openvswitch-agent restart
    service neutron-l3-agent restart
    service neutron-dhcp-agent restart
    service neutron-metadata-agent restart

* On the **controller node**, perform these commands to check Neutron agents::

    source admin_creds
    neutron agent-list


Compute Node
------------

Finally, let's install the services on the compute node!

It uses KVM as hypervisor and runs nova-compute, the Networking plug-in and layer 2 agent.  

.. image:: https://raw.githubusercontent.com/ChaimaGhribi/OpenStack-Juno-Installation/master/images/compute.jpg
		:align: center


* Install ntp service::
    
    apt-get install -y ntp

* Set the compute node to follow up your conroller node::

    vi /etc/ntp.conf
    
    [replace]
    server 0.ubuntu.pool.ntp.org
    server 1.ubuntu.pool.ntp.org
    server 2.ubuntu.pool.ntp.org
    server 3.ubuntu.pool.ntp.org
    
    # Use Ubuntu's ntp server as a fallback.
    server ntp.ubuntu.com
    
    [with]
    server controller iburst
    
* Restart NTP service::

    service ntp restart
    
* Enable the OpenStack repository::

    apt-get install -y ubuntu-cloud-keyring
    echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu" \
    "trusty-updates/juno main" > /etc/apt/sources.list.d/cloudarchive-juno.list

* Upgrade the packages on your system::

    apt-get update -y && apt-get dist-upgrade -y


* Install the Compute packages::
    
    apt-get install -y nova-compute sysfsutils

* Modify the /etc/nova/nova.conf like this::

    vi /etc/nova/nova.conf
    [DEFAULT]
    verbose = True
    
    auth_strategy = keystone
    
    rpc_backend = rabbit
    rabbit_host = controller
    rabbit_password = service_pass
    
    my_ip = 10.0.0.31
    
    vnc_enabled = True
    vncserver_listen = 0.0.0.0
    vncserver_proxyclient_address = 10.0.0.31
    novncproxy_base_url = http://192.168.100.11:6080/vnc_auto.html
    
    network_api_class = nova.network.neutronv2.api.API
    security_group_api = neutron
    linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    
    [keystone_authtoken]
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = nova
    admin_password = service_pass
    
    [glance]
    host = controller
    
    [neutron]
    url = http://controller:9696
    auth_strategy = keystone
    admin_auth_url = http://controller:35357/v2.0
    admin_tenant_name = service
    admin_username = neutron
    admin_password = service_pass

* Edit /etc/nova/nova-compute.conf with the correct hypervisor type (set to qemu if using virtualbox for example, kvm is default)::

    vi /etc/nova/nova-compute.conf
    
    [DEFAULT]
    compute_driver=libvirt.LibvirtDriver
    
    [libvirt]
    virt_type=qemu


* Delete /var/lib/nova/nova.sqlite file::
    
    rm -f /var/lib/nova/nova.sqlite
    
    
* Edit /etc/sysctl.conf to contain the following::

    vi /etc/sysctl.conf
    net.ipv4.conf.all.rp_filter=0
    net.ipv4.conf.default.rp_filter=0

* Implement the changes::

    sysctl -p

* Install the Networking components::
    
    apt-get install -y neutron-plugin-ml2 neutron-plugin-openvswitch-agent


* Update /etc/neutron/neutron.conf::

    vi /etc/neutron/neutron.conf
    
    [DEFAULT]
    verbose = True
    
    rpc_backend = rabbit
    rabbit_host = controller
    rabbit_password = service_pass
    
    auth_strategy = keystone
    
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    
    [keystone_authtoken]
    auth_uri = http://controller:5000/v2.0
    identity_uri = http://controller:35357
    admin_tenant_name = service
    admin_user = neutron
    admin_password = service_pass

    **[remove]**
    connection = sqlite:////var/lib/neutron/neutron.sqlite


* Configure the Modular Layer 2 (ML2) plug-in::
    
    vi /etc/neutron/plugins/ml2/ml2_conf.ini
    
    [ml2]
    type_drivers = flat,gre
    tenant_network_types = gre
    mechanism_drivers = openvswitch
    
    [ml2_type_gre]
    tunnel_id_ranges = 1:1000
    
    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [ovs]
    local_ip = 10.0.1.31
    enable_tunneling = True
    
    [agent]
    tunnel_types = gre
    
    
* Restart the OVS service::

    service openvswitch-switch restart
    
* Restart nova-compute services::

    service nova-compute restart
   
* Restart the Open vSwitch (OVS) agent::

    service neutron-plugin-openvswitch-agent restart

* On the **controller node**, list agents to verify successful launch of the neutron agents:: 

    source admin_creds
    neutron agent-list
    nova service-list


That's it !! ;) Just try it!

If you want to create your first instance with Neutron, follow the instructions in our VM creation guide available
here `Create-First-Instance-with-Neutron <https://github.com/ChaimaGhribi/OpenStack-Juno-Installation/blob/master/Create-your-first-instance-with-Neutron.rst>`_.  

Your contributions are welcome, as are questions and requests for help :)

Hope this manual will be helpful and simple!


License
=======
Institut Mines Télécom - Télécom SudParis  

Copyright (C) 2014  Authors

Original Authors - Chaima Ghribi and Marouen Mechtri

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except 

in compliance with the License. You may obtain a copy of the License at::

    http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.


Authors
==========

Copyright (C) Chaima Ghribi: chaima.ghribi@it-sudparis.eu

Copyright (C) Marouen Mechtri : marouen.mechtri@it-sudparis.eu
