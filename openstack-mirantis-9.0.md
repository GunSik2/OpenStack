
## Installation
- [Install Fuel Node](https://docs.mirantis.com/openstack/fuel/fuel-9.0/quickstart-guide.html#introduction)
- [Install OpenStack using Fuel UI](https://www.mirantis.com/blog/now-zero-openstack-hosted-website-4-easy-steps/)
- [Setting up local mirrors](https://docs.mirantis.com/openstack/fuel/fuel-8.0/operations.html#downloading-ubuntu-system-packages)

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
- openstack console access: http://172.16.0.3/horizon

- controller
```
# ssh root@controller
# . openrc
# openstack compute service list
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
