---
title: Ceph RBD Bare-Metal Install for Kubernetes (on Packet.net)
date: 2016-05-24T20:59:47+00:00
tags:
  - Ansible
  - Bare-Metal
  - Ceph
  - Container
  - Docker
  - Kubernetes
  - Packet.net
draft: false
---

In this guide, we&#8217;ll create a bare-metal Ceph RBD cluster which may provide persistent volume support for a Kubernetes environment.

Our Ceph RBD cluster will be composed of a single Ceph monitor (MON) and two Ceph Object Storage Daemon (OSD) nodes. In Ceph, MON nodes track the state of the cluster, and OSD nodes hold the data to be persisted.

## Kubernetes and the Need for Ceph RBD

Creating a Kubernetes cluster environment equivalent to hosted solutions (GKE) and turn-key solutions (Kubernetes on AWS or GCE) requires 3 things:

  1. The Kubernetes System Itself: Masters and Nodes ([Setup Guide][1])
  2. **Persistent Storage** (_This_ Guide)
  3. External Load-Balancers (Future Guide)

Use this guide to implement persistent storage to make a Kubernetes environment suitable for hosting stateful applications. When Pods of persistent applications such as databases fail, the Kubernetes system directs the Pod to migrate from one Node to another. The data is often left behind and lost&#8230; unless it was stored on a Kubernetes volume backed by a network block device. In this case Kubernetes will automatically migrate the Volume mount along with the Pod to the new destination Node.

[Ceph RBD][2] may be used to create a redundant, highly available storage cluster that provides network-mountable block storage devices, similar to that of Rackspace Cloud Block Storage and Amazons EBS (minus the API). Ceph itself is a large project which also provides network-mountable posix filesystems (CephFS) and network object storage like S3 or Swift. However, for Kubernetes we only need to deploy a subset of Ceph&#8217;s functionality&#8230; Ceph RBD.

This guide should work on any machines, whether bare-metal, virtual, or cloud, so long as the following criterial are met:

  * All Ceph cluster machines are running Ubuntu 14.04 LTS 
      * Centos7 may work with yum-plugin-prorities disabled.
  * All machines are network reachable, without restriction 
      * i.e. open iptables, open security groups
  * Root access through password-less SSH is enabled 
      * i.e. configured /root/.ssh/authorized_keys

Because not many of us have multiple bare-metal machines laying around, we&#8217;ll rent them from [packet.net][3]. If you already have machines provisioned, you may skip the provisioning section below.

## Step Summary

  * Setup a Development Environment
  * Provision Bare-Metal Machines (on packet.net)
  * Configure and Run the Ceph [ceph/ceph-ansible][4] scripts
  * Test the Cluster by Creating Storage Volumes
  * Configure Remote Kubernetes Nodes to Use Ceph RBD Volumes

## Setup a Development Environment

The easiest method is to instantiate a [vagrant machine][5] with all of the necessary tools. Feel free to do this manually. The vagrant environment includes the following:

  * Docker
  * Kubectl
  * Ansible
  * Hashicorp Terraform

Fetch the vagrant machine

    # Tested at dcwangmit01/vagrant-devenv - branch master, commit abe7d520
    git clone git@github.com:dcwangmit01/vagrant-devenv
    cd vagrant-devenv
    

If you are on OSX, ensure and install the host dependencies.

    # Installs brew, virtualbox, vagrant, etc.
    make deps
    

Create and connect to the vagrant environment

    # Start ssh-agent and load your key
    eval `ssh-agent`; ssh-add
    
    # Create the vagrant machine
    vagrant up
    
    # Connect to the box
    vagrant ssh
    

&#42; From now on, do everything from within this vagrant machine.

## Provision Packet.net Bare-Metal Machines

The [ceph/ceph-ansible][4] scripts
  
work on Ubuntu 14.04, but fail on Centos 7 unless yum-plugin-priorities are disabled.

If you enjoy clicking around WebUI&#8217;s, follow the manual instructions below. Otherwise, the only automated provisioning method supported by [packet.net][3] is to use Hashicorp Terraform. A CLI client does not yet exist.

### Manual WebUI Instructions

  * Login to the web UI at [packet.net][3]
  * Create a project
  * Set an SSH key (which will be provisioned on new servers)
  * Create 3 servers with Ubuntu 14.04 
      * ceph-mon-0001
      * ceph-osd-0001
      * ceph-osd-0002
  * Follow the Create and Attach Disks section below
  * Note the names and IPs of the newly created machines

### Semi-Automatic Instructions (Using Terraform)

Instantiating servers via Hashicorp Terraform must happen in two steps if a &#8220;packet&#95;project&#8221; has not yet been created.

This is because the &#8220;packet&#95;device&#8221; (aka. bare metal machine) definitions require a &#8220;project&#95;id&#8221; upon execution. However, one does not know the &#8220;project&#95;id&#8221; until after the &#8220;packet&#95;project&#8221; has been created. One might resolve this issue in the future by having the packet&#95;device require a project&#95;name instead of a project&#95;id. Then we could &#8220;terraform apply&#8221; this file in one go.

See the Appendix for curl commands that will help you discover project&#95;ids from the API.

#### Step 1: Create the Packet.net Project

First, create your auth token via the [packet.net][3] UI and note it down.

Create a cluster.tf, and tweak the {variables} according to your needs.

    # Configure the Packet Provider
    provider "packet" {
            auth_token = "{REPLACE_WITH_AUTH_TOKEN}"
    }
    
    # Create a new SSH key
    #   You may have to stage an ssh key on the vagrant box,
    #     or run "ssh-keygen -b 4096" to create a new key.
    #   This key specified below will be auto-added to the 
    #     /root/.ssh/authorized_keys of newly provisioned machines.
    resource "packet_ssh_key" "{REPLACE_WITH_A_NAME_FOR_SSH_KEY}" {
        name = "{REPLACE_WITH_A_NAME_FOR_SSH_KEY}"
        public_key = "${file("/{REPLACE_WITH_PATH_TO_SSH_KEY}")}"
    }
    
    # Create a project
    resource "packet_project" "{REPLACE_WITH_A_NAME_FOR_CLUSTER}" {
            name = "{REPLACE_WITH_A_NAME_FOR_CLUSTER}"
    }
    

Run Terraform to create the project.

    terraform plan -out cluster
    terraform apply
    

#### Step 2: Create the Packet.net Machines

Locate your PACKET&#95;PROJECT&#95;ID, so that you may embed it in the cluster.tf Terraform file.

    # Locate your API Key from the Web Portal https://app.packet.net/
    export PACKET_AUTH={REPLACE_WITH_AUTH_TOKEN}
    
    # Get a list of projects
    curl -s -H  "X-Auth-Token: $PACKET_AUTH" https://api.packet.net/projects |python -m json.tool
    
    # Search the output above for the PACKET_PROJECT_ID
    #   and use it in the packet_device definitions below
    

Append to cluster.tf, and tweak the {variables} according to your needs. As of May 2016, only the ewr1 packet.net facility supports disk creation. Because of this, ensure all machines and disks are instantiated in facility ewr1.

    # Create a device
    resource "packet_device" "ceph-mon-0001" {
            hostname = "ceph-mon-0001"
            plan = "baremetal_0"
            facility = "ewr1"
            operating_system = "ubuntu_14_04"
            billing_cycle = "hourly"
            project_id = "{REPLACE_WITH_PACKET_PROJECT_ID}"
    }
    
    resource "packet_device" "ceph-osd-0001" {
            hostname = "ceph-osd-0001"
            plan = "baremetal_0"
            facility = "ewr1"
            operating_system = "ubuntu_14_04"
            billing_cycle = "hourly"
            project_id = "{REPLACE_WITH_PACKET_PROJECT_ID}"
    }
    
    resource "packet_device" "ceph-osd-0002" {
            hostname = "ceph-osd-0002"
            plan = "baremetal_0"
            facility = "ewr1"
            operating_system = "ubuntu_14_04"
            billing_cycle = "hourly"
            project_id = "{REPLACE_WITH_PACKET_PROJECT_ID}"
    }
    

Run Terraform

    terraform plan -out cluster
    terraform apply
    

Note the names and IPs of the newly created machines

    cat terraform.tfstate|grep -E '(hostname|network.0.address)'
    # Returns something like this
    # "hostname": "ceph-mon-0001",
    # "network.0.address": "147.75.199.133",
    # "hostname": "ceph-osd-0001",
    # "network.0.address": "147.75.192.205",
    # "hostname": "ceph-osd-0002",
    # "network.0.address": "147.75.197.189",
    

Create an ~/.ssh/config file with the hostname to IP mappings. This allows you to refer to the machines via hostnames from your Dev box, as well as Ansible inventory definitions.

    Host ceph-mon-0001
      User root
      Hostname {REPLACE_WITH_IP_OF_ceph-mon-0001}
    Host ceph-osd-0001
      User root
      Hostname {REPLACE_WITH_IP_OF_ceph-osd-0001}
    Host ceph-osd-0002
      User root
      Hostname {REPLACE_WITH_IP_OF_ceph-osd-0002}
    

#### Step 3: Create and Attach Disks

This part is manual for now, using the [packet.net][3] WebUI.

For each OSD node

  * Using the WebUI, create a 100GB disk and name it the be the same as the respective OSD node (e.g. ceph-osd-0001)
  * Using the WebUI, attach the disk to the respective OSD node
  * Using SSH, connect to the OSD node and run the on-machine attach command

```
# Connect to the box
ssh root@ceph-osd-{id}
# This will attach the disk and create a new entry under /dev/mapper/volume-{id}
packet-block-storage-attach
```

  * Using SSH, connect to the OSD node and rename the attached volume to be a consistent name. The Ansible scripts we run later will expect the same volume names (e.g. /dev/mapper/data01) and the same number of volumes across all of the OSD nodes.

```
# Connect to the box
ssh root@ceph-osd-{id}
# Rename the newly mapped volume from /dev/mapper/volume-{id} to /dev/mapper/data01
dmsetup rename /dev/mapper/volume-{id} data01
```

You may add additional disks to each OSD machine, so long as the number of and naming of disks on across all of the OSD machines are the consistent.

## Configure and Run the ceph/ceph-ansible scripts

Fetch the ceph/ceph-ansible code

    # Tested at ceph/ceph-ansible - branch master, commit 84dad9a7
    git clone git@github.com:ceph/ceph-ansible.git
    cd ceph-ansible
    
    # Work in your own branch
    git checkout master
    git checkout -b bare-metal
    
    # Set links from the sample configs to your own configs
    #   This allows you to "git diff master" later to see your config changes
    ln -s site.yml.sample site.yml
    pushd group_vars
    ln -s all.sample all
    ln -s mons.sample mons
    ln -s osds.sample osds
    popd
    

Create an Ansible inventory file name &#8220;inventory&#8221;, and use the hosts and IPs recorded prior.

    # configure monitor IPs or Names
    [mons]
    ceph-mon-0001
    
    # configure object storage daemon IPs or Names
    [osds]
    ceph-osd-0001
    ceph-osd-0002
    
    # configure clients
    #   Setup your local vagrant machine as a ceph client.
    #   If you have a Kubernetes cluster that needs access to 
    #     Ceph volumes, add the nodes here as well.
    [clients]
    localhost ansible_connection=local
    # kube-node-0001
    # kube-node-0002
    

Test the ansible connection, and accept host-keys into your ~/.ssh/knownhosts

    ansible all -u root -i inventory -m ping
    

Configure and edit **group_vars/all**. You may add the following
  
settings, or replace/uncomment existing lines:

    #####################################################################
    # Custom Configuration
    
    # Set Ansible to use the "root" user for remote management
    ansible_ssh_user: root
    
    # Set ceph-ansible to use the latest stable ceph release, rather than
    #   default packages from any Linux Distribution
    ceph_stable: true # use ceph stable branch
    ceph_stable_key: https://download.ceph.com/keys/release.asc
    ceph_stable_release: jewel # ceph stable release
    ceph_stable_repo: "http://download.ceph.com/debian-{{ ceph_stable_release }}"
    
    # Set ceph-ansible to use listen on the bond0 interface.
    #   This is very specific to packet.net, whose primary interface is bond0.
    #   Other distributions may use eth0.
    monitor_interface: bond0
    
    # Set ceph-ansible journal_size (arbitrarily chosen)
    journal_size: 256
    
    # Ceph OSDs may be setup to use the public_network for servicing
    #   clients (volume access, etc), and a cluster_network for OSD data
    #   replication.  In our case on packet.net, we set these to be the
    #   same.  Packet.net machines have a single bond0 interface (2 NICs
    #   ganged as one) with a public internet address.  There is also a
    #   private IP alias created on the same interface that seems to
    #   simulate a "Rackspace servicenet" type of private network.
    public_network: 10.0.0.0/8
    cluster_network: 10.0.0.0/8
    
    # Set the type of filesystem that Ceph should use for underlying Ceph
    #   RBD storage.
    osd_mkfs_type: xfs
    osd_mkfs_options_xfs: -f -i size=2048
    osd_mount_options_xfs: noatime,largeio,inode64,swalloc
    osd_objectstore: filestore
    

Configure and edit **groups_vars/osds**. You may add the following
  
settings, or replace/uncomment existing lines:

    #####################################################################
    # Custom Configuration
    
    # Set ceph-ansible to use the "data01" device across all OSD nodes.
    #   If you have added additional disks, you may include them below.
    #   Beware that we do not enable auto-discovery of disks.
    #      Ceph-ansible does not recognize handle multipath disks.
    #      Thus, the individual two /dev/sd{a,b} disks will be discovered
    #      instead of the single proper volume under
    #      /dev/mapper/volume-{id}.
    devices:
      - /dev/mapper/data01
    # - /dev/mapper/data[XX] if you have added additional disks
    
    # Set ceph-ansible to place the journal on the same device as the data
    #   disk.  This is usually suboptimal for performace, but in our case
    #   we only have a single disk.
    journal_collocation: true
    

Check if any &#8220;inventory&#8221; machine targets are running Centos 7. Disable yum-plugin-priorities, or else the ceph-ansible scripts will fail. If enabled, the scripts will fail to locate packages and dependencies because other repositories with the older versions of packages will take precedence over the ceph repository.

    # Connect to the machine
    ssh root@{centos7_machine}
    # Set the flag to disable yum-plugin-priorities
    sed -i 's@enabled = 1@enabled = 0@' /etc/yum/pluginconf.d/priorities.conf
    

Run the ansible setup script to install Ceph on all machines

    ansible-playbook site.yml -i inventory
    

&#42; Run this more than once until zero errors. It&#8217;s idempotent!

## Test the Cluster by Creating Storage Volumes

### Ensure a Healthy Ceph RBD Cluster

Your local vagrant machine should now have the Ceph CLI tools installed and configured. If for some reason the tools are not working, try the following from the Ceph MON machine.

Query Ceph cluster status

    ceph -s
    #    cluster d1eab1dc-a050-46a4-9771-f2494714b96e
    #     health HEALTH_WARN
    #            64 pgs degraded
    #            64 pgs stuck unclean
    #            64 pgs undersized
    #     monmap e1: 1 mons at {ceph-mon-0001=147.75.196.123:6789/0}
    #            election epoch 3, quorum 0 ceph-mon-0001
    #     osdmap e9: 2 osds: 2 up, 2 in
    #            flags sortbitwise
    #      pgmap v18: 64 pgs, 1 pools, 0 bytes data, 0 objects
    #            72528 kB used, 199 GB / 199 GB avail
    #                  64 active+undersized+degraded
    

You can see that the cluster is working, but in a WARN state. Let us debug this.

List all osd pools, and find only the default &#8220;rbd&#8221; pool.

    # list all OSD pools
    ceph osd pool ls
    # rbd
    

List the parameters of the default &#8220;rbd&#8221; osd pool.

    ceph osd pool get rbd all
    # size: 3
    # min_size: 2
    # crash_replay_interval: 0
    # pg_num: 64
    # pgp_num: 64
    # crush_ruleset: 0
    # hashpspool: true
    # nodelete: false
    # nopgchange: false
    # nosizechange: false
    # write_fadvise_dontneed: false
    # noscrub: false
    # nodeep-scrub: false
    # use_gmt_hitset: 1
    # auid: 0
    # min_write_recency_for_promote: 0
    # fast_read: 0
    

Notice that `size == 3` which means that a healthy state requires 3 replicas for all placement groups within the &#8220;rbd&#8221; pool. However, we only have 2 OSDs, and thus are running in a degraded HEALTH_WARN state. If we decrease the number of required replicas, the cluster will return to a healthy state.

    # set redundancy of rbd default pool from 3 to 2, because we only have 2 OSDs
    ceph osd pool set rbd size 2
    

Check that Ceph cluster status has returned to HEALTH_OK. This may take a few seconds.

    ceph -s
    #     cluster d1eab1dc-a050-46a4-9771-f2494714b96e
    #      health HEALTH_OK
    #      monmap e1: 1 mons at {ceph-mon-0001=147.75.196.123:6789/0}
    #             election epoch 3, quorum 0 ceph-mon-0001
    #      osdmap e11: 2 osds: 2 up, 2 in
    #             flags sortbitwise
    #       pgmap v26: 64 pgs, 1 pools, 0 bytes data, 0 objects
    #             73432 kB used, 199 GB / 199 GB avail
    #                   64 active+clean
    

### Create a Volume for Testing

We&#8217;ll test the Ceph cluster from the OSD nodes as Ceph clients. According to the earlier configuration:

`public_network == cluster_network == 10.0.0.0/8`

Ceph will only listen to requests on the 10.0.0.0/8 network, which is a pseudo-servicenet alias for a private network between the machines within the same packet.net project. (_Note: Perhaps this may not work as expected. It seems that the ceph-monitor process is bound to the IP on the public network, which must be used in the Kubernetes Volume definitions_).

Copy the credentials from the Ceph MON to the Ceph OSDs, so that they will be able to access the Ceph cluster as clients

    scp root@ceph-mon-0001:/etc/ceph/ceph.client.admin.keyring .
    scp ceph.client.admin.keyring root@ceph-osd-0001:/etc/ceph/ceph.client.admin.keyring
    scp ceph.client.admin.keyring root@ceph-osd-0002:/etc/ceph/ceph.client.admin.keyring
    

From the first OSD node, create a volume, mount it, add data, and unmount the volume.

    # SSH to the first Ceph OSD node where we will simulate a client
    ssh root@ceph-osd-0001
    
    # Create an RBD device and specify *only* the layering feature
    #   Man Page: http://docs.ceph.com/docs/master/man/8/rbd/
    #   Many default features based on exclusive-lock are not supported
    #     by kernel mounts of non-cutting-edge operating systems.  Thus,
    #     we dumb down the feature set to only support layering.  Older
    #     kernels are able to support mounting the default image-feature's
    #     via the userspace utility "rbd-nbd" (apt-get install -y rbd-nbd; 
    #     rbd-nbd map foo).
    
    # Create the volume "vol01" within the "rbd" pool
    rbd create rbd/vol01 --size 100M --image-feature layering
    
    # List available volumes
    rbd ls
    # vol01
    
    # Map the volume to the system, which creates /dev/rbd0
    #   All named mounts are found under /dev/rbd/rbd/*
    #   The resulting volume may be accessed at: /dev/rbd/rbd/vol01
    rbd map rbd/vol01
    
    # Create a filesystem on the new block device
    mkfs.ext4 /dev/rbd/rbd/vol01
    
    # Mount the volume to the system
    mkdir -p /mnt/tmp
    mount -t ext4 /dev/rbd/rbd/vol01 /mnt/tmp
    
    # Check that the device is actually mounted
    df -h
    
    # Create some files in the new volume
    pushd /mnt/tmp
    echo "Hello World" | tee welcome.txt
    cat welcome.txt
    dd if=/dev/urandom of=urandom.bin bs=1M count=5
    popd
    
    # Unmount the new volume
    umount /mnt/tmp
    
    # Unmap the RBD device
    rbd unmap rbd/vol01
    

From the second OSD node, mount the volume that the first node created, read data, and unmount the volume.

    # SSH to the second Ceph OSD node where we will simulate a client
    ssh root@ceph-osd-0002
    
    # Map the volume to the system, which creates /dev/rbd0
    #   All named mounts are found under /dev/rbd/rbd/*
    #   The resulting volume may be accessed at: /dev/rbd/rbd/vol01
    rbd map rbd/vol01
    
    # Mount the volume to the system
    mkdir -p /mnt/tmp
    mount -t ext4 /dev/rbd/rbd/vol01 /mnt/tmp
    
    # Check the contents of the volume
    ls -la /mnt/tmp
    
    # Unmount the new volume
    umount /mnt/tmp
    
    # Unmap the RBD device
    rbd unmap rbd/vol01
    
    

## Configure a Kubernetes Container to Mount a Ceph RBD Volume

The rest of this guide assumes that you have a working Kubernetes environment created by following this ([Setup Guide][1]).

Ensure that your Kubernetes environment is working.

    export KUBECONFIG=~/.kube/config.pkt
    kubectl config use-context pkt-east
    kubectl config current-context
    kubectl cluster-info
    kubectl get svc,rc,po
    

First, base64-encode the ceph client key.

    ssh ceph-mon-0001 cat /etc/ceph/ceph.client.admin.keyring | \
      grep key|awk '{print $3}' | base64
    # VhyV0E9PQoVhyV0E9PQoVhyV0E9PQoVhyV0E9PQoVhyV0E9PQoVhyV0=
    

Create a file called &#8220;ceph-test.yaml&#8221;, which will contain definitions of the Secret, Volume, VolumeClaim, and ReplicationController. We will mount the &#8220;rbd/vol01&#8221; test volume created in the prior step into any test Container (in our case, nginx).

    ---
    
    # Replace the following configuration items
    # - {REPLACE_WITH_BASE64_ENCODED_CEPH_CLIENT_KEY}
    # - {REPLACE_WITH_PUBLIC_IP_OF_CEPH_MON}
    
    ---
    
    apiVersion: v1
    kind: Secret
    metadata:
      name: ceph-secret
    data:
      key: {REPLACE_WITH_BASE64_ENCODED_CEPH_CLIENT_KEY}
    
    ---
    
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: ceph-vol01
    spec:
      capacity:
        storage: 100Gi
      accessModes:
      - ReadWriteOnce
      rbd:
        monitors:
          - "{REPLACE_WITH_PUBLIC_IP_OF_CEPH_MON}:6789"
         #- LIST OTHER CEPH_MON'S HERE FOR REDUNDANCY
        pool: rbd
        image: vol01
        user: admin
        keyring: "/etc/ceph/ceph.client.admin.keyring"
        secretRef:
          name: ceph-secret
        fsType: ext4
        readOnly: false
    
    ---
    
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: ceph-vol01
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
    
    ---  
    
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: ceph-test
      namespace: default
    spec:
      replicas: 1
      selector:
        instance: ceph-test
      template:
        metadata:
          labels:
            instance: ceph-test
        spec:
          containers:
          - name: nginx
            image: nginx
            volumeMounts:
            - mountPath: /data
              name: ceph-vol01
          volumes:
            - name: ceph-vol01
              persistentVolumeClaim:
                claimName: ceph-vol01
    

Create the resources defined within &#8220;ceph-test.yaml&#8221;

    cat ceph-test.yaml | kubectl create -f -
    

Check that the nginx Pod/Container is running, and locate its name to use in the next command

    kubectl get po
    # NAME                           READY     STATUS    RESTARTS   AGE
    # ceph-test-61cks                1/1       Running   0          1s
    

Verify that the earlier-created files are readable within the &#8220;rbd/vol01&#8221; mount inside of the POD/Container

    kubectl exec -it ceph-test-61cks -- find /data
    # /data
    # /data/lost+found
    # /data/welcome.txt
    # /data/urandom.bin
    
    

We now have a functioning Ceph cluster that is mountable by a Kubernetes environment.

If the Pod is destroyed and migrated to a different Kubernetes node, the volume mount will also follow.

 [1]: https://www.davidwang.com/2016/05/21/kubernetes-bare-metal-install-on-packet-net/
 [2]: http://ceph.com/ceph-storage/block-storage/
 [3]: https://packet.net
 [4]: https://github.com/ceph/ceph-ansible
 [5]: https://github.com/dcwangmit01/vagrant-devenv