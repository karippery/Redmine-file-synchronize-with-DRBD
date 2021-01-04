# Redmine-file-synchronize-with-DRBD
synchronize file folder of redmine using DRBD 
###### please make sure your servers already setup pacemaker and redmine

### install packages 

      sudo apt-get update sudo apt-get install -y drbd8-utils

We should also make sure that NTP, Network Time Protocol is installed for accurate time:

           sudo apt-get install -y ntp
           
Next, on each host edit the /etc/drbd.conf

      global { usage-count no; }
      common { protocol C;}
      resource r0 {
              on oreo {
                      device /dev/drbd0;
                      disk /dev/sda3;
                      address 10.226.777.222:7788;
                      meta-disk internal;
                      }
              on cupcone  {
                      device /dev/drbd0;
                      disk /dev/sda3;
                      address 10.226.777.222:7788;
                      meta-disk internal;
                      }
              }
              
 Still working on all systems we need to load the kernel module with:
 
                 sudo modprobe drbd

Create the mirror device:

             sudo drbdadm create-md r0
             
And then bring the mirror device online

            sudo drbdadm up r0
            
To view the status of the mirroring we can use one of the following commands

             sudo drbd-overview
             sudo cat /etc/drbd
We will see both nodes a secondary and the data to be inconsistent. We can correct this by forcing
the Primary role on one node. Only do this on one node!

This can take many hours on large disk, a few minutes on this 1 GB disk. To view the progress:

          sudo drbdadm -- --overwrite-data-of-peer primary r0/0
 mount folder to disk 
 
    mount /dev/drbd0 /usr/share/redmine/files

 
 ### SETUP filesystem in pacemaker
 
         pcs cluster cib drbd_cfg

        pcs -f drbd_cfg resource create Test ocf:linbit:drbd  drbd_resource=r0 op monitor interval=60s

        pcs -f drbd_cfg resource master TestDataClone Test master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true

        pcs cluster cib-push drbd_cfg --config
        
### Configure the Cluster for the Filesystem
 
             pcs cluster cib fs_cfg

        pcs -f fs_cfg resource create TestFS ocf:heartbeat:Filesystem device="/dev/drbd0" directory="/usr/share/redmine/files" fstype="ext4" 

        pcs -f fs_cfg constraint colocation add TestFS with TestDataClone INFINITY with-rsc-role=Master

        pcs -f fs_cfg constraint order promote TestDataClone then start TestFS

        pcs -f fs_cfg constraint colocation add Apache with TestFS INFINITY

        pcs -f fs_cfg constraint order TestFS then Apache

        pcs -f fs_cfg constraint

        pcs cluster cib-push fs_cfg --config

        
 
