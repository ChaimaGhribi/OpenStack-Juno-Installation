####
Create your first instance with Neutron
####

=============================

In this guide we will provide a description of the steps followed to create an instance with Neutron.

It's simple and it takes three main steps :


1. Create your image
======================


* Edit the demo_creds file and add the following content::
   
    export OS_TENANT_NAME=demo
    export OS_USERNAME=demo
    export OS_PASSWORD=DEMO_PASS
    export OS_AUTH_URL=http://controller:5000/v2.0


* Upload the image to the Image Service::

    source demo_creds
    mkdir /tmp/images

    wget -P /tmp/images http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

    glance image-create --name "cirros-0.3.4-x86_64" --file /tmp/images/cirros-0.3.4-x86_64-disk.img \
    --disk-format qcow2 --container-format bare --is-public True --progress

    rm -r /tmp/images


* List Images::

    glance image-list
    
    
2. Create initial network (Neutron)
===================================

After creating the image, let's create the virtual network infrastructure to which 
the instance will connect.


* Create an external network::

    source admin_creds
    
    #Create the external network:
    neutron net-create ext-net --router:external True \
    --provider:physical_network external --provider:network_type flat
    
    #Create the subnet for the external network:
    neutron subnet-create ext-net --name ext-subnet \
    --allocation-pool start=192.168.100.101,end=192.168.100.200 \
    --disable-dhcp --gateway 192.168.100.1 192.168.100.0/24


* Create an internal (tenant) network::

    source demo_creds
    
    #Create the internal network:
    neutron net-create demo-net
    
    #Create the subnet for the internal network:
    neutron subnet-create demo-net --name demo-subnet 172.16.1.0/24 \
    --dns-nameservers list=true 8.8.4.4 8.8.8.8 --gateway 172.16.1.1 


* Create a router on the internal network and attach it to the external network::

    source demo_creds
    
    #Create the router:
    neutron router-create demo-router
    
    #Attach the router to the demo tenant subnet:
    neutron router-interface-add demo-router demo-subnet

    
    #Attach the router to the external network by setting it as the gateway:
    neutron router-gateway-set demo-router ext-net


* Verify network connectivity::

    #Ping the router gateway:
    ping 192.168.100.101    
 
 
3. Launch your instance 
=======================

* Source the demo tenant credentials::

   source demo_creds
 
* Generate a key pair::
 
   ssh-keygen

* Add the public key to your OpenStack environment::
    
    nova keypair-add --pub-key ~/.ssh/id_rsa.pub demo-key

* Verify the public key is added::
    
    nova keypair-list


* Add rules to the default security group to access your instance remotely::

   # Permit ICMP (ping):
   nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

   # Permit secure shell (SSH) access:
   nova secgroup-add-rule default tcp 22 22 0.0.0.0/0

* Launch your instance::
    
    DEMO_NET_ID=$(neutron net-list | awk '/ demo-net / { print $2 }')
    nova boot --flavor m1.tiny --image cirros-0.3.3-x86_64 --nic net-id=$DEMO_NET_ID \
    --security-group default --key-name demo-key demo-instance1
  
  
* Note: To choose your instance parameters you can use these commands::
    
    nova flavor-list   : --flavor m1.tiny 
    nova image-list    : --image cirros-0.3.3-x86_64
    neutron net-list   : --nic net-id=$DEMO_NET_ID
    nova secgroup-list : --security-group default 
    nova keypair-list  : --key-name demo-key 

* Check the status of your instance::

    nova list
  

* Create a floating IP address on the external network to enable the instance to acess to the internet and also to make it reachable from external networks::

    neutron floatingip-create ext-net

* Associate the floating IP address with your instance::

    nova floating-ip-associate demo-instance1 192.168.100.102

* Check the status of your floating IP address::

    nova list

* Verify network connectivity using ping and ssh::

    ping 192.168.100.102
    
    # ssh into your vm using its ip address:
    ssh cirros@192.168.100.102


 
Now you are finally done! You can enjoy your new instance ;)

Do not hesitate to contact to us for any question or suggestion :)


License
=======
Institut Mines Télécom - Télécom SudParis  

Copyright (C) 2015  Authors

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


Contacts
========

Chaima Ghribi: chaima.ghribi@it-sudparis.eu

Marouen Mechtri : marouen.mechtri@it-sudparis.eu
