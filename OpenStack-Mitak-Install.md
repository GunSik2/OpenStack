
## Overview
- Node & Service Deployment
  - Node1 - Controller : Horizon, Neutron, Keystone, Glance, Heat
  - Node2 - Compute01 : Nova, Swift, Cinder
  - Node3 - Compute02 
  - Node4 - Compute04
- Networking 
  - [Self-service networks](http://docs.openstack.org/mitaka/install-guide-ubuntu/overview.html)
    - provider network : public network
    - management network : private network
    
## Environment
- Security 
```
echo "ADMIN_PASS=$(openssl rand -hex 10)
CEILOMETER_DBPASS=$(openssl rand -hex 10)
CEILOMETER_PASS=$(openssl rand -hex 10)
CINDER_DBPASS=$(openssl rand -hex 10)
CINDER_PASS=$(openssl rand -hex 10)
DASH_DBPASS=$(openssl rand -hex 10)
DEMO_PASS=$(openssl rand -hex 10)
GLANCE_DBPASS=$(openssl rand -hex 10)
GLANCE_PASS=$(openssl rand -hex 10)
HEAT_DBPASS=$(openssl rand -hex 10)
HEAT_DOMAIN_PASS=$(openssl rand -hex 10)
HEAT_PASS=$(openssl rand -hex 10)
KEYSTONE_DBPASS=$(openssl rand -hex 10)
NEUTRON_DBPASS=$(openssl rand -hex 10)
NEUTRON_PASS=$(openssl rand -hex 10)
NOVA_DBPASS=$(openssl rand -hex 10)
NOVA_PASS=$(openssl rand -hex 10)
RABBIT_PASS=$(openssl rand -hex 10)
SWIFT_PASS=$(openssl rand -hex 10)" > password.sh
```
- Host networking
  - All nodes have two interfaces, provider & management interfaces.
    - Management on 10.0.0.0/24 with gateway 10.0.0.1
    - Provider on 203.0.113.0/24 with gateway 203.0.113.1
- NTP
  - Compute
```
# apt-get install chrony
# /etc/chrony/chrony.conf
server NTP_SERVER iburst
# service chrony restart
```

Reference
- http://docs.openstack.org/mitaka/install-guide-ubuntu/common/conventions.html
