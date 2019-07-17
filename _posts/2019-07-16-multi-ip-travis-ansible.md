---
layout: post
title: Testing Multi-Machine Ansible Roles with Travis-CI and Docker compose
excerpt_separator:  <!--more-->
description: Aiming to increase my Ansible skills, I spent some time improving my Ansible role for deploying the Slurm scheduler. This is how I managed to test the server-client Ansible role in Travis-CI.org using Docker compose.
categories:
    - technology
tags:
    - Ansible
    - Docker
    - debops
    - Docker-compose
    - Testing
    - automation
    - CI/CD
    - continuous integration
    - continuous delivery
    
---
A little while ago I spent some time writing various Ansible roles and playbooks for the infrastructure at my place of work. My Ansible skills are intermediate and by no means refined. As a result of this, a lot of the roles were not developed to best practice specifications.

I took some time to try to improve my roles and properly test them before using them by taking advantages of the free continuous testing service that [Travis-CI](https://travis-ci.org) offers. I quickly ran into an issue while working on my role for deploying the [Slurm](https://slurm.schedmd.com/) scheduler, that being how do I test the deployment if I only have one VM to work on?

The answer? Docker and Docker-compose!

<!--more-->

## Slurm Scheduler

Slurm is a scheduling software. In brief, it manages a fleet of machines (which may have different hardware configurations) and decides which machines should execute jobs from which users in a multi-user cluster environment. This decision is based on various factors such as the sharing policy and the resources and/or time that the user is requesting from the cluster.

This is useful for environments where you have many users that need to run software that requires high amounts of system resources and takes long to run to completion, such as academic environments.

Slurm has 3 main services:

| Service | Description                                                                                    |
|---------|------------------------------------------------------------------------------------------------|
|slurmctld| This manages the fleet of hardware and the client service connects to this.                    |
|slurmdbd | This service connects to an existing DB and stores accounting information (think job histroy). |
|slurmd   | This is the client service that runs on each of the machines that will execute jobs.           |

For a typical environment, the `slurmctld` and `slurmd` services won't be running on the same machine. This is to ensure that the `slurmctld` does not get starved of resources.

## Ansible Role

The Ansible role that I wrote for deploying Slurm at my workplace ([found here](https://github.com/Banshee1221/ansible-role-slurm)) is capable of deploying all of the Slurm services on an Ubuntu based (for now) cluster. For the gist of it, you specify an inventory file that determines which machines will have the `slurmctld`, `slurmdbd` and `slurmd` services deployed on them. This is specified in the following manner:

```text
[slurm]
headnode.cluster.com slurm_builder=True headnode=True
computenode1.cluster.com
computenode2.cluster.com
```
Where `slurm_builder` defines which machine needs to do the building of the actual slurm software and a true value for `headnode` determines which machine needs to have the `slurmctld` and `slurmdbd` software deployed on it.

It should be clear to see that to accurately test this role I would need to run it with at lest two machines, having one speak to the other.

## Testing the Role
### Setup

In order to test the role with Travis, I made sure to link my Github profile with Travis-CI. After that, the directory structure of the project needs to be modified.

Here is what the original project structure looks like __without__ Travis-CI:

```text
ansible-role-slurm/
├── defaults
│   └── <ansible_default_variables>
├── files
│   └── <template_files_for_slurm>
├── handlers
│   └── <ansible_handlers>
├── tasks
│    └── <ansible_tasks>
└── README.md
       
```

Adding the Travis-CI stuff brings us to this:

```text
ansible-role-slurm/
├── defaults
│   └── <ansible_default_variables>
├── files
│   └── <template_files_for_slurm>
├── handlers
│   └── <ansible_handlers>
├── tasks
│   └── <ansible_tasks>
├── travis
│   ├── ansible
│   │   ├── group_vars
│   │   │   └── all.yml
│   │   ├── inventory
│   │   └── site.yml
│   ├── headnode
│   │   └── Dockerfile
│   ├── computenode
│   │   └── Dockerfile
│   └── docker-compose.yml
├── .travis.yml
└── README.md
```

Travis-CI gives you a single virtual machine to run your tests on. It also gives you the ability to write a pretty simple YAML document to specify the kind of environment that should be deployed onto your virtual machine. We want to use the Docker environmnet for this as it will provide us with both the Docker runtime as well as the docker-compose script. Since I created a `travis/` directory in the main repo, I want to use that to place all our testing related files. So the `.travis.yml` file will look something like this:

```ruby
services: docker

before_install:
  - docker -v  # Make sure that you're running with latest docker
install:
  - cd travis  # Go into the travis/ directory
  - docker-compose up -d  # Run the docker-compose.yml in the travis/ directory
  - sleep 10   # Wait to ensure everything is up as expected
```
This will allow you to place anything inside of the travis/ directory and execute it on the VM that Travis-CI provides for you. 

### Docker and Docker Compose

Since I know that Travis-CI will now operate out of the `travis/` directory, I created a `docker-compose.yml` file in there and populated it with a bunch of stuff that allows the simulation two independent networked machines. To do this, a Docker network needs to be created in the bridged mode so that static IP addresses can be assigned. With this done, each of the machines or "services" you create in the same `docker-compose.yml` will need that same network attached to it. 

Here's the full `docker-compose.yml` with some added comments for your understanding.
```ruby
version: "3.4"

services:

  headnode:  # Defines the headnode container to be created
    build:   # Tells docker-compose script to build a Docker image based on a Dockerfile
      context: './headnode/.' # Specifies where the Dockerfile lives
    privileged: true # This is needed for systemd to work
    networks: # Specifies the network that the container should use
      vpcbr:  # We are using our pre-defined network as shown below 
        ipv4_address: 10.5.0.2  # Assign this container a specific IP address in our network
    hostname: "headnode.cluster"  # Assign this container with a specific hostname
    ports:  # Expose the following ports for Slurm/munge and Ansible needs
      - '80'
      - '443'
      - '60000-60100'
      - '6817-6819'
      - '22'
    cap_add:  # Also needed for systemd
      - SYS_ADMIN
    volumes:  # Directories from the host machine to bind inside the container
      - './ansible/inventory:/ansible/inventory'  # bind the test ansible inventory to /ansible/inventory inside the container
      - './ansible/site.yml:/ansible/site.yml'  # bind the test ansible site.yml to /ansible/site.yml inside the container
      - './ansible/group_vars/:/ansible/group_vars'  # bind the test ansible group_vars to /ansible/group_vars/ inside the container
      - '../:/ansible/roles/ansible-role-slurm'  # bind the role directory (the project dir) to /ansible/roles/ansible-role-slurm inside the container
      - '/sys/fs/cgroup:/sys/fs/cgroup:ro'  # Systemd related
    extra_hosts: # Define static DNS mappings for Ansible purposes
      - "headnode.cluster:10.5.0.2" 
      - "computenode.cluster:10.5.0.3"
      - "headnode:10.5.0.2"
      - "computenode:10.5.0.3"

  # This is defined in the same way as above and acts as the second machine. 
  # It has a different IP address to the one above and also does not require
  # the Ansible files.
  computenode:
    build:
      context: './computenode/.'
    privileged: true
    networks:
      vpcbr:
        ipv4_address: 10.5.0.3
    hostname: "computenode.cluster"
    ports:
      - '80'
      - '443'
      - '60000-60100'
      - '6817-6819'
      - '22'
    cap_add:
      - SYS_ADMIN
    volumes:
      - '/sys/fs/cgroup:/sys/fs/cgroup:ro'
    extra_hosts:
      - "headnode.cluster:10.5.0.2"
      - "computenode.cluster:10.5.0.3"
      - "headnode:10.5.0.2"
      - "computenode:10.5.0.3"

# This defines the network that will be used for the containers to be
# connected together with.
networks:
  vpcbr:
    driver: bridge
    ipam:
     config:
       - subnet: 10.5.0.0/24
```

As shown above, I need to build the Docker images that I needed for actually deploying the Ansible role onto. I created a `headnode/` and `computenode/` directory in the `travis/` directory and placed a `Dockerfile` file in each of them. Both of the images are based off of the `systemd-ubuntu:18.04` by `jrei` on DockerHub since the Ansible script relies on working with systemd. Along with this, some customisations were needed. I generated a random ssh keypair and stored the private key in the `headnode` Dockerfile and the public key in the `computenode`'s `authorized_hosts` file in order to allow keyless sshing for the Ansible to do its thing. These two Docker images can now, when deployed, act as a 2-node cluster on which we can test the Ansible deployment.

With all that being done, when deploying to GitHub, Travis should automatically trigger a job and execute whatever is in the `install` section of the `.travis.yml` file. It will bring up the two containers and they should be able to network to one another. The next step is to actually deploy the Ansible and to do some testing. To do this, I added the following to the `.travis.yml` file:

```ruby
services: docker

before_install:
  - docker -v
install:
  - cd travis
  - docker-compose up -d
  - sleep 10
script:
  - docker-compose exec headnode bash -c "cd /ansible && ansible-playbook -i inventory site.yml --syntax-check"
  - docker-compose exec headnode bash -c "cd /ansible && ansible-playbook -i inventory site.yml"
  - sleep 10
  - docker-compose exec headnode bash -c "cat /tools/admin/slurm/etc/slurm.conf"
  - docker-compose exec headnode bash -c "ls /tools/admin/slurm"
  - docker-compose exec headnode bash -c "ps aux | grep slurm"
  - docker-compose exec computenode bash -c "ps aux | grep slurm"
  - docker-compose exec headnode bash -c "journalctl --unit=slurmctld -n 100 --no-pager --full"
  - docker-compose exec computenode bash -c "journalctl --unit=slurmd -n 100 --no-pager --full"
  - docker-compose exec headnode bash -c "/tools/admin/slurm/bin/sinfo"
  - docker-compose exec computenode bash -c "/tools/admin/slurm/bin/sinfo"
```

The steps under the `script` section will run syntax checks and do the deployment of the Ansible. Once that is complete, the contents of some files and directories will be outputted to stdout so that I may verify whether they are correct. I watch for the existence of some processes that I expect with `ps` and `grep`, check some logs for the services I've deployed and lastly I run some Slurm commands to confirm that all is working well.

The result works quite well!
![Travis-CI Build Log](/assets/images/tarvis_ci_ansible_result.png){:class="img-responsive"}


## Future Steps

My role refactoring is still a long way away and I'm sure that this is not the best way to test Ansible roles with Travis. I want to experiment with other testing systems such as [Molecule](https://molecule.readthedocs.io/en/stable/). I also want to expand my roles to support different operating systems and make them more fault tolerant. I might also do some Travis matrix stuff in order to test all of the different operating system environments.

Maybe I'll write another post about that when I get to it 😁

