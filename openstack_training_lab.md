## Env
Install openstack ocata on Windows 10 host.

## Steps
- Donwload and unzip labs file: http://tarballs.openstack.org/training-labs/dist/labs-stable-ocata.zip
- Donwload ubuntu iso image ([ubuntu-16.04.4-server-amd64.iso](https://www.ubuntu.com/download/server)) and copy it to the folder: labs-stable-ocata\labs\osbash\img\
- Open the create-base script file and replace "ubuntu-16.04.3-server-amd64.iso" to "ubuntu-16.04.4-server-amd64.iso" :  labs-stable-ocata/labs/osbash/wbatch/create_base.bat 
- Open Command window and Run
```
cd labs-stable-ocata\labs\osbash\wbatch
create_hostnet.bat
create_base.bat
create_ubuntu_cluster_node.bat
```

## Use
- Access horizon: http://127.0.0.1:8888/horizon
  - Domain: default
  - ID/PW: admin/admin_user_secret , demo/demo_user_pass
  
- Access ssh
```
ssh -p 2230 osbash@localhost   # controller (note the network node is incorporated here since OpenStack Liberty)
ssh -p 2232 osbash@localhost   # compute node
```
## Reference
- [OpenStack Training Labs](https://docs.openstack.org/training_labs/)
- [Training Lab Guide with Ocata](http://www.netlabsug.org/documentum/Openstack-Laboratory-Guide_v4.0.3-Ocata-Release.pdf)
