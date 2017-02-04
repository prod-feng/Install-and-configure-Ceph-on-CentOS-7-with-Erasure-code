# Install-and-configure-Ceph-on-CentOS-7-with-Erasure-code

The following lists the steps I used to set CephFS on a single PC, just for test purpose. I have been curious that whether 
I can use the erasure code pool of Ceph to set a RAID-like local drive on a PC for home use.

(1) Set up loop devices
I used loop devices on my PC to set Ceph Jewel 10.2 on CentOS 7. To set the loop devices, I first set 9 1GB filse 
using dd command:

>for var in 21 22 23 24 25 26 27 28 29
>
>do
>
>  dd if=/dev/zero of=/data/users/hd${var}.img bs=1024K count=1024
>
>done

And then:


>[ceph@master my-cluster]$losetup  -o 1048576  /dev/loop21 /data/users/hd21.img
>
> ...
>
>[ceph@master my-cluster]$losetup  -o 1048576  /dev/loop29 /data/users/hd29.img


After that, running "losetup" command to check the status of these loop devices:

> [ceph@master my-cluster]$losetup
>     
>     NAME        SIZELIMIT  OFFSET AUTOCLEAR RO BACK-FILE
>     /dev/loop21         0 1048576         0  0 /data/users/hd21.img
>     /dev/loop22         0 1048576         0  0 /data/users/hd22.img
>     ...
>     /dev/loop29         0 1048576         0  0 /data/users/hd29.img
     
     
(2) Install and configure Ceph

There are a lot of documents online on how to install and configure Ceph on CentOS. 
The official doc can be found at: http://docs.ceph.com/docs/master/install/

I used "ceph-deploy" to do the installation of Ceph. Here I ignore the pre-setup steps, such as user account, networking,etc., 
which can be found in the above link.

> [ceph@master my-cluster]$ceph-deploy new master

Here, "master" is the name of my computer. 
Then modify the "ceph.conf" file to enable Ceph to be installed on a single node.


> [ceph@master my-cluster]$vi ceph.conf
(to be added)

Then,
  
>[ceph@master my-cluster]$ceph-deploy install master

>[ceph@master my-cluster]$ceph-deploy mon create-initial

Now set the OSDs:


>[ceph@master my-cluster]$ceph-deploy osd prepare master:/dev/loop21 master:/dev/loop22 master:/dev/loop23 master:/dev/loop24 master:/dev/loop25 master:/dev/loop26 master:/dev/loop27 master:/dev/loop28 master:/dev/loop29


>[ceph@master my-cluster]$ceph-deploy osd activate master:/dev/loop21p1 master:/dev/loop22p1 master:/dev/loop23p1 master:/dev/loop24p1 master:/dev/loop25p1 master:/dev/loop26p1 master:/dev/loop27p1 master:/dev/loop28p1 master:/dev/loop29p1


Now, set the admin,rgw and mds service, all on local PC:

> [ceph@master my-cluster]$ceph-deploy admin master


> [ceph@master my-cluster]$sudo chmod +r /etc/ceph/ceph.client.admin.keyring



>[ceph@master my-cluster]$ceph-deploy rgw create master

>[ceph@master my-cluster]$ceph-deploy mds create master

Now the OSDs should have been mounted locally:
>[ceph@master my-cluster]$df -h

>     ...
>     /dev/loop21p1            507M   29M  479M   6% /var/lib/ceph/osd/ceph-0
>     ...
>     /dev/loop29p1            507M   28M  480M   6% /var/lib/ceph/osd/ceph-8
>     
>     

>[ceph@master my-cluster]$ ceph -s
>         
>         cluster c735fc20-fe09-4815-8777-3ab7b6dd461a
>          health HEALTH_WARN
>                 too few PGs per OSD (23 < min 30)
>          monmap e1: 1 mons at {master=191.168.0.9:6789/0}
>                 election epoch 3, quorum 0 master
>          osdmap e74: 9 osds: 9 up, 9 in
>                 flags sortbitwise,require_jewel_osds
>           pgmap v157: 104 pgs, 6 pools, 1588 bytes data, 171 objects
>                 249 MB used, 4310 MB / 4559 MB avail
>                      104 active+clean
>          



Check the default erasure code profile Ceph uses:
>
> [ceph@master my-cluster]$ceph osd erasure-code-profile get default
>     
>      k=2
>      m=1
>      plugin=jerasure
>      technique=reed_sol_van
>      

I will set my own erasure code profile, named "myprofile". Before that, I will set the ruleset to be per OSD.
>
>[ceph@master my-cluster]$ceph osd erasure-code-profile set myprofile k=6 m=3 ruleset-failure-domain=osd
>

>[ceph@master my-cluster]$ceph osd erasure-code-profile get myprofile
>     
>     jerasure-per-chunk-alignment=false
>     k=6
>     m=3
>     plugin=jerasure
>     ruleset-failure-domain=osd
>     ruleset-root=default
>     technique=reed_sol_van
>     w=8
>     





Now I will set 2 erasure pools using the above profile:


>[ceph@master my-cluster]$ceph osd pool create mdspool 256 256 erasure myprofile
>
>[ceph@master my-cluster]$ceph osd pool create datapool 256 256 erasure myprofile


and then, I will set 2 replicate pool as tier pools for later use to set CepfFS using the above 2 erasure pools:



>[ceph@master my-cluster]$ceph osd pool create bbbpool 12
>
>[ceph@master my-cluster]$ceph osd pool create bbbpool 12
>
>
>[ceph@master my-cluster]$ceph osd tier add mdspool aaapool
>
>[ceph@master my-cluster]$ceph osd tier add datapool bbbpool
>
>[ceph@master my-cluster]$ceph osd tier cache-mode aaapool writeback
>
>[ceph@master my-cluster]$ceph osd tier cache-mode bbbpool writeback
>
>[ceph@master my-cluster]$ceph osd tier set-overlay mdspool aaapool
>
>[ceph@master my-cluster]$ceph osd tier set-overlay  datapool bbbpool
>
>[ceph@master my-cluster]$ceph osd pool ls detail
> 
  
>     

>      pool 0 'rbd' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 77 flags hashpspool stripe_width 0
>      
>      pool 1 '.rgw.root' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 55 owner 18446744073709551615 flags hashpspool stripe_width 0
>      
>      pool 2 'default.rgw.control' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 57 owner 18446744073709551615 flags hashpspool stripe_width 0
>      
>      pool 3 'default.rgw.data.root' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 59 owner 18446744073709551615 flags hashpspool stripe_width 0
>      
>      pool 4 'default.rgw.gc' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 60 owner 18446744073709551615 flags hashpspool stripe_width 0
>      
>      pool 5 'default.rgw.log' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 61 owner 18446744073709551615 flags hashpspool stripe_width 0
>      
>      pool 12 'default.rgw.users.uid' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 8 pgp_num 8 last_change 89 owner 18446744073709551615 flags hashpspool stripe_width 0
>      
>      pool 22 'mdspool' erasure size 9 min_size 6 crush_ruleset 3 object_hash rjenkins pg_num 12 pgp_num 12 last_change 205 lfor 205 flags hashpspool tiers 24 read_tier 24 write_tier 24 stripe_width 4224
> 
>      pool 23 'datapool' erasure size 9 min_size 6 crush_ruleset 4 object_hash rjenkins pg_num 12 pgp_num 12 last_change 206 lfor 206 flags hashpspool tiers 25 read_tier 25 write_tier 25 stripe_width 4224
>      
>              removed_snaps [1~3]
>      
>      pool 24 'aaapool' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 12 pgp_num 12 last_change 205 flags hashpspool,incomplete_clones tier_of 22 cache_mode writeback stripe_width 0

>      pool 25 'bbbpool' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 12 pgp_num 12 last_change 206 flags hashpspool,incomplete_clones tier_of 23 cache_mode writeback stripe_width 0
>      
>              removed_snaps [1~3]
>      
>      

Now I can set RBD block device using my erasure pool:
> 
> [ceph@master my-cluster]$rbd create --size 2G datapool/myvolume
> 



>  [ceph@master my-cluster]$rbd ls datapool
>      
>      myvolume
>      
> 
>  [ceph@master my-cluster]$rbd --image datapool/myvolume info
>      
>      rbd image 'myvolume':
>              size 2048 MB in 512 objects
>              order 22 (4096 kB objects)
>              block_name_prefix: rbd_data.11742ae8944a
>              format: 2
>              features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
>              flags:
>      
>      

In the following, I will setup a CephFS using the 2 erasure pools wit 2 tier pools:
> 
> [ceph@master my-cluster]$ceph fs new cephfs mdspool datapool
>      
>      new fs with metadata pool 22 and data pool 23
>      



Then, mount it:
> 
> [ceph@master my-cluster]$cat ceph.client.admin.keyring
>      
>      [client.admin]
>      
>              key = AQBxZotYqZjdOBAAMxNx/pbH0AaxxEJp9CXtpA==
>              caps mds = "allow *"
>              caps mon = "allow *"
>              caps osd = "allow *"
>      

> [ceph@master my-cluster]$sudo mount -t ceph master:6789:/ /mnt/cephfs -o name=admin,secret=AQBxZotYqZjdOBAAMxNx/pbH0AaxxEJp9CXtpA==
> 
> [ceph@master my-cluster]$df -h
>      
>      Filesystem               Size  Used Avail Use% Mounted on
>     ...
>      /dev/loop29p1            507M   33M  475M   7% /var/lib/ceph/osd/ceph-8
>      192.168.0.9:6789:/     4.5G  308M  4.2G   7% /mnt/cephfs
>      

Now test the CepfFS:
> 
> [ceph@master my-cluster]$sudo mkdir /mnt/cephfs/ceph
> 
> [ceph@master my-cluster]$sudo chown ceph:ceph /mnt/cephfs/ceph
> 
> [ceph@master my-cluster]$history >/mnt/cephfs/ceph/as.log
> 
> [ceph@master my-cluster]$tail /mnt/cephfs/ceph/as.log
> 
>        837  ls -l /mnt/
>        838  ls -l /mnt/cephfs/
>        839  mkdir /mnt/cephfs/ceph
>        840  sudo mkdir /mnt/cephfs/ceph
>        841  sudo chown ceph:ceph /mnt/cephfs/ceph
>        842  history >/mnt/cephfs/ceph/as.log
>     

That's it.

Please NOTE: to install and configure CephFS on a single PC, I did some tricks which are not suitable for formal 
production environment. For example, the OSDs should be distributes on different nodes, other than one a single node; 
the tier pool should use fast disks, like SSD.




