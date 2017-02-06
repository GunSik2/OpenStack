
## Installation
- [Install Fuel Node](https://docs.mirantis.com/openstack/fuel/fuel-9.0/quickstart-guide.html#introduction)
- [Install OpenStack using Fuel UI](https://www.mirantis.com/blog/now-zero-openstack-hosted-website-4-easy-steps/)
- [Setting up local mirrors](https://docs.mirantis.com/openstack/fuel/fuel-8.0/operations.html#downloading-ubuntu-system-packages)
## Fule Configuration
- default network (public network) : 172.16.0.0/24
- stroage network : 192.168.1.0/24
- management network : 192.168.0.0/24

## Fule node add
- add-slave-nodes.sh
```
CONFIG_FOR=8G
ADD_NODES="4 5"
vm_slave_memory_default=8192
vm_slave_cpu_default=4
for idx in $ADD_NODES; do #for idx in $(eval echo {1..$cluster_size}); do
```
## Management
- login
```
# ssh root@10.20.0.2
# cat /etc/hosts
10.20.0.4 controller 
10.20.0.3 compute 
10.20.0.5 cinder 
# ssh root@controller
```
- fule console access: http://10.20.0.2:8000/ https://172.16.0.1:8443/ https://192.168.200.15:8443
- openstack console access: http://172.16.0.3/horizon

- controller
```
# ssh root@controller
# . openrc
# openstack compute service list
```

## Tenant 설정 

Admin 계정 로그인
- project 생성 (Identity>Projects)
- user 생성 (Identity>Users)

Tenant 계정 로그인
- netowrk 생성 (Network>Networks)
  - CIDR: 192.168.10.0/24
  - GW: 192.168.10.1
  - DNS: 8.8.8.8, 8.8.4.4
- router 생성 (Network>Routers)
- subnet interface 생성 (Network>Router >> Interface tab)
- SSH keypair 생성 (Project>Compute>Access & Security >> keypair tab)
- Default Security Group Rules 생성 (Project>Compute>Access & Security >> Security Groups tab)
- Floating IPs 할당 (Project>Compute>Access & Security >> Floating IPs tab)

- 이미지 생성 (Project>Compute>Image >> Create Image)
  - name (e.g., ubuntu-trusty)
  - URL: http://uec-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img
  - Format, pick QCOW2
  - Ensure that Copy Data is checked
- VM 생성 (Project>Compute>Images >> click Lauch instance)
- VM 테스트 
```
ssh -i hellokeypair.pem ubuntu@172.16.0.140
sudo apt-get update
sudo apt-get install apache2
```

## Reference
- Mirantis
  - https://www.mirantis.com/get-started/
  - https://www.mirantis.com/software/openstack/releases/
  - https://docs.mirantis.com/openstack/fuel/fuel-9.0/pdfs.html
  - https://docs.mirantis.com/openstack/fuel/fuel-9.0/pdf/Mirantis-OpenStack-9.0-QuickStartGuide.pdf
  - https://docs.mirantis.com/openstack/fuel/fuel-9.0/pdf/Mirantis-OpenStack-9.0-ReferenceArchitecture.pdf
  - https://docs.mirantis.com/openstack/fuel/fuel-9.0/pdf/Mirantis-OpenStack-9.0-PlanningGuide.pdf
- Fuel
  - https://www.fuel-infra.org
  - https://wiki.openstack.org/wiki/Fuel#Source_code
  - https://github.com/openstack/fuel-main
  - http://docs.openstack.org/developer/fuel-docs/userdocs/fuel-user-guide.html
- Virtualbox
  - https://www.virtualbox.org/wiki/Downloads
- Blog
  - https://sagarruchandani.wordpress.com/tag/mirantis-openstack/
