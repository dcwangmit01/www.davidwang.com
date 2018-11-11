---
title: Kubernetes Bare-Metal Install (on Packet.net)
date: 2016-05-21T00:11:37+00:00
tags:
  - bare-metal
  - kubernetes
  - packet.net
  - technical
  - ansible
  - bare-metal
  - ceph
  - container
  - docker
  - kubernetes
  - packet.net
draft: false
---
Installing Kubernetes on bare-metal machines is dead simple, and a million times easier than installing OpenStack. The bulk of the instructions below involve setting up the bare-metal machines on [packet.net][1].

## What You'll Get with These Instructions

One may use these instructions to create a _basic_ Kubernetes cluster. In order to create a cluster environment equivalent to a hosted solution (GKE) or turn-key solutions (Kubernetes on AWS or GCE), you'll need persistent volume and load-balancer support. A future post will cover how to setup persistent volume storage as a Ceph RBD cluster, and how to work around the need for external load-balancer integration by deploying a Kubernetes Ingress DaemonSet with DNS.

In the following guide, we'll build a Kubernetes cluster with 1 master and 2 nodes.

## Bare-Metal or Cloud VMs, All the Same

These instructions should also work cloud VMs, so long as the following criterial are met:

  * All machines are network reachable, without restriction 
      * i.e. open iptables, open security groups
  * Root access through password-less SSH is enabled 
      * i.e. configured /root/.ssh/authorized_keys

Because not many of us have multiple bare-metal machines laying around, we'll rent them from [packet.net][1].

## Step Summary

  * Setup a Development Environment
  * Provision Bare-Metal Machines (on packet.net)
  * Configure and Run the Kubernetes [contrib/ansible][2] scripts
  * Test the Cluster by Creating a POD
  * Configure a Remote Kubernetes Client to Talk to the Cluster

## Setup a Development Environment

The easiest method is to instantiate a [vagrant machine][3] with all of the necessary tools. Feel free to do this manually. The vagrant environment includes the following:

  * Docker
  * Kubectl
  * Ansible
  * Hashicorp Terraform

Fetch the vagrant machine

    # Tested at dcwangmit01/vagrant-devenv - branch master, commit a6c65478
    git clone git@github.com:dcwangmit01/vagrant-devenv
    cd vagrant-devenv
    

If you are on OSX, ensure and install the host dependencies.

    # Installs brew, virtualbox, etc.
    make deps
    

Create your vagant environment

    # Start ssh-agent and load your key
    eval `ssh-agent`; ssh-add
    
    # Create the vagrant machine
    vagrant up
    
    # Connect to the box
    vagrant ssh
    
Note: From now on, do everything from within this vagrant machine.

## Provision Bare-Metal Machines (on packet.net)

The [contrib/ansible][2] scripts are targeted to offical Redhat distributions including RHEL and Fedora. However, packet.net does not currently support these operating systems. Thus, we deploy Centos7 servers, which is the next-closest thing and happens to work.

If you enjoy clicking around Web UI's, follow the manual instructions below. Otherwise, the only automated provisioning method supported by [packet.net][1] is to use Hashicorp Terraform. A CLI client does not yet exist.

### Manual WebUI Instructions

  * Login to the web UI at [packet.net][1]
  * Create a project
  * Set an SSH key (which will be provisioned on new servers)
  * Create 3 servers with Centos7
  * Note the names and IPs of the newly created machines

### Automated Instructions (using Terraform)

Instantiating servers via Hashicorp Terraform must happen in two steps if a &#8220;packet&#95;project&#8221; has not yet been created.

This is because the &#8220;packet&#95;device&#8221; (aka. bare metal machine) definitions require a &#8220;project&#95;id&#8221; upon execution. However, one does not know the &#8220;project&#95;id&#8221; until after the &#8220;packet&#95;project&#8221; has been created. One might resolve this issue in the future by having the packet&#95;device require a project&#95;name instead of a project&#95;id. Then we could &#8220;terraform apply&#8221; this file in one go.

See the Appendix for curl commands that will help you discover project&#95;ids from the API.

#### Step 1: Create the Packet.net Project

First, create your auth token via the [packet.net][1] UI and note it down.

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
    

Append to cluster.tf, and tweak the {variables} according to your needs.

    # Create a device
    resource "packet_device" "kube-master-0001" {
            hostname = "kube-master-0001"
            plan = "baremetal_0"
            facility = "ewr1"
            operating_system = "centos_7"
            billing_cycle = "hourly"
            project_id = "{REPLACE_WITH_PACKET_PROJECT_ID}"
    }
    
    resource "packet_device" "kube-node-0001" {
            hostname = "kube-node-0001"
            plan = "baremetal_0"
            facility = "ewr1"
            operating_system = "centos_7"
            billing_cycle = "hourly"
            project_id = "{REPLACE_WITH_PACKET_PROJECT_ID}"
    }
    
    resource "packet_device" "kube-node-0002" {
            hostname = "kube-node-0002"
            plan = "baremetal_0"
            facility = "ewr1"
            operating_system = "centos_7"
            billing_cycle = "hourly"
            project_id = "{REPLACE_WITH_PACKET_PROJECT_ID}"
    }
    

Run Terraform

    terraform plan -out cluster
    terraform apply
    

Note the names and IPs of the newly created machines

    cat terraform.tfstate|grep -E '(hostname|network.0.address)'
    # Returns something like this
    # "hostname": "kube-master-0001",
    # "network.0.address": "147.75.199.63",
    # "hostname": "kube-node-0001",
    # "network.0.address": "147.75.199.131",
    # "hostname": "kube-node-0002",
    # "network.0.address": "147.75.196.83",
    

Create an ~/.ssh/config file with the hostname to IP mappings. This allows you to refer to the machines via hostnames from your Dev box, as well as Ansible inventory definitions.

    Host kube-master-0001
      User root
      Hostname {REPLACE_WITH_IP_OF_kube-master-0001}
    Host kube-node-0001
      User root
      Hostname {REPLACE_WITH_IP_OF_kube-node-0001}
    Host kube-node-0002
      User root
      Hostname {REPLACE_WITH_IP_OF_kube-node-0002}
    

## Configure and Run the Kubernetes contrib/ansible Scripts

Fetch the kubernetes/contrib code

    # Tested at kubernetes/contrib commit 3634e300
    git clone git@github.com:kubernetes/contrib.git
    cd contrib/ansible
    

Create an Ansible inventory file (more info in the contrib/ansible/README.md file)

    # Create the inventory file from the example
    cp inventory.example.single_master inventory
    
    # Edit the inventory file and fill in the hostnames or IP addresses
    emacs inventory
    

Example &#8220;inventory&#8221; file, which will work without modification if ~/.ssh/config is configured with host IP mappings.

    # configure the master IPs
    [masters]
    kube-master-0001
    
    # the following section should be the same as the [masters] above
    [etcd]
    kube-master-0001
    
    # configure the node IPs
    [nodes]
    kube-node-0001
    kube-node-0002
    

Test the ansible connection, and accept host-keys into your knownhosts

    ansible all -u root -i inventory -m ping
    

Configure group&#95;vars/all.yml to set the ansible&#95;ssh&#95;user to &#8220;root&#8221;

    # The following will uncomment "#ansible_ssh_user: root"
    sed -i 's@^#ansible_ssh_user@ansible_ssh_user@' group_vars/all.yml
    

Run the ansible setup script to install Kubernetes on all machines

    ./setup.sh
    

&#42; Run this more than once until zero errors. It's idempotent!

## Test the Cluster by Creating a POD

The kubernetes master is auto-configured to talk with the cluster. Run some test commands

    ssh root@kube-master-0001
    kubectl cluster-info
    kubectl get po,rc,svc
    

Create a POD, and test that it is running

    # Start a POD
    kubectl run echoheaders --image=gcr.io/google_containers/echoserver:1.3 --port=8080
    # Find the pod ID
    kubectl get po
    # Start a new bash process within the first container within the pod
    kubeclt exec -it {THE_POD_ID} bash
    # Make a call
    curl localhost:8080
    # Exit with {ctrl-d}
    

## Configure a Remote Kubernetes Client to Talk to the Cluster

Now that the kubernetes cluster has been setup, you may use &#8220;kubectl&#8221; directly after ssh'ing to the master. However, it would be nice to use &#8220;kubectl&#8221; from the vagrant dev environment.

    # Copy the kube/config from the master to your dev machine
    scp root@kube-master-0001:/etc/kubernetes/kubectl.kubeconfig ~/.kube/config.pkt
    
    # Rename the context-name to be "pkt-east"
    sed -i 's@kubectl-to-cluster.local@pkt-east@' ~/.kube/config.pkt
    
    # Set the search/merge path for KUBECONFIG
    export KUBECONFIG=~/.kube/config.pkt:~/.kube/config
    
    # Verify the context by looking at ~/.kube/config.pkt
    less ~/.kube/config.pkt
    
    # Set the context
    kubectl config use-context pkt-east
    
    # Verify the context
    kubectl config current-context
    
    # Run a some commands to verify that it all works
    kubectl get svc,rc,po
    

  * Now you have a working kubernetes cluster on Packet.net 
      * Excluding persistent storage and load-balancing

## Appendix

### Curl Commands

    # Locate your API Key from the Web Portal https://app.packet.net/
    export PACKET_AUTH={REPLACE_WITH_API_KEY}
    
    # Get a list of projects
    curl -s -H  "X-Auth-Token: $PACKET_AUTH" https://api.packet.net/projects |python -m json.tool |less
    
    # Get a list of plans (aka flavors)
    curl -s -H  "X-Auth-Token: $PACKET_AUTH" https://api.packet.net/plans |python -m json.tool|less
    
    # Get a list of facilities
    curl -s -H  "X-Auth-Token: $PACKET_AUTH" https://api.packet.net/facilities |python -m json.tool|less
    
    # Get a list of operating systems
    curl -s -H  "X-Auth-Token: $PACKET_AUTH" https://api.packet.net/operating-systems |python -m json.tool|less
    

### Example Settings

    # Plan IDs
    baremetal_0 baremetal_1
    
    # Facilities
    sjc1 ewr1
    
    # Operating Systems
    centos_7

 [1]: https://packet.net
 [2]: https://github.com/kubernetes/contrib
 [3]: https://github.com/dcwangmit01/vagrant-devenv
