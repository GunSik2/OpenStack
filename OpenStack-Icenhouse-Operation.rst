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

    source admin_creds
    
    #Create the external network:
    neutron net-create ext-net --shared --router:external=True
    
    #Create the subnet for the external network:
    neutron subnet-create ext-net --name ext-subnet \
    --allocation-pool start=192.168.100.101,end=192.168.100.200 \
    --disable-dhcp --gateway 192.168.100.1 192.168.100.0/24


* Create an internal (tenant) network::

    source demo_creds
    
    #Create the internal network:
    neutron net-create demo-net
    
    #Create the subnet for the internal network:
    neutron subnet-create demo-net --name demo-subnet \
    --dns-nameserver 8.8.8.8 --gateway 172.16.1.1 172.16.1.0/24

* Create a router on the internal network and attach it to the external network::

    source demo_creds
    
    #Create the router:
    neutron router-create demo-router
    
    #Attach the router to the internal subnet:
    neutron router-interface-add demo-router demo-subnet
    
    #Attach the router to the external network by setting it as the gateway:
    neutron router-gateway-set demo-router ext-net

* Verify network connectivity::

    #Ping the router gateway:
    ping 192.168.100.101


Create a ubuntu instace 
==============================

Use Console Command 
^^^^^^^^^^^^^^^^^^^^

* Generate a key pair::

    ssh-keygen -f demo-key

* Add the public key::

    source demo_creds
    nova keypair-add --pub-key demo-key.pub demo-key

* Verify the public key is added::

    nova keypair-list
  
* Add rules to the default security group to access your instance remotely::

    nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
    nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
  
* Launch an instance::

    NET_ID=$(neutron net-list | awk '/ demo-net / { print $2 }')
    nova boot --flavor m1.tiny --image cirros-0.3.2-x86_64 --nic net-id=$NET_ID \
    --security-group default --key-name demo-key instance1

* Note: To choose your instance parameters you can use these commands::

    nova flavor-list   : --flavor m1.tiny
    nova image-list    : --image cirros-0.3.2-x86_64
    neutron net-list   : --nic net-id=$NET_ID
    nova secgroup-list : --security-group default
    nova keypair-list  : --key-name demo-key
  
* Check the status of your instance::

    nova list
  
* Create a floating IP address on the external network::

    neutron floatingip-create ext-net

* Associate the floating IP address with your instance::

    nova floating-ip-associate instance1 192.168.100.102

* Check the status of your floating IP address::

    ping 192.168.100.102
    
    # ssh into your vm using its ip address
    ssh cirros@192.168.100.102


Use Horizon Dashboard
^^^^^^^^^^^^^^^^^^^^
* Create Keypair ::
   (프로젝트 > Compute > 접근 & 시큐리티)메뉴 - (키 패어)탭 - (+ 키 패어 생성)버튼
   (키 패어 이름: key-test) 입력 - (키 패어 생성)버튼
   
* Create instance ::
   (프로젝트 > Compute > 인스턴스)메뉴 - (+ 인스턴스 시작)버튼
   (세부 정보)탬 - (인스턴스 이름:instance1, Flavor: m1.tiny, 인스턴스 부팅 소스: 이미지로 부팅, 이미지 이름: Ubuntu_14.04_LTS)입력
   (접근 & 시큐리티)탭 - (키 패어: key-test, 시큐리티 그룹: defualt)확인
   (네트워킹) 
   


Set floating ip to an instace 
==============================


Create volume stoarge
==============================


Create volume stoarge
==============================
