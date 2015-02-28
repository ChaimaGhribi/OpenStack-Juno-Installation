####
OpenStack Juno Installation - Multi Node
####

Welcome to OpenStack Juno installation manual !

This document is based on `the OpenStack Official Documentation <http://docs.openstack.org/juno/install-guide/install/apt/content/index.html>`_ for Juno. 

===============================

**Authors:**

Copyright (C) `Chaima Ghribi <https://www.linkedin.com/profile/view?id=53659267&trk=nav_responsive_tab_profile>`_

Copyright (C) `Marouen Mechtri <https://www.linkedin.com/in/mechtri>`_


================================

.. contents::
   

Basic Architecture and Network Configuration
==========================================

In this installation guide, we cover the step-by-step process of installing Openstack Juno on Ubuntu 14.04.  We consider a multi-node architecture with Openstack Networking (Neutron) that requires three node types: 

+ **Controller Node** that runs management services (keystone, Horizon…) needed for OpenStack to function.

+ **Network Node** that runs networking services and is responsible for virtual network provisioning  and for connecting virtual machines to external networks.

+ **Compute Node** that runs the virtual machine instances in OpenStack. 

We have deployed a single compute node (see the Figure below) but you can simply add more compute nodes to our multi-node installation, if needed.  



.. image:: https://raw.githubusercontent.com/ChaimaGhribi/OpenStack-Juno-Installation/master/images/network-topo.jpg

For OpenStack Multi-Node setup you need to create three networks:

+ **Management Network** (10.0.0.0/24): A network segment used for administration, not accessible to the public Internet. **This network is configured with natting in order to be able to install OpenStack pachages**.

+ **VM Traffic Network** (10.0.1.0/24): This network is used as internal network for traffic between virtual machines in OpenStack, and between the virtual machines and the network nodes that provide L3 routes out to the public network.

+ **Public Network**: This network is connected to the network nodes to provide VMs with publicly routable traffic functionality.


In the next subsections, we describe in details how to set up, configure and test the network architecture. We want to make sure everything is ok before install ;)

So, let’s prepare the nodes for OpenStack installation!

Configure Controller node
-------------------------

The controller node has one Network Interface: eth0.

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

* Edit network settings to configure the interface eth0::

    vi /etc/network/interfaces
      
    # The management network interface
      auto eth0
      iface eth0 inet static
      address 10.0.0.11
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 8.8.8.8


* Restart network::

    ifdown eth0 && ifup eth0
    
           
    
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
      gateway 10.0.0.1
      dns-nameservers 8.8.8.8

    
    # VM traffic interface
      auto eth1
      iface eth1 inet static
      address 10.0.1.21
      netmask 255.255.255.0
    
    # The public network interface
      auto eth2
      iface eth2 inet manual
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down



* Restart network::

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
      gateway 10.0.0.1
      dns-nameservers 8.8.8.8

  
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

**TO UPDATE**
An additional install guide for optional services (Heat, Celiometer...) are provided in this guide ;) 

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
    admin_token=ADMIN
    verbose = True

    [token]
    provider = keystone.token.providers.uuid.Provider
    driver = keystone.token.persistence.backends.sql.Token


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
    keystone user-create --name demo --pass=demo_pass --email=demo@domain.com
    keystone tenant-create --name=demo --description "Demo Tenant"
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

    #As the admin tenant and user, request an authentication token:
     keystone --os-tenant-name admin --os-username admin --ospassword ADMIN_PASS \--os-auth-url http://controller:35357/v2.0 token-get

    #As the admin tenant and user, list tenants: 
     keystone --os-tenant-name admin --os-username admin --os-password admin_pass --os-auth-url http://controller:35357/v2.0 tenant-list
    
    #As the admin tenant and user, list users to verify that the Identity service contains the users that you created:
     keystone --os-tenant-name admin --os-username admin --os-password admin_pass --os-auth-url http://controller:35357/v2.0 user-list
     
    #As the admin tenant and user, list roles to verify that the Identity service contains the role that you created:
     keystone --os-tenant-name admin --os-username admin --os-password admin_pass --os-auth-url http://controller:35357/v2.0 role-list
     
    #As the demo tenant and user, request an authentication token:
     keystone --os-tenant-name demo --os-username demo --os-password demo_pass --os-auth-url http://controller:35357/v2.0 token-get 

    #As the demo tenant and user, attempt to list users to verify that you cannot execute admin-only CLI commands:
     keystone --os-tenant-name demo --os-username demo --os-password demo_pass --os-auth-url http://controller:35357/v2.0 user-list
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

    keystone service-create --name glance --type image   --description "OpenStack Image Service"
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
    cd /tmp/images

    wget http://cdn.download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img

    glance image-create --name "cirros-0.3.3-x86_64" --file cirros-0.3.3-x86_64-disk.img --disk-    format qcow2 --container-format bare --is-public True --progress

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

    apt-get install nova-api nova-cert nova-conductor nova-consoleauth \
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

* Check Nova is running. The :-) icons indicate that everything is ok !::
    
    nova-manage service list

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

    apt-get install neutron-server neutron-plugin-ml2 python-neutronclient
    
* Update /etc/neutron/neutron.conf::
      
    vi /etc/neutron/neutron.conf
    
    [database]
    replace connection = sqlite:////var/lib/neutron/neutron.sqlite with
    connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron
    
    [DEFAULT]
    rpc_backend = rabbit
    rabbit_host = controller
    rabbit_password = RABBIT_PASS
    
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
    # Replace the SERVICE_TENANT_ID with the output of this command (keystone tenant-list | awk '/ service / { print $2 }')
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


* Check OpenStack Dashboard at http://192.168.100.11/horizon. login admin/admin_pass

Enjoy it !
