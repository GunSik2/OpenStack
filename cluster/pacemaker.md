
## Error
- net ip was not match. 
```
$ sudo crm node online primary
$ sudo crm status
Last updated: Mon Apr 23 12:13:57 2018          Last change: Mon Apr 23 12:13:54 2018 by root via crm_attribute on primary
Stack: corosync
Current DC: primary (version 1.1.14-70404b0) - partition with quorum
2 nodes and 1 resource configured

Online: [ primary secondary ]

Full list of resources:

 virtual_public_ip      (ocf::heartbeat:IPaddr2):       Stopped

$ sudo tail -f /var/log/corosync/corosync.log
Failed Actions:
* virtual_public_ip_start_0 on primary 'unknown error' (1): call=12, status=complete, exitreason='[findif] failed',
    last-rc-change='Mon Apr 23 12:13:54 2018', queued=0ms, exec=46ms
* virtual_public_ip_start_0 on secondary 'unknown error' (1): call=18, status=complete, exitreason='[findif] failed',
    last-rc-change='Mon Apr 23 12:13:54 2018', queued=0ms, exec=25ms
```

## Reference
- pacemaker ubuntu 16.04 https://medium.com/@yenthanh/high-availability-using-corosync-pacemaker-on-ubuntu-16-04-bdebc6183fc5
- ansible pacemaker https://github.com/devgateway/ansible-role-pacemaker
- pacemaker ubuntu 14.04 https://www.digitalocean.com/community/tutorials/how-to-create-a-high-availability-setup-with-corosync-pacemaker-and-floating-ips-on-ubuntu-14-04#create-floating-ip-reassignment-resource-agent
