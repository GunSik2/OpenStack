####
OpenStack Icehouse 운영 가이드
####

.. contents::

Upload ubuntu image to Glance
==============================

Use Console Command 
^^^^^^^^^^^^^^^^^^^^
* Upload ubuntu cloud image ::

    souce admin_creds
    glance image-create --name "Ubuntu_14.04_LTS" --is-public true \
    --container-format bare --disk-format qcow2  \
    --location http://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img

* List Images::
    glance image-list

Use Horizon Dashboard
^^^^^^^^^^^^^^^^^^^^
* Go to the page after login to Horizon ::
   관리자 > 시스템 패널 > 이미지 


Create initial network (Neutron)
==============================

After creating the image, let's create the virtual network infrastructure to which 
the instance will connect.

* Create an external network::

    source creds
    
    #Create the external network:
    neutron net-create ext-net --shared --router:external=True
    
    #Create the subnet for the external network:
    neutron subnet-create ext-net --name ext-subnet \
    --allocation-pool start=192.168.100.101,end=192.168.100.200 \
    --disable-dhcp --gateway 192.168.100.1 192.168.100.0/24

* Create an internal (tenant) network::

    source creds
    
    #Create the internal network:
    neutron net-create int-net
    
    #Create the subnet for the internal network:
    neutron subnet-create int-net --name int-subnet \
    --dns-nameserver 8.8.8.8 --gateway 172.16.1.1 172.16.1.0/24


* Create a router on the internal network and attach it to the external network::

    source creds
    
    #Create the router:
    neutron router-create router1
    
    #Attach the router to the internal subnet:
    neutron router-interface-add router1 int-subnet
    
    #Attach the router to the external network by setting it as the gateway:
    neutron router-gateway-set router1 ext-net

* Verify network connectivity::

    #Ping the router gateway:
    
    ping 192.168.100.101

Create a ubuntu instace 
==============================

Use Console Command 
^^^^^^^^^^^^^^^^^^^^

* Generate a key pair:
    ssh-keygen -f vm

* Add the public key:
    source creds
    nova keypair-add --pub-key vm.pub key1

* Verify the public key is added:
    nova keypair-list
  
* Add rules to the default security group to access your instance remotely:
    nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
    nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
  
* Create demo credential info ::
    vi demo_creds
      export OS_USERNAME=demo
      export OS_PASSWORD=demo_pass
      export OS_TENANT_NAME=demo
      export OS_AUTH_URL=http://controller:35357/v2.0
    
    souce demo_creds

* Launch an instance:
    NET_ID=$(neutron net-list | awk '/ int-net / { print $2 }')
    nova boot --flavor m1.tiny --image cirros-0.3.2-x86_64 --nic net-id=$NET_ID \
    --security-group default --key-name key1 instance1

* Note: To choose your instance parameters you can use these commands:
    nova flavor-list   : --flavor m1.tiny
    nova image-list    : --image cirros-0.3.2-x86_64
    neutron net-list   : --nic net-id=$NET_ID
    nova secgroup-list : --security-group default
    nova keypair-list  : --key-name key1
  
* Check the status of your instance:
    nova list
  
* Create a floating IP address on the external network
    neutron floatingip-create ext-net

* Associate the floating IP address with your instance:
    nova floating-ip-associate instance1 192.168.100.102

* Check the status of your floating IP address:
    ping 192.168.100.102
    
    # ssh into your vm using its ip address:
    ssh cirros@192.168.100.102


Use Horizon Dashboard
^^^^^^^^^^^^^^^^^^^^
* Go to the page after login to Horizon ::
   Project > Compute > Access & Security - Key Pair - (+ create key pair)


Set floating ip to an instace 
==============================


Create volume stoarge
==============================


Create volume stoarge
==============================
