# Ceph 5 Playground (CNV) - Workshop Instructions
> *Note: This workshop is not intended to replace the official [“CL260 - Cloud Storage with Red Hat Ceph Storage”](https://www.redhat.com/en/services/training/cloud-storage-red-hat-ceph-storage-cl260) course in any instance. It is just a half day test-drive for the ones aiming to get their first hands-on experience with the product. For a more detailed and extended training, take the official course.*

#### <u>BOOTSTRAPPING</u>

Connect to the `workstation`, and then to the `ceph-mon01` node from the workstation:

<em>
<pre>
[user@localhost ~]$ ssh lab-user@<\your-workstation-fqdn-here> -p 31147
<br>  
[lab-user@workstation ~]$ sudo -i
<br>
[root@workstation ~]# ssh ceph-mon01
</pre>
</em>

List the available repositories:

<em>
<pre>
[cloud-user@ceph-mon01 ~]$ dnf repolist
Not root, Subscription Management repositories not updated
repo id                                                    repo name
ansible-2-for-rhel-8-x86_64-rpms                           Red Hat Ansible Engine 2 for RHEL 8 x86_64 (RPMs)
ansible-2.9-for-rhel-8-x86_64-rpms                         Red Hat Ansible Engine 2.9 for RHEL 8 x86_64 (RPMs)
rhceph-5-tools-for-rhel-8-x86_64-rpms                      Red Hat Ceph Storage Tools 5 for RHEL 8 x86_64 (RPMs)
rhel-8-for-x86_64-appstream-rpms                           Red Hat Enterprise Linux 8 for x86_64 - AppStream (RPMs)
rhel-8-for-x86_64-baseos-rpms                              Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs)
</pre>
</em>

Install the `cephadm` utility:

<em>
<pre>
[cloud-user@ceph-mon01 ~]$ sudo -i
<br>
[root@ceph-mon01 ~]# dnf install -y cephadm
</pre>
</em>

Navigate to [Registry Service Accounts](https://access.redhat.com/terms-based-registry/#/accounts) to create a service account and obtain the pull screen.

Click on the `New Service Account`button. In the `Name` field, the first 8 characters are automatically provided to you as seven integers and a pipeline (e.g. `1234567|`). You need to fill the box with your preffered name. The full name should something like `1234567|username`. This is going to be your user. After creating it, search for your user typing its name in the search bar.

The password is generated automatically after the creation of the user and it is going to be shown as a token. Use the full username as created before as your user and the token as your password for the next command.

Run `podman login` as shown below:

<em>
<pre>
[root@ceph-mon01 ~]# podman login -u='1234567|username' -p='your_token_here' registry.redhat.io
</pre>
</em>

Bootstrap the standalone Ceph cluster:

<em>
<pre>
[root@ceph-mon01 ~]# cephadm bootstrap --ssh-user cloud-user  --mon-ip 192.168.56.64 --allow-fqdn-hostname
</pre>
</em>

Use the command `cephadm shell` to interactive operate with the cluster:

<em>
<pre>
[root@ceph-mon01 ~]# cephadm shell
Inferring fsid b0958ad8-e8fa-11ee-b03e-525400da0103
Using recent ceph image registry.redhat.io/rhceph/rhceph-5-rhel8@sha256:72432b0839e14d91219f57c9255876ac1964e5eb2ccfb31d45102bba2bead444
<br>
[ceph: root@ceph-mon01 /]# ceph -s
  cluster:
    id:     b0958ad8-e8fa-11ee-b03e-525400da0103
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum ceph-mon01 (age 2m)
    mgr: ceph-mon01.gefecy(active, since 94s)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
</pre>
</em>

List services of the new installation:

<em>
<pre>
[ceph: root@ceph-mon01 /]# ceph orch ls
NAME           PORTS        RUNNING  REFRESHED  AGE  PLACEMENT  
alertmanager   ?:9093,9094      1/1  113s ago   4m   count:1    
crash                           1/1  113s ago   4m   *          
grafana        ?:3000           1/1  113s ago   4m   count:1    
mgr                             1/2  113s ago   4m   count:2    
mon                             1/5  113s ago   4m   count:5    
node-exporter  ?:9100           1/1  113s ago   4m   *          
prometheus     ?:9095           1/1  113s ago   4m   count:1    
</pre>
</em>

#### <u>SCALING UP</u>

Copy the ceph key to the `ceph-mon02`, `ceph-mon03`, `ceph-node01`, `ceph-node02` and `ceph-node03` servers and login the registry:

<em>
<pre>
[ceph: root@ceph-mon01 /]# exit
<br>
[root@ceph-mon01 ~]# cephadm shell ceph cephadm get-pub-key > ~/ceph.pub
<br>
[root@ceph-mon01 ~]# for node in ceph-mon0{2,3} ceph-node0{1,2,3}; do cat ~/ceph.pub | ssh cloud-user@${node} "tee -a ~/.ssh/authorized_keys"; ssh ${node} --  "sudo podman login -u='1234567|username' -p='your_token_here' registry.redhat.io"; done
</pre>
</em>

Add new hosts to the cluster:

<em>
<pre>
[root@ceph-mon01 ~]# cephadm shell
<br>
[ceph: root@ceph-mon01 /]# ceph orch host add ceph-mon02 192.168.56.65
Added host 'ceph-mon02' with addr '172.16.7.65'
<br>
[ceph: root@ceph-mon01 /]# ceph orch host add ceph-mon03 192.168.56.66
Added host 'ceph-mon03' with addr '172.16.7.66'
</pre>
</em>

List hosts after creating them:

<em>
<pre>
[ceph: root@ceph-mon01 /]# ceph orch host ls
HOST        ADDR         LABELS  STATUS  
ceph-mon01 192.168.56.64  _admin          
ceph-mon02 192.168.56.65                  
ceph-mon03 192.168.56.66                  
3 hosts in cluster
</pre>
</em>

Check the status after a while and ensure the `ceph-mon02` and `ceph-mon03` are part of the monitoring service:

<em>
<pre>
[ceph: root@ceph-mon01 /]# ceph -s
  cluster:
    id:     b0958ad8-e8fa-11ee-b03e-525400da0103
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum ceph-mon01,ceph-mon02,ceph-mon03 (age 4m)
    mgr: ceph-mon01.gefecy(active, since 21m), standbys: ceph-mon02.bcbfky
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
</pre>
</em>

#### <u>ADDING OSD NODES</u>

Add OSD nodes to the cluster:

<em>
<pre>
[ceph: root@ceph-mon01 /]# ceph orch host add ceph-node01 192.168.56.61 osd
<br>
[ceph: root@ceph-mon01 /]# ceph orch host add ceph-node02 192.168.56.62 osd
<br>
[ceph: root@ceph-mon01 /]# ceph orch host add ceph-node03 192.168.56.63 osd
</pre>
</em>

Add the disks for each of the nodes:

<em>
<pre>
[ceph: root@ceph-mon01 /]# for node in ceph-node0{1,2,3}; do for disk in /dev/vd{b,c,d,e}; do  ceph orch daemon add osd ${node}:${disk}; done; done
Created osd(s) 0 on host 'ceph-node01'
Created osd(s) 1 on host 'ceph-node01'
Created osd(s) 2 on host 'ceph-node01'
Created osd(s) 3 on host 'ceph-node01'
Created osd(s) 4 on host 'ceph-node02'
Created osd(s) 5 on host 'ceph-node02'
Created osd(s) 6 on host 'ceph-node02'
Created osd(s) 7 on host 'ceph-node02'
Created osd(s) 8 on host 'ceph-node03'
Created osd(s) 9 on host 'ceph-node03'
Created osd(s) 10 on host 'ceph-node03'
Created osd(s) 11 on host 'ceph-node03'
</pre>
</em>

List the added OSDs to the cluster:

<em>
<pre>
[ceph: root@ceph-mon01 /]# ceph osd tree
ID   CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
 -1         0.14694  root default                                   
 -9         0.03918      host ceph-node01                           
  3    hdd  0.00980          osd.3             up   1.00000  1.00000
  6    hdd  0.00980          osd.6             up   1.00000  1.00000
  9    hdd  0.00980          osd.9             up   1.00000  1.00000
 13    hdd  0.00980          osd.13            up   1.00000  1.00000
-13         0.03918      host ceph-node02                           
  5    hdd  0.00980          osd.5             up   1.00000  1.00000
  8    hdd  0.00980          osd.8             up   1.00000  1.00000
 11    hdd  0.00980          osd.11            up   1.00000  1.00000
 14    hdd  0.00980          osd.14            up   1.00000  1.00000
-11         0.03918      host ceph-node03                           
  4    hdd  0.00980          osd.4             up   1.00000  1.00000
  7    hdd  0.00980          osd.7             up   1.00000  1.00000
 10    hdd  0.00980          osd.10            up   1.00000  1.00000
 12    hdd  0.00980          osd.12            up   1.00000  1.00000
<br>
[ceph: root@ceph-mon01 /]# ceph orch device ls
HOST         PATH      TYPE  DEVICE ID   SIZE  AVAILABLE  REFRESHED  REJECT REASONS                                                 
ceph-mon01   /dev/vdb  hdd              10.7G  Yes        20m ago                                                                   
ceph-mon01   /dev/vdc  hdd              10.7G  Yes        20m ago                                                                   
ceph-mon01   /dev/vdd  hdd              1048k             20m ago    Insufficient space (<5GB)                                      
ceph-mon02   /dev/vdb  hdd              10.7G  Yes        20m ago                                                                   
ceph-mon02   /dev/vdc  hdd              10.7G  Yes        20m ago                                                                   
ceph-mon02   /dev/vdd  hdd              1048k             20m ago    Insufficient space (<5GB)                                      
ceph-mon03   /dev/vdb  hdd              10.7G  Yes        20m ago                                                                   
ceph-mon03   /dev/vdc  hdd              10.7G  Yes        20m ago                                                                   
ceph-mon03   /dev/vdd  hdd              1048k             20m ago    Insufficient space (<5GB)                                      
ceph-node01  /dev/vdb  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node01  /dev/vdc  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node01  /dev/vdd  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node01  /dev/vde  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node01  /dev/vdf  hdd              1048k             20m ago    Insufficient space (<5GB)                                      
ceph-node02  /dev/vdb  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node02  /dev/vdc  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node02  /dev/vdd  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node02  /dev/vde  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node02  /dev/vdf  hdd              1048k             20m ago    Insufficient space (<5GB)                                      
ceph-node03  /dev/vdb  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node03  /dev/vdc  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node03  /dev/vdd  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node03  /dev/vde  hdd              10.7G             20m ago    Insufficient space (<10 extents) on vgs, LVM detected, locked  
ceph-node03  /dev/vdf  hdd              1048k             20m ago    Insufficient space (<5GB)
</pre>
</em>

#### <u>MANAGING RBD</u>

Create a RBD pool and client:

<em>
<pre>
[ceph: root@ceph-mon01 /]# exit
<br>
[root@ceph-mon01 ~]# ceph osd pool create rbd 64
pool 'rbd' created
<br>
[root@ceph-mon01 ~]# ceph osd pool application enable rbd rbd
enabled application 'rbd' on pool 'rbd'
<br>
[root@ceph-mon01 ~]# ceph auth get-or-create client.rbd.ceph-mon01 osd 'allow rwx' mon 'allow r' -o /etc/ceph/ceph.client.rbd.ceph-mon01.keyring
<br>
[root@ceph-mon01 ~]# rbd create rbd/test --size=128M
<br>
[root@ceph-mon01 ~]# rbd ls
test
<br>
[root@ceph-mon01 ~]# rbd info rbd/test
rbd image 'test':
	size 128 MiB in 32 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 5fd8de40a9d0
	block_name_prefix: rbd_data.5fd8de40a9d0
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features: 
	flags: 
	create_timestamp: Sat Mar 23 10:25:24 2024
	access_timestamp: Sat Mar 23 10:25:24 2024
	modify_timestamp: Sat Mar 23 10:25:24 2024
</pre>
</em>

Disconnect from `ceph-mon01` (_Note: You will to run only one command inside this host_) and go back to workstation. Then map the RBD volume, format it with a filesystem, mount it, test it, then delete everything:

<em>
<pre>
[root@workstation ~]# ssh ceph-mon01 sudo cat /etc/ceph/ceph.client.rbd.ceph-mon01.keyring > /etc/ceph/ceph.client.rbd.ceph-mon01.keyring
<br>
[root@workstation ~]# ssh ceph-mon01 sudo cat /etc/ceph/ceph.conf > /etc/ceph/ceph.conf
<br>
[root@workstation ~]# rbd --id rbd.ceph-mon01 map rbd/test
/dev/rbd0
<br>
[root@workstation ~]# rbd showmapped
id  pool  namespace  image  snap  device   
0   rbd              test   -     /dev/rbd0
<br>
[root@workstation ~]# mkfs.ext4 /dev/rbd0
mke2fs 1.45.6 (20-Mar-2020)
Discarding device blocks: done                            
Creating filesystem with 131072 1k blocks and 32768 inodes
Filesystem UUID: e0bc23e3-152e-4a15-af48-d293213b3214
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 
<br>
[root@workstation ~]# mount /dev/rbd0 /mnt
<br>
[root@workstation ~]# mount | grep rbd
/dev/rbd0 on /mnt type ext4 (rw,relatime,seclabel,stripe=64)
<br>
[root@workstation ~]# df -hT /mnt
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/rbd0      ext4  120M  1.6M  110M   2% /mnt
<br>
[root@workstation ~]# umount /mnt 
<br>
[root@workstation ~]# rbd unmap /dev/rbd0
<br>
[root@workstation ~]# rbd showmapped
<br>
[root@ceph-mon01 ~]# rbd rm rbd/test
Removing image: 100% complete...done.
<br>
[root@workstation ~]# rbd -p rbd ls
</pre>
</em>

#### <u>MANAGING MDS/CEPHFS</u>

Connect again to the ceph-mon01 to create a new filesystem. It will deploy a mds service:

<em>
<pre>
[root@ceph-mon01 ~]# ceph fs volume create fs_name --placement=ceph-mon01,ceph-mon02
<br>
[root@ceph-mon01 ~]# ceph fs volume ls
[
    {
        "name": "fs_name"
    }
]
<br>
[root@ceph-mon01 ~]# ceph fs status fs_name
fs_name - 0 clients
\=======
RANK  STATE              MDS                ACTIVITY     DNS    INOS   DIRS   CAPS  
 0    active  fs_name.ceph-mon01.scpanh  Reqs:    0 /s    10     13     12      0   
        POOL           TYPE     USED  AVAIL  
cephfs.fs_name.meta  metadata  96.0k  37.9G  
cephfs.fs_name.data    data       0   37.9G  
       STANDBY MDS         
fs_name.ceph-mon02.qaomxe  
MDS version: ceph version 16.2.10-248.el8cp (0edb63afd9bd3edb333364f2e0031b77e62f4896) pacific (stable)
<br>
[root@ceph-mon01 ~]# ceph mds stat
fs_name:1 {0=fs_name.ceph-mon02.ianmhh=up:active} 1 up:standby
</pre>
</em>

Go back to workstation to mount the cephfs filesystem using mount:

<em>
<pre>
[root@workstation ~]# ssh ceph-mon01 sudo cat /etc/ceph/ceph.client.admin.keyring > /etc/ceph/ceph.client.admin.keyring
<br>
[root@workstation ~]# umount /mnt
<br>
[root@workstation ~]# ceph auth get-key client.admin > /root/asecret
<br>
[root@workstation ~]# mkdir /mnt/cephfs
<br>
[root@workstation ~]#  mount -t ceph ceph-mon01.example.com:6789:/ /mnt/cephfs -o name=admin,secretfile=/root/asecret
<br>
[root@workstation ~]# chown ceph:ceph /mnt/cephfs
<br>
[root@workstation ~]# df -hT /mnt/cephfs/
Filesystem           Type  Size  Used Avail Use% Mounted on
192.168.56.64:6789:/ ceph   38G     0   38G   0% /mnt/cephfs
<br>
[root@workstation ~]# umount /mnt/cephfs/
</pre>
</em>

_**Optional:**_ Now mount the same filesystem using the ceph-fuse client:

<em>
<pre>
[root@workstation ~]# dnf install ceph-fuse -y
<br>
[root@workstation ~]# mkdir /mnt/mycephfuse
<br>
[root@workstation ~]# ceph-fuse -n client.admin --client_fs fs_name /mnt/mycephfuse
ceph-fuse[3058]: starting ceph client
2025-05-14T14:43:05.211-0400 7fb63022f380 -1 init, newargv = 0x55933a88db10 newargc=15
ceph-fuse[3058]: starting fuse
<br>
[root@workstation-vncbz 1 ~]# mount | grep cephfuse
ceph-fuse on /mnt/mycephfuse type fuse.ceph-fuse (rw,nosuid,nodev,relatime,user_id=0,group_id=0,allow_other)
<br>
[root@workstation ~]# umount /mnt/mycephfuse
</pre>
</em>

#### <u>MANAGING RGW</u>

From `ceph-mon01` node run the following commands to configure RADOS Gateway:

<em>
<pre>
[root@ceph-mon01 ~]# ceph config set mon mon_max_pg_per_osd 512
<br>
[root@ceph-mon01 ~]# radosgw-admin realm create --rgw-realm=test_realm --default
<br>
{
    "id": "90c18949-f07a-4dcf-930c-c1ed637c5a4e",
    "name": "test_realm",
    "current_period": "3df3bf9d-1b2c-4f75-b768-b462689b069a",
    "epoch": 1
}
<br>
[root@ceph-mon01 ~]#  radosgw-admin zonegroup create --rgw-zonegroup=default  --master --default
{
    "id": "4cfa060d-1812-472f-a828-cac56adb9f5f",
    "name": "default",
    "api_name": "default",
    "is_master": "true",
    "endpoints": [],
    "hostnames": [],
    "hostnames_s3website": [],
    "master_zone": "",
    "zones": [],
    "placement_targets": [],
    "default_placement": "",
    "realm_id": "90c18949-f07a-4dcf-930c-c1ed637c5a4e",
    "sync_policy": {
        "groups": []
    },
    "enabled_features": [
        "compress-encrypted",
        "resharding"
    ]
}
<br>
[root@ceph-mon01 ~]# radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=test_zone --master --default
{
    "id": "a9be07a2-1ac7-46b9-8a64-b46a64558a64",
    "name": "test_zone",
    "domain_root": "test_zone.rgw.meta:root",
    "control_pool": "test_zone.rgw.control",
    "gc_pool": "test_zone.rgw.log:gc",
    "lc_pool": "test_zone.rgw.log:lc",
    "log_pool": "test_zone.rgw.log",
    "intent_log_pool": "test_zone.rgw.log:intent",
    "usage_log_pool": "test_zone.rgw.log:usage",
    "roles_pool": "test_zone.rgw.meta:roles",
    "reshard_pool": "test_zone.rgw.log:reshard",
    "user_keys_pool": "test_zone.rgw.meta:users.keys",
    "user_email_pool": "test_zone.rgw.meta:users.email",
    "user_swift_pool": "test_zone.rgw.meta:users.swift",
    "user_uid_pool": "test_zone.rgw.meta:users.uid",
    "otp_pool": "test_zone.rgw.otp",
    "system_key": {
        "access_key": "",
        "secret_key": ""
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "test_zone.rgw.buckets.index",
                "storage_classes": {
                    "STANDARD": {
                        "data_pool": "test_zone.rgw.buckets.data"
                    }
                },
                "data_extra_pool": "test_zone.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "realm_id": "90c18949-f07a-4dcf-930c-c1ed637c5a4e",
    "notif_pool": "test_zone.rgw.log:notif"
}
<br>
[root@ceph-mon01 ~]# radosgw-admin period update --rgw-realm=test_realm --commit
</pre>
</em>

#### <u>DEPLOYING RGW</u>

<em>
<pre>
[root@ceph-mon01 ~]# ceph orch apply rgw test --realm=test_realm --zone=test_zone --placement="2 ceph-mon02 ceph-mon03"
Scheduled rgw.test update...
</pre>
</em>

Ensure connection is possible:

<em>
<pre>
[root@ceph-mon01 ~]# curl http://192.168.56.65:80
<?xml version="1.0" encoding="UTF-8"?><ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Owner><ID>anonymous</ID><DisplayName></DisplayName></Owner><Buckets></Buckets></ListAllMyBucketsResult>
</pre>
</em>

Create a user to operate with RGW:

<em>
<pre>
[root@ceph-mon01 ~]# radosgw-admin user create --uid='user1' --display-name='First User' --access-key='S3user1' --secret-key='S3user1key'
{
    "user_id": "user1",
    "display_name": "First User",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "user1",
            "access_key": "S3user1",
            "secret_key": "S3user1key"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
</pre>
</em>

Create a subuser to use with Swift:

<em>
<pre>
[root@ceph-mon01 ~]# radosgw-admin subuser create --uid='user1' --subuser='user1:swift' --secret-key='Swiftuser1key' --access=full
</pre>
</em>

#### <u>TESTING RGW</u>

Now you can use `swift` command (from `python-swiftclient` library) or `s3cmd` to operate with RGW.

Go back to the `workstation` machine. If strict region enforcement is causing issues, you can enable relaxed region enforcement by setting the `rgw_relaxed_region_enforcement` option to `true` in your Ceph configuration. This allows for more flexibility in region constraints:

<em>
<pre>
[root@workstation ~]# ceph config set global rgw_relaxed_region_enforcement true
</pre>
</em>

To use the `s3cmd` utilitary, install it using `pip`:

<em>
<pre>
[root@workstation ~]# python -m pip install s3cmd
</pre>
</em>

To use the `swift` utilitary, install it using `pip`:

<em>
<pre>
[root@workstation ~]# python -m pip install --upgrade python-swiftclient
</pre>
</em>

To use the aws utilitary, install it using the following commands as per the official documentation:

<em>
<pre>
[root@workstation ~]# curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
<br>
[root@workstation ~]# unzip awscliv2.zip
<br>
[root@workstation ~]# ./aws/install
</pre>
</em>

Now configure it to use Ceph RGW:

<em>
<pre>
[root@workstation ~]# aws configure --profile=ceph
AWS Access Key ID [****************ser1]: S3user1
AWS Secret Access Key [****************1key]: S3user1key
Default region name [None]: 
Default output format [None]: 
</pre>
</em>

To test access and usability of RGW, create a bucket:

<em>
<pre>
[root@workstation ~]# aws --profile=ceph --endpoint=http://ceph-mon02.example.com s3 mb s3://testbucket
make_bucket: testbucket
<br>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 ls
2025-05-13 14:41:08 testbucket
<br>
[root@workstation ~]# radosgw-admin bucket list
[
    "testbucket"
]
<br>
[root@workstation ~]# radosgw-admin metadata get bucket:testbucket
{
    "key": "bucket:testbucket",
    "ver": {
        "tag": "_ZKKqirF8iCMEM8xaVW0RPeX",
        "ver": 1
    },
    "mtime": "2025-05-14T17:16:05.613540Z",
    "data": {
        "bucket": {
            "name": "testbucket",
            "marker": "707b118a-c136-4531-868e-f11e1b31f919.74184.1",
            "bucket_id": "707b118a-c136-4531-868e-f11e1b31f919.74184.1",
            "tenant": "",
            "explicit_placement": {
                "data_pool": "",
                "data_extra_pool": "",
                "index_pool": ""
            }
        },
        "owner": "user1",
        "creation_time": "2025-05-14T17:16:05.592882Z",
        "linked": "true",
        "has_bucket_info": "false"
    }
}
</pre>
</em>

Now create a file and upload it to the newly created bucket:

<em>
<pre>
[root@workstation ~]# echo "Ceph 5 Workshop RGW Testing File" > myfile.txt
<br>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 cp ./myfile.txt s3://testbucket/myfile.txt
upload: ./myfile.txt to s3://testbucket/myfile.txt
<br>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 ls s3://testbucket
2025-05-13 14:41:55         33 myfile.txt
<br>
[root@workstation ~]# radosgw-admin bucket list --bucket=testbucket
[
    {
        "name": "myfile.txt",
        "instance": "",
        "ver": {
            "pool": 10,
            "epoch": 1
        },
        "locator": "",
        "exists": "true",
        "meta": {
            "category": 1,
            "size": 33,
            "mtime": "2025-05-14T17:19:03.000287Z",
            "etag": "6701ec40f61c06a3cbe0ef0695b28a35",
            "storage_class": "",
            "owner": "user1",
            "owner_display_name": "First User",
            "content_type": "text/plain",
            "accounted_size": 33,
            "user_data": "",
            "appendable": "false"
        },
        "tag": "707b118a-c136-4531-868e-f11e1b31f919.74131.5428523535151071339",
        "flags": 0,
        "pending_map": [],
        "versioned_epoch": 0
    }
]
</pre>
</em>

Delete the local copy of the file and then download the version available within the bucket:

<em>
<pre>
[root@workstation ~]# rm -f myfile.txt
<br>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 cp s3://testbucket/myfile.txt ./myfile.txt
download: s3://testbucket/myfile.txt to ./myfile.txt
<br>
[root@workstation ~]# cat myfile.txt
Ceph 5 Workshop RGW Testing File
</pre>
</em>

Finally, remove the file and the bucket:

<em>
<pre>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 rm s3://testbucket/myfile.txt
delete: s3://testbucket/myfile.txt
<br>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 ls s3://testbucket/
<br>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 rb s3://testbucket/
remove_bucket: testbucket
<br>
[root@workstation ~]# aws --profile=ceph --endpoint-url=http://ceph-mon02.example.com s3 ls
</pre>
</em>

_**Optional:**_ To run the same tests using the `swift` client, do the following:

<em>
<pre>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key stat
                                    Account: v1
                                 Containers: 0
                                    Objects: 0
                                      Bytes: 0
Objects in policy "default-placement-bytes": 0
  Bytes in policy "default-placement-bytes": 0
   Containers in policy "default-placement": 0
      Objects in policy "default-placement": 0
        Bytes in policy "default-placement": 0
                                X-Timestamp: 1747243975.71445
                X-Account-Bytes-Used-Actual: 0
                                 X-Trans-Id: tx0000031607f50a70bb4b6-006824d3c7-12193-test_zone
                     X-Openstack-Request-Id: tx0000031607f50a70bb4b6-006824d3c7-12193-test_zone
                              Accept-Ranges: bytes
                               Content-Type: text/plain; charset=utf-8
                                 Connection: Keep-Alive
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key post testbucket
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key stat
                                    Account: v1
                                 Containers: 1
                                    Objects: 0
                                      Bytes: 0
Objects in policy "default-placement-bytes": 0
  Bytes in policy "default-placement-bytes": 0
   Containers in policy "default-placement": 1
      Objects in policy "default-placement": 0
        Bytes in policy "default-placement": 0
                                X-Timestamp: 1747244027.32031
                X-Account-Bytes-Used-Actual: 0
                                 X-Trans-Id: tx00000ff006e10ee765b9d-006824d3fb-12193-test_zone
                     X-Openstack-Request-Id: tx00000ff006e10ee765b9d-006824d3fb-12193-test_zone
                              Accept-Ranges: bytes
                               Content-Type: text/plain; charset=utf-8
                                 Connection: Keep-Alive
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key list
testbucket
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key upload testbucket myfile.txt 
myfile.txt
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key stat testbucket
                      Account: v1
                    Container: testbucket
                      Objects: 1
                        Bytes: 33
                     Read ACL:
                    Write ACL:
                      Sync To:
                     Sync Key:
                  X-Timestamp: 1747244024.20742
X-Container-Bytes-Used-Actual: 4096
             X-Storage-Policy: default-placement
              X-Storage-Class: STANDARD
                Last-Modified: Wed, 14 May 2025 17:36:25 GMT
                   X-Trans-Id: tx00000d702f04027cde12e-006824d4b6-12193-test_zone
       X-Openstack-Request-Id: tx00000d702f04027cde12e-006824d4b6-12193-test_zone
                Accept-Ranges: bytes
                 Content-Type: text/plain; charset=utf-8
                   Connection: Keep-Alive

[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key list testbucket
myfile.txt
<br>
[root@workstation ~]# rm -rf myfile.txt
<br>
[root@workstation-vncbz 1 ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key download testbucket myfile.txt 
myfile.txt [auth 0.009s, headers 0.013s, total 0.013s, 0.007 MB/s]
<br>
[root@workstation ~]# cat myfile.txt 
Ceph 5 Workshop RGW Testing File
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key delete testbucket myfile.txt
myfile.txt
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key list testbucket
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key stat testbucket
                      Account: v1
                    Container: testbucket
                      Objects: 0
                        Bytes: 0
                     Read ACL:
                    Write ACL:
                      Sync To:
                     Sync Key:
                  X-Timestamp: 1747244024.20742
X-Container-Bytes-Used-Actual: 0
             X-Storage-Policy: default-placement
              X-Storage-Class: STANDARD
                Last-Modified: Wed, 14 May 2025 17:36:25 GMT
                   X-Trans-Id: tx000007305c21eb201768a-006824d4f6-12193-test_zone
       X-Openstack-Request-Id: tx000007305c21eb201768a-006824d4f6-12193-test_zone
                Accept-Ranges: bytes
                 Content-Type: text/plain; charset=utf-8
                   Connection: Keep-Alive
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key delete testbucket
testbucket
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key list
<br>
[root@workstation ~]# swift -A http://ceph-mon02.example.com/auth/1.0 -U user1:swift -K Swiftuser1key stat
                                    Account: v1
                                 Containers: 0
                                    Objects: 0
                                      Bytes: 0
   Containers in policy "default-placement": 0
      Objects in policy "default-placement": 0
        Bytes in policy "default-placement": 0
Objects in policy "default-placement-bytes": 0
  Bytes in policy "default-placement-bytes": 0
                                X-Timestamp: 1747244446.58349
                X-Account-Bytes-Used-Actual: 0
                                 X-Trans-Id: tx00000f97f91006616fdb3-006824d59e-12193-test_zone
                     X-Openstack-Request-Id: tx00000f97f91006616fdb3-006824d59e-12193-test_zone
                              Accept-Ranges: bytes
                               Content-Type: text/plain; charset=utf-8
                                 Connection: Keep-Alive
</pre>
</em>
