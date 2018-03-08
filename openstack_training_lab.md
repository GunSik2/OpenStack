## Env
Install openstack ocata on Windows 10 host.

## Steps
- Donwload and unzip labs file: http://tarballs.openstack.org/training-labs/dist/labs-stable-ocata.zip
- Donwload ubuntu iso image (ubuntu-16.04.4-server-amd64.iso) and copy it to the folder: labs-stable-ocata\labs\osbash\img\
- Open the create-base script file and replace "ubuntu-16.04.3-server-amd64.iso" to "ubuntu-16.04.4-server-amd64.iso" :  labs-stable-ocata/labs/osbash/wbatch/create_base.bat 
- Open Command window and Run
```
cd labs-stable-ocata\labs\osbash\wbatch
create_hostnet.bat
create_base.bat
create_ubuntu_cluster_node.bat
```
## Reference
- [OpenStack Training Labs](https://docs.openstack.org/training_labs/)
- [Training Lab Guide with Pike](http://www.netlabsug.org/documentum/Openstack-Laboratory-Guide_v5.0.1-Pike-Release.pdf)
