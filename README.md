# Docker Swarm on OVH Public Cloud with persistent volume

Hi! Here is the first implementation (for testing purpose) of a Docker Swarm infrastructure with persistent volumes on OVH Public Cloud. Due to Mesos/Marathon stack end of life.

# Architecture model

For this test we will use:

 - 1 private network (Docker-Swarm-Prod)
	 - This network is for the Swarm communication
	 - All instances are linked to this network
	 - Subnet: 192.168.0.0/24
 - 2 instances on CentOS 7
	 - 1 manager
		 - Network:
			 - eth0: public IP address
			 - eth1: 192.168.0.10 
	 - 1 worker
		 - Network:
			 - eth0: public IP address
			 - eth1: 192.168.0.20

![Architecture model](https://lh3.googleusercontent.com/9Jl_qfUBf7yBZnFQS5nGnkiWivcGsOTDR4gGraHKSotHV7XqrAEgcXzgPaFKyczBFXenLIZanHJlsJw1D3xKBwPHW8sLVJfQO0b_Y-BNNkobe0rClMNZeBDJ5n2VT3O2KaxSgt185JswSyZLqEqymxWpCsJtgmfUv5WtCSYtlRorCfRMfgxAtY-ZnWPlnq6yZOf1YG4PXq7JVyd3LrbDYr6XoAw-5NkkGol9zj1hR1MZF40NNoKX1UHH2E7cc_E_ILTKW4uRYRZa1SjV7oi06gClFg2Md8KNefRk7eV0yhb5TBXJdQo-Ssynu27kgdaiIQhAy5oMJ_yUZoyH_ioj48bHHTGOUslwsFer_E1nxR7gBDXHtHgl4uqXZC9V1uVenYVwF6DJjNoKImBDeSji2UVxrOXw61exoUzyd6Tsm6IBC71Xl6x8uxcd_3MEeo_eAjHJcJDMDum1kAUHZ-coQ_LeN02G-QK7ZMt68pOb3zF-ZZmqTPXdaFHSFX7HUsugfzSS570UGnJe3Z-4n7IeL9LTtk9nWErC3h6eX8ly_CNM_cPrWqscba1qYvixdUa8cP8v_np-cBZEefubKWxuFGT5HW74ahOg25TJ141VYgbMQNGHLf99aAraRwuHPC_pHu9hm2M-guRX_eX7YM0E_mnczO_Jj0ex=w1366-h752-no)

## Installation

### On each node as root

#### Update the server

    yum -y update

#### Network configuration

Verify and configure (if needed) the network

 - /etc/sysconfig/network-scripts/ifcfg-eth0
- /etc/sysconfig/network-scripts/ifcfg-eth1

#### Cleanup some firewall rules

    echo "ZONE=public" >> /etc/sysconfig/network-scripts/ifcfg-eth0
    echo "ZONE=internal" >> /etc/sysconfig/network-scripts/ifcfg-eth1
    
    systemctl restart network
    
    firewall-cmd --get-active-zones
    
    firewall-cmd --zone=public --list-all
    
    firewall-cmd --zone=internal --permanent --remove-service=mdns
    firewall-cmd --zone=internal --permanent --remove-service=samba-client
    firewall-cmd --zone=internal --permanent --remove-service=dhcpv6-client
    firewall-cmd --reload
    
    firewall-cmd --zone=internal --list-all

(Optional) you can remove ssh access from the public IP address if you can access it from the private network

    firewall-cmd --zone=public --permanent --remove-service=ssh
    firewall-cmd --reload

#### Configure the firewall for Docker Swarm

On the public zone we allow http/https by default but you can add your services ports here

    firewall-cmd --zone=public --permanent --add-service=http
    firewall-cmd --zone=public --permanent --add-service=https
    
    firewall-cmd --reload
    
    firewall-cmd --zone=public --list-all

On the private zone we allow ports for Swarm's management

    firewall-cmd --zone=internal --permanent --add-port=2376/tcp
    firewall-cmd --zone=internal --permanent --add-port=2377/tcp
    firewall-cmd --zone=internal --permanent --add-port=7946/tcp
    firewall-cmd --zone=internal --permanent --add-port=7946/udp
    firewall-cmd --zone=internal --permanent --add-port=4789/udp
    
    firewall-cmd --reload
    
    firewall-cmd --zone=internal --list-all
    
## Install Docker CE

### On each node as root

    yum install -y yum-utils

    yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

    yum install -y docker-ce

    systemctl enable docker

    systemctl start docker

### On the Manager node as root

#### Install Swarm mode

    docker swarm init --advertise-addr 192.168.0.10

(For reminder) You can use these command on the manager to retrieve the tokens for adding another manager or worker node

    docker swarm join-token manager
    docker swarm join-token worker

### On the Worker node as root

#### Join the Swarm pool

    docker swarm join --token <token> 192.168.0.10:2377

### On the manager node as root

See the status of your pool

    docker node ls

![show the swarm pool](https://lh3.googleusercontent.com/eBjzLvpPGChYLmAjTQQrZ4qKTqy2_xlsapSP-9kpwXXu5nd6BfXU8bxitmtvdrQQrQt7eiaG0OxQ3911KodHWOMToKvoiXOMBtGlyJsVCHSdEu174rCZRWMFUDbHwcKyIvW321s2bg=w1132-h137-no)

## Install Rex-Ray for persistent volume with OpenStack Cinder

### On each node as root

    curl -sSL https://rexray.io/install | sh

Here you need to change some information:
- Openstack Username
- Openstack Password
- Openstack Tenant
- Openstack Tenant Name
- Openstack Region

You can find these information inside the OpenRC file. You can follow this guide to download the OpenRC file: https://docs.ovh.com/fr/public-cloud/charger-les-variables-denvironnement-openstack/


```
echo "libstorage:
  service: cinder
  integration:
    volume:
      operations:
        mount:
          preempt: true
cinder:
  userName: <Openstack Username>
  password: <Openstack Password>
  authURL: https://auth.cloud.ovh.net/v2.0/
  tenantID: <Openstack Tenant>
  tenantName: <Openstack Tenant Name>
  regionName: <Openstack Region e.i: GRA3>
  availabilityZoneName: nova" > /etc/rexray/config.yml
```

    systemctl enable rexray
    
    systemctl start rexray

## Test your infra

### Create the persistent volume

    rexray volume create docker-swarm-prod-pg_data --size=5

You can see the volume on each swarm node

    rexray volume ls

![volume list](https://lh3.googleusercontent.com/4PfbG_RwFlC-VYNtCJvsYsWJJcZsYliq4LckozH_Vnh8rl7Zx3lAx4psv1dLa5zwraM5muNfpgk5N9jr15lkzqcUyAHc8sTXlzeoLDcGPkRz26usH_Rw3gtEzzVgppI9JtB3jN2KpDZ7YrUiZ4RbYgGXo4pSYtmQ2bdlAtEcRhtCM1y_DqENyYdR-BvQrs-naVlXLFvC08wLIzKNVDfQR5VMNSncJu7lqXC8sH6yul84f0koX5JTIPjXXD-YmSFx38_MhkXGiUCgH_QwRWuD9PiXBcwT9yKpo_4gOryO9ciNzUhjp7hvCOuw5sYpTEhKtmyAVG7KzlKTEOBVvzfoQvG6ckHEZtfPqlj5DuSHA5VudyV61DzGo4KF1WQznb7ooCrnrxacMXsbeOKCyfm9mkMh_OsA0OQg_ikUBGLsZBDbilSw-ddUT6Unjt8sndVFm4XtBrnnclytG_DKneFt5qMEp9gD_4rgKv_VMLh1-CYviHmZI3gFyGO5bsQ_krFu8M_epdsNJsCQKpgVyWLAMqDc9PjkHEroiMR81W52XB3YKC7HHTh3GKHOuFrjsiAb8BB1lsNFYxzWeIlg7BjJelQuq8DbW61ebMlwIv1QgJiZvmo1sd7KYS831e16l2Z8e-hbrkCV7zCRdr50sR02yB3llxP2Z_N9=w748-h223-no)

Same with the docker volume list command

    docker volume ls

![docker volume ls](https://lh3.googleusercontent.com/1Xi1zO15Ja7KRY1xY0MwX56e42rIAzM_mYvsHOtpBxWntXcIlyygmGRYn03V7NqgmYXpqsd5mgOGItFo51cKbujG54iDnUROxXHvibY5hh4wXqnsk1rWgqEwh6r9-o-xKDbdaXB8BA=w481-h224-no)

### Create a test container with the persistent volume attached

    docker service create --replicas 1 --name pg -e POSTGRES_PASSWORD=mysecretpassword --mount type=volume,source=docker-swarm-prod-pg_data,target=/var/lib/postgresql/data,volume-driver=rexray --constraint 'node.role==worker' postgres

This container will start on the worker node (due to the constraint)

![start instance](https://lh3.googleusercontent.com/B2k-2GGlSSxPC78AiT1e4biAFPSkWhg29p4Pr2fC8z8kP3KYVjf4r1MzUXSf56hDv-_QlWhisrALRxGzgkQP9I6krfqlybNcZ1QEEXHotSOuon-Dp_89Tmi307mNRPjnOA3QXhkumw=w1679-h279-no)

On the worker, you can see the container with this command

    docker ps

Also in the OVH interface (the 5go attached hard drive)
![persistent volume attached to the worker](https://lh3.googleusercontent.com/ptqWVtAcufLmMSwFOfA5fGTSwpWjLka8R_LeyOLUcMftvgmh5IEQQHehT824IrhlTvw9-L_ztVoXZQhqdQbkDixGQOo9lxa6V36gLinlzSqYqNhCCt4enf0_06cMjh9-voc6pzKCDA=w395-h643-no)

On the worker node, connect to the container and create a new database

![create database](https://lh3.googleusercontent.com/NkbjLDTEf5wcAC6kALN3R6fSBPowA33AijASDdxUWFK2jkB6WPqc0VRLA3Uhp-QtLNg7gIIROnEZ8HjDytqb86G4uXjbpBbiCL_H4Pjc8KEEm21Rkrk6L-SEaultrYREax7IuGKzew=w758-h524-no)

    docker exec -ti <name> sh
    
    su - postgres
    
    psql

    CREATE DATABASE testvolumepersistance;
    
    \l
    
    \q

    exit
    
    exit

### Simulate worker failed

On the manager, we remove the placement constraint of the container (because we only have 1 worker)

    docker service update --constraint-rm node.role==worker pg

On the worker, we stop docker

    systemctl stop docker

On the manager we can see the status of the pool

    docker node ls

![worker is down](https://lh3.googleusercontent.com/qjgBiGbbsxgizzWJ-VeYc533gueuEtbVFp66ApSsUzQdAVzFNQC2K6Ic_ZoyhpoC7AZ0OWx9SwY7eiRfxIdR12VIa2P3QaevaAb8p8e2xAL3NnWiGAQ0f4UMpCHTMMOd9zF68_oXZg=w1165-h149-no)

We can also see the status of the service

    docker service ps pg

![docker service status](https://lh3.googleusercontent.com/zIDdc_CfVPhjFQdnt7G6PSQByU1dIpE8E-zELyd5G13GKD-LDzU6rUVEt2hFs8RBv3nYQtEZUTLUTI7vJ8KFC8u-Xyuztnw91TBxVfJh0JgOsRvCUsNBtB7GAHbp7Bv9Rda0VhJxWQ=w1486-h97-no)

Our service has been restart to the manager node.

We can see in the OVH Interface, our volume has been attached to the manager

![volume attached to manager](https://lh3.googleusercontent.com/qnA3e_IQge0jC_1tQ2j9yNUyxPEtP4ueemZJkEhY5O-yDXe5fMlFb6NCyuDLHJF8IEEOhj5-aHg0V6b3Pzx-dzSjkwCvl4SEJH6gIFhseuosBf6LPBcTeJ-EMITQlyLp4QV19FE5GA=w395-h638-no)

What about the data... Connect to the container on the manager node

![persistent data works](https://lh3.googleusercontent.com/G_2J_NUHx9cz4VA7abApp3ne_mZDBdqkellVwLRKQ_oWDqy9KFweLQwFidOvf1VDYqL7ozffUMUYOMZ00jPxVCLaZPUQMm-T97bR2AJON9BRx1cGh8CSAqrt2XoJD-zHkddpnxTlxw=w1409-h556-no)

    docker ps
    
    docker exec -ti <name> sh
    
    su - postgres
    
    psql
    
    \l
    
    \q
    
    exit
    
    exit

As we can see, our test database is here!
