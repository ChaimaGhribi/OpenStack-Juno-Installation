####
OpenStack Juno Installation - Multi Node
####

Welcome to OpenStack Juno installation manual !

This document is based on `the OpenStack Official Documentation <http://docs.openstack.org/juno/install-guide/install/apt/content/index.html>`_ for Icehouse. 

:Version: 1.0
:Authors: Chaima Ghribi and Marouen Mechtri
:License: Apache License Version 2.0
:Keywords: OpenStack, Juno, Heat, Neutron, Nova, Ubuntu 14.04, Glance, Horizon


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



.. image:: https://raw.githubusercontent.com/ChaimaGhribi/OpenStack-Icehouse-Installation/master/images/network-topo.jpg

For OpenStack Multi-Node setup you need to create three networks:

+ **Management Network** (10.0.0.0/24): A network segment used for administration, not accessible to the public Internet.
It is configured with natting. 

+ **VM Traffic Network** (10.0.1.0/24): This network is used as internal network for traffic between virtual machines in OpenStack, and between the virtual machines and the network nodes that provide L3 routes out to the public network.

+ **Public Network** (192.168.100.0/24): This network is connected to the controller nodes so users can access the OpenStack interfaces, and connected to the network nodes to provide VMs with publicly routable traffic functionality.


In the next subsections, we describe in details how to set up, configure and test the network architecture. We want to make sure everything is ok before install ;)

So, let’s prepare the nodes for OpenStack installation!

Configure Controller node
-------------------------

The controller node has one Network Interface: eth0 is internal (used for connectivity for OpenStack nodes).

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
