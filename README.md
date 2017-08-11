# Midokura MidoNet v5.2 Intergration with HOS 3.x Liberty

## Introduction

MidoNet is an open-source network virtualization platform for Infrastructure-as-a-Service (IaaS) clouds. MidoNet decouples your IaaS cloud from your network hardware, creating an intelligent software abstraction layer between your end hosts and your physical network. This network abstraction layer allows the cloud operator to move what has traditionally been hardware-based network appliances into a software-based multi-tenant virtual domain. 

Some of the benefits that come from using MidoNet in your IaaS cloud are:
- the ability to scale IaaS networking into the thousands of compute hosts.
- the ability to offer L2 isolation which is not bounded by the VLAN limitation (4096 unique VLANs)
- making your entire IaaS networking layer completely distributed and fault-tolerant

## Versions Tested
Midokura Midonet : v5.2 \
HOS versions : 3.0 & 3.0.1 

## What is supported after deployment?
### Neutron
Tested and deploying using playbooks. Ansible Tempest tests pass without errors and manually testing also works. 

| Command | Description |
| --- | --- |
| `git status` | List all *new or modified* files |
| `git diff` | Show file differences that **haven't been** staged |
| Action| Status|
| ------------- |:-------------:|
| Create Networks| :white_check_mark: |
| Create Subnets | :white_check_mark: |
| Create Routers | :white_check_mark: |
| Create Floating IPs | :white_check_mark: |
| Create Subnets | :white_check_mark: |
| Add Router Interface | :white_check_mark: |
| Set Gateway Router | :white_check_mark: |

### FWaaS 
Tested and deploying using the playbooks below. The logging firewall creating doesn't work as it is not a subcommand in our neutron version. 
| Action| Status|
| ------------- |:-------------:|
| Create Rules| :white_check_mark: |
| Create Policy | :white_check_mark: |
| Create Firewall | :white_check_mark: |
| Update Firewall | :white_check_mark: |
| Delete Routers From Firewall | :white_check_mark: |
| Logging Creation (doesn't affect FWaas functionality) | :red_circle: |
| Firewall Logging Creation (doesn't affect FWaas functionality) | :red_circle: |

### VPNaaS (No Liberty Support) 
Tested and not deploying using the playbooks below. It seems the last part for the VPN fails to create and Neutron doesn't report any errors. Just returns a API URL error. 
| Action| Status|
| ------------- |:-------------:|
| Create VPN IKE Policy| :white_check_mark: |
| Create VPN IPsec Policy | :white_check_mark: |
| Create VPN Service | :white_check_mark: |
| Create IPsec Site Connection | :red_circle: |

### LBaaS 
Tested and deploying using the playbooks below. \
| Action| Status|
| ------------- |:-------------:|
| Create Pools| :white_check_mark: |
| Associate Floating IP | :white_check_mark: |
| Associate Monitor | :white_check_mark: |
| Create VIP | :white_check_mark: |
| Add Members | :white_check_mark: |
| Add Monitor | :white_check_mark: |


## Architecture

### HOS and MidoNet Architecture
![Alt text](docs/MidoNet_Arch.png?raw=true "HOS and MidoNet Arch")

- MidoNet Agent (Midolman) has to be installed on all nodes where traffic enters or leaves the virtual topology. In this guide these are **gateway1, gateway2, gateway3, compute1 and compute2 hosts.**
- Midonet Cluster can be installed on a separate hosts, but this guide assumes it to be installed on the **controllers hosts.**
- Midonet Command Line Interface (CLI) can be installed on any host that has connectivity to the MidoNet Cluster. This guide assumes it to be installed on all **controller hosts**.
- Midonet Neutron Plugin replaces the ML2 Plugin and has to be installed on the **controllers.**

### MidoNet Network Topology 
![Alt text](docs/Midonet_Network_Topology.png?raw=true "HOS and MidoNet Arch")

## MidoNet Integration

### Pre HOS Installation Configuration
Before deplying HOS, we need to make a few changes to the disks in the **controllers**. Comment out the Zookeeper section in the **disk_controller_1TB.yml** and **disk_controller_600GB.yml** . Create a new Volume group for Zookeeper and Cassandra using a none used drive, in this case we selected **sdd**. MidoNet recommends having these two partition separately and with as much space as we can give.

```
# Zookeeper is used to provide cluster co-ordination in the monitoring
# system. Although not a high user of disc space we have seen issues
# with zookeeper snapshots filling up filesystems so we keep it in its
# own space for stability.
#- name: zookeeper
# size: 1%
# mount: /var/lib/zookeeper
# fstype: ext4
 
        consumer:
           name: os
```
```
    consumer:
       name: os
 
 
# MidoNet Zookeeper and Cassandra partitions.
volume-groups:
  - name: nsdb-vg
    physical-volumes:
    - /dev/sdd
    logical-volumes:
      - name: zookeeper
        size: 45%
        mount: /var/lib/zookeeper
        fstype: ext4
      - name: cassandra
        size: 45%
        mount: /var/run/cassandra
        fstype: ext4
    consumer:
       name: nsdb
```
*This will cause Warnings while you run **config-processor-run.yml** but you can ignore those error messages.* 

Midonet requires 2 NIC interfaces. 1 for nodes intercommunication and 1 for external networking. We will be using eth1 as the external network interface. Update **net\_interfaces.yml** in your my_cloud directory.

```
---
  product:
    version: 2
  interface-models:
    - name: LIFECYCLE-MANAGER-INTERFACES
      network-interfaces:
        - name: eth0
          device:
              name: eth0
          network-groups:
            - MANAGEMENT
# Making changes here are is it needed for the MidoNet Gateways
    - name: CONTROLLER-INTERFACES
      network-interfaces:
        - name: eth0
          device:
              name: eth0
          network-groups:
            - EXTERNAL-API
            - GUEST
            - MANAGEMENT
        - name: eth1
          device:
              name: eth1
          network-groups:
            - EXTERNAL-VM
    - name: COMPUTE-INTERFACES
      network-interfaces:
        - name: eth0
          device:
              name: eth0
          network-groups:
            - EXTERNAL-API
            - GUEST
            - MANAGEMENT
        - name: eth1
          device:
              name: eth1
          network-groups:
            - EXTERNAL-VM
```

### HOS Deployment
Continue and deploy HOS using https://docs.hpcloud.com/#3.x/helion/installation/installing_kvm.html

### MidoNet Deployment
We have created a few Ansible Playbooks to allow you to deploy MidoNet faster and more efficient. 
1. Clone this repo into its own directory.

2. You need the Horizon IP address (VIP), Deployer Node IP and the Horizon admin password.
```
stack@helion-cp1-c0-m1-mgmt:~/hos-midonet_5.2$ grep -E "OS_PASSWORD|OS_AUTH_URL" ../service.osrc
export OS_PASSWORD=WWWWWWWW
export OS_AUTH_URL=https://xx.xx.xx.xx:5000/v3
stack@helion-cp1-c0-m1-mgmt:~/hos-midonet_5.2$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 5c:b9:01:c4:1f:00 brd ff:ff:ff:ff:ff:ff
    inet yy.yy.yy.yy/24 brd 10.246.74.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5eb9:1ff:fec4:1f00/64 scope link
       valid_lft forever preferred_lft forever
stack@helion-cp1-c0-m1-mgmt:~/hos-midonet_5.2$
```
3. Run midonet_deploy.yml with your HOS installation details.
```
ansible-playbook -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts midonet_deploy.yml \
--limit '!localhost' \
-e deployer="<DEPLOYER_IP>" \
-e vip="<KEEPALIVED_VIP>" \
-e admin_password="<KEYSTONE_ADMIN_PASS>"
```

### MidoNet Static Setup Configuration
Next we will need to deploy **midonet\_static.yml** (unless you have BGP Peers which you can refer to steps 5 to 7 at Midonet Official Documentation https://docs.midonet.org/docs/latest-en/quick-start-guide/ubuntu-1404_liberty/content/index.html ). **FOLLOW STEPS BY STEPS.**

1. We need to get the External VLAN ID located at **networks.yml** in my_cloud folder.
```
- name: EXTERNAL-VM-NET
  vlanid: 818
  tagged-vlan: true
  network-group: EXTERNAL-VM
  ```
2. Now we need to create the external network using Neutron API.
```
stack@helion-cp1-c0-m1-mgmt:~$ neutron net-create ext-net --router:external
stack@helion-cp1-c0-m1-mgmt:~$ neutron subnet-create ext-net xx.xx.xx.xx/24 --name ext-subnet --allocation-pool start=xx.xx.xx.xx,end=xx.xx.xx.xx --disable-dhcp --gateway xx.xx.xx.xx
```
3. Now, deploy **midonet_static.yml** with your details. 
```
ansible-playbook -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts midonet_static.yml -e ext_subnet="<EXTERNAL SUBNET>" -e ext_vlan="<EXTERNAL-VM-NET>"
```
- Per example at step 1 and 2 it should look like:
```
ansible-playbook -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts midonet_static.yml -e ext_subnet="xx.xx.xx.xx/24" -e ext_vlan="818"
```
4. Once the deployment is completed, we can verify that EXT VLAN is plugged at the MidoNet External Bridge.
```
stack@helion-cp1-c1-m1-mgmt:~$ EXT_NET=$(midonet-cli -A -e bridge list|awk '/ext-net/ {print $2}')
stack@helion-cp1-c1-m1-mgmt:~$ midonet-cli -A -e bridge $EXT_NET list port
```
- Here is an example, you should see a port for two **controller/gateway** node  "plugged yes". Plugged will indicate the port is attached to a network interface in this case vlan818
```
stack@helion-cp1-c1-m1-mgmt:~$ EXT_NET=$(midonet-cli -A -e bridge list|awk '/ext-net/ {print $2}')
stack@helion-cp1-c1-m1-mgmt:~$ midonet-cli -A -e bridge $EXT_NET list port
port 9dbb204b-a1ad-4623-9923-d6c4c24da878 device 421f1236-562c-4642-a771-e8beb2dcb91d state up plugged no vlan 0 peer 018b1045-68b2-493a-9167-4bf32456c888
port 13ff4404-5e1e-42a0-ade0-323c8fa775b6 device 421f1236-562c-4642-a771-e8beb2dcb91d state up plugged no vlan 0 peer c05b1480-0e31-41c6-b237-e05048f11cc4
port 9ca0f3ad-71a2-4b61-bcc8-7d80399815bd device 421f1236-562c-4642-a771-e8beb2dcb91d state up plugged yes vlan 0
port 6c0782fd-5d41-448c-839d-ec4171aef909 device 421f1236-562c-4642-a771-e8beb2dcb91d state up plugged yes vlan 0
stack@helion-cp1-c1-m1-mgmt:~$
```

## Verification
By now you should be able to create private networks, instances and firewalls. But we can use some of these to verify the deployment.
1. Verify Zookeeper is running on all **controller** nodes.
```
$ ansible -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts NEU-SVR -m shell -a "sudo echo ruok | nc 127.0.0.1 2181"
```
2. Verifiy Cassandra Cluster Nodes are Up and Normal.
```
$ ansible -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts NEU-SVR -m shell -a "sudo nodetool status"
```
3. Verify MidoNet Cluster service is running on all **controllers**.
```
$ ansible -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts NEU-SVR -m shell -a "sudo systemctl status midonet-cluster"
```
4. Verify Midolman service is also running on all **controllers** and **compute** nodes.
```
$ ansible -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts NEU-SVR,entry-scale-kvm-vsa-control-plane-1-compute -m shell -a "sudo systemctl status midolman"
```
5. Verify you can create private networks.
```
neutron net-create demo_net
neutron subnet-create demo_net 172.0.0.0/24 --name demo_subnet
neutron router-create demo_router
neutron router-interface-add demo_router demo_subnet
neutron router-gateway-set demo_router ext-net
```
6. Verify FWaaS is working.
```
stack@helion-cp1-c0-m1-mgmt:~$  neutron firewall-rule-create --protocol tcp --destination-port 22:22 --action deny
Created a new firewall_rule:
+------------------------+--------------------------------------+
| Field                  | Value                                |
+------------------------+--------------------------------------+
| action                 | deny                                 |
| description            |                                      |
| destination_ip_address |                                      |
| destination_port       | 22                                   |
| enabled                | True                                 |
| firewall_policy_id     |                                      |
| id                     | 40456b72-f111-427f-8a70-62d5c5421968 |
| ip_version             | 4                                    |
| name                   |                                      |
| position               |                                      |
| protocol               | tcp                                  |
| shared                 | False                                |
| source_ip_address      |                                      |
| source_port            |                                      |
| tenant_id              | c2d24c29dd7348269b02ab1c9c072c1c     |
+------------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron firewall-policy-create --firewall-rules 40456b72-f111-427f-8a70-62d5c5421968 myfirewallpolicy
Created a new firewall_policy:
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| audited        | False                                |
| description    |                                      |
| firewall_rules | 40456b72-f111-427f-8a70-62d5c5421968 |
| id             | 87a87027-dfea-4bdc-ba40-5ecd8362b11e |
| name           | myfirewallpolicy                     |
| shared         | False                                |
| tenant_id      | c2d24c29dd7348269b02ab1c9c072c1c     |
+----------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron firewall-create 87a87027-dfea-4bdc-ba40-5ecd8362b11e
Created a new firewall:
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| admin_state_up     | True                                 |
| description        |                                      |
| firewall_policy_id | 87a87027-dfea-4bdc-ba40-5ecd8362b11e |
| id                 | 544037d8-90f9-4367-a106-a44ae3424b43 |
| name               |                                      |
| router_ids         | 4ee5b56d-397f-4ffd-addd-3cfc522fcb25 |
|                    | a46b7701-5052-408b-83c0-c63ff2d55b45 |
|                    | e63f0fec-1a81-4707-8fe5-c8708a2e7e29 |
| status             | PENDING_CREATE                       |
| tenant_id          | c2d24c29dd7348269b02ab1c9c072c1c     |
+--------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron firewall-show 544037d8-90f9-4367-a106-a44ae3424b43
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| admin_state_up     | True                                 |
| description        |                                      |
| firewall_policy_id | 87a87027-dfea-4bdc-ba40-5ecd8362b11e |
| id                 | 544037d8-90f9-4367-a106-a44ae3424b43 |
| name               |                                      |
| router_ids         | 4ee5b56d-397f-4ffd-addd-3cfc522fcb25 |
|                    | a46b7701-5052-408b-83c0-c63ff2d55b45 |
|                    | e63f0fec-1a81-4707-8fe5-c8708a2e7e29 |
| status             | ACTIVE                               |
| tenant_id          | c2d24c29dd7348269b02ab1c9c072c1c     |
+--------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ nova list
+--------------------------------------+--------+--------+------------+-------------+----------------------------------+
| ID                                   | Name   | Status | Task State | Power State | Networks                         |
+--------------------------------------+--------+--------+------------+-------------+----------------------------------+
| a277c018-6a09-4d04-af5b-08f6374c075b | lb-01  | ACTIVE | -          | Running     | lb-net=10.1.0.12, 10.246.77.54   |
| 6e81ba06-36b5-4ca5-8bf7-53d82ab98fdb | lb-02  | ACTIVE | -          | Running     | lb-net=10.1.0.13, 10.246.77.55   |
| cfe1cb80-5e11-4734-9ad7-b3c2f0c4df66 | test-1 | ACTIVE | -          | Running     | demo_net=172.0.0.2, 10.246.77.52 |
| 63192e2c-0c97-4ace-82d4-6d097c0abd71 | test-2 | ACTIVE | -          | Running     | demo_net=172.0.0.3, 10.246.77.51 |
+--------------------------------------+--------+--------+------------+-------------+----------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$
```
  
7. The instances with IP addresses 10.246.77.52 and 10.246.77.55 are located in different routers. Since we block SSH port, none should have access.
```
stack@helion-cp1-c0-m1-mgmt:~$ ssh -o ConnectTimeout=5 10.246.77.52
ssh: connect to host 10.246.77.52 port 22: Connection timed out
stack@helion-cp1-c0-m1-mgmt:~$
stack@helion-cp1-c0-m1-mgmt:~$ ssh -o ConnectTimeout=5 10.246.77.55
ssh: connect to host 10.246.77.55 port 22: Connection timed out
stack@helion-cp1-c0-m1-mgmt:~$
```
8. Now we only assign the firewall to be used in the edge_router and lb-router per example above and demo_router should'nt be affected.
```
stack@helion-cp1-c0-m1-mgmt:~$ neutron firewall-update 544037d8-90f9-4367-a106-a44ae3424b43 --router edge_router --router lb-router
Updated firewall: 544037d8-90f9-4367-a106-a44ae3424b43
stack@helion-cp1-c0-m1-mgmt:~$ neutron firewall-show 544037d8-90f9-4367-a106-a44ae3424b43
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| admin_state_up     | True                                 |
| description        |                                      |
| firewall_policy_id | 87a87027-dfea-4bdc-ba40-5ecd8362b11e |
| id                 | 544037d8-90f9-4367-a106-a44ae3424b43 |
| name               |                                      |
| router_ids         | a46b7701-5052-408b-83c0-c63ff2d55b45 |
|                    | e63f0fec-1a81-4707-8fe5-c8708a2e7e29 |
| status             | ACTIVE                               |
| tenant_id          | c2d24c29dd7348269b02ab1c9c072c1c     |
+--------------------+--------------------------------------+
```
- _Notice router ID a46b7701-5052-408b-83c0-c63ff2d55b45(demo_router) is not in the routers using this firewall and SSH traffic should work._
9. Let's verify the SSH connection in the demo_router and lb_router.
```
stack@helion-cp1-c0-m1-mgmt:~$ ssh -o ConnectTimeout=5 10.246.77.52
The authenticity of host '10.246.77.52 (10.246.77.52)' can't be established.
RSA key fingerprint is 76:b9:ee:0a:f7:9a:4f:ec:ac:08:1b:46:c6:aa:84:c8.
Are you sure you want to continue connecting (yes/no)? no
Host key verification failed.
stack@helion-cp1-c0-m1-mgmt:~$ ssh -o ConnectTimeout=5 10.246.77.55
ssh: connect to host 10.246.77.55 port 22: Connection timed out
stack@helion-cp1-c0-m1-mgmt:~$
```
 - _As expected, 10.246.77.55 should still be blocking SSH traffic._
10. Verify LBaaS Pool creation Works. You will need to have a Apache server running in your Members in order to test. Don't forget to have port 80 in the Security Groups
```
stack@helion-cp1-c0-m1-mgmt:~$ neutron subnet-list
+--------------------------------------+-------------+----------------+---------------------------------------------------+
| id                                   | name        | cidr           | allocation_pools                                  |
+--------------------------------------+-------------+----------------+---------------------------------------------------+
| 7812029f-3143-415b-84bd-bd86c24906ca | ext-subnet  | 10.246.77.0/24 | {"start": "10.246.77.50", "end": "10.246.77.250"} |
| 5deb62c2-10e4-4066-bc36-c1ce094ca24f | demo_subnet | 172.0.0.0/24   | {"start": "172.0.0.2", "end": "172.0.0.254"}      |
+--------------------------------------+-------------+----------------+---------------------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ nova list
+--------------------------------------+--------+--------+------------+-------------+--------------------+
| ID                                   | Name   | Status | Task State | Power State | Networks           |
+--------------------------------------+--------+--------+------------+-------------+--------------------+
| 862f9ba5-dd00-485a-a4fc-8fbdd60c5517 | test-1 | ACTIVE | -          | Running     | demo_net=172.0.0.3 |
| b8498785-1fab-4361-8cca-5bdeb8e1f30a | test-2 | ACTIVE | -          | Running     | demo_net=172.0.0.2 |
| 47edb6f4-0620-4d79-84df-97e7bef0816a | test-3 | ACTIVE | -          | Running     | demo_net=172.0.0.4 |
+--------------------------------------+--------+--------+------------+-------------+--------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-pool-create --lb-method ROUND_ROBIN --name Apache --subnet-id 5deb62c2-10e4-4066-bc36-c1ce094ca24f --protocol HTTP --provider midonet
Created a new pool:
+------------------------+--------------------------------------+
| Field                  | Value                                |
+------------------------+--------------------------------------+
| admin_state_up         | True                                 |
| description            |                                      |
| health_monitors        |                                      |
| health_monitors_status |                                      |
| id                     | 37770d6c-0c04-47e1-bc85-62cffaf4cb8b |
| lb_method              | ROUND_ROBIN                          |
| members                |                                      |
| name                   | Apache                               |
| protocol               | HTTP                                 |
| provider               | midonet                              |
| status                 | ACTIVE                               |
| status_description     |                                      |
| subnet_id              | 5deb62c2-10e4-4066-bc36-c1ce094ca24f |
| tenant_id              | eaf3c06d99a241cbbe7dce24dabdc30f     |
| vip_id                 |                                      |
+------------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-member-create --address 172.0.0.2 --protocol-port 80 37770d6c-0c04-47e1-bc85-62cffaf4cb8b
Created a new member:
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| address            | 172.0.0.2                            |
| admin_state_up     | True                                 |
| id                 | 8cd7a9ca-1e9a-4afc-80de-61a07111d7a9 |
| pool_id            | 37770d6c-0c04-47e1-bc85-62cffaf4cb8b |
| protocol_port      | 80                                   |
| status             | PENDING_CREATE                       |
| status_description |                                      |
| tenant_id          | eaf3c06d99a241cbbe7dce24dabdc30f     |
| weight             | 1                                    |
+--------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-member-create --address 172.0.0.3 --protocol-port 80 37770d6c-0c04-47e1-bc85-62cffaf4cb8b
Created a new member:
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| address            | 172.0.0.3                            |
| admin_state_up     | True                                 |
| id                 | 6590b2b5-ebe0-40d9-a28f-8f7fc69f05e4 |
| pool_id            | 37770d6c-0c04-47e1-bc85-62cffaf4cb8b |
| protocol_port      | 80                                   |
| status             | PENDING_CREATE                       |
| status_description |                                      |
| tenant_id          | eaf3c06d99a241cbbe7dce24dabdc30f     |
| weight             | 1                                    |
+--------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-member-create --address 172.0.0.4 --protocol-port 80 37770d6c-0c04-47e1-bc85-62cffaf4cb8b
Created a new member:
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| address            | 172.0.0.4                            |
| admin_state_up     | True                                 |
| id                 | 749d461e-d26f-431d-93dc-c41a8fd83b05 |
| pool_id            | 37770d6c-0c04-47e1-bc85-62cffaf4cb8b |
| protocol_port      | 80                                   |
| status             | PENDING_CREATE                       |
| status_description |                                      |
| tenant_id          | eaf3c06d99a241cbbe7dce24dabdc30f     |
| weight             | 1                                    |
+--------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-member-list
+--------------------------------------+-----------+---------------+--------+----------------+--------+
| id                                   | address   | protocol_port | weight | admin_state_up | status |
+--------------------------------------+-----------+---------------+--------+----------------+--------+
| 6590b2b5-ebe0-40d9-a28f-8f7fc69f05e4 | 172.0.0.3 |            80 |      1 | True           | ACTIVE |
| 749d461e-d26f-431d-93dc-c41a8fd83b05 | 172.0.0.4 |            80 |      1 | True           | ACTIVE |
| 8cd7a9ca-1e9a-4afc-80de-61a07111d7a9 | 172.0.0.2 |            80 |      1 | True           | ACTIVE |
+--------------------------------------+-----------+---------------+--------+----------------+--------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-healthmonitor-create --type HTTP --delay 5 --max-retries 10 --timeout 5
Created a new health_monitor:
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| admin_state_up | True                                 |
| delay          | 5                                    |
| expected_codes | 200                                  |
| http_method    | GET                                  |
| id             | 31b0cb8f-c242-495d-a0f3-cbd78aad19b3 |
| max_retries    | 10                                   |
| pools          |                                      |
| tenant_id      | eaf3c06d99a241cbbe7dce24dabdc30f     |
| timeout        | 5                                    |
| type           | HTTP                                 |
| url_path       | /                                    |
+----------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-healthmonitor-associate 31b0cb8f-c242-495d-a0f3-cbd78aad19b3 37770d6c-0c04-47e1-bc85-62cffaf4cb8b
Associated health monitor 31b0cb8f-c242-495d-a0f3-cbd78aad19b3
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-vip-create --name Apache_VIP --protocol-port 80 --protocol HTTP --subnet-id 7812029f-3143-415b-84bd-bd86c24906ca 37770d6c-0c04-47e1-bc85-62cffaf4cb8b
Created a new vip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| address             | 10.246.77.52                         |
| admin_state_up      | True                                 |
| connection_limit    | -1                                   |
| description         |                                      |
| id                  | 25986c66-7954-4258-a503-e46d708d4ff1 |
| name                | Apache_VIP                           |
| pool_id             | 37770d6c-0c04-47e1-bc85-62cffaf4cb8b |
| port_id             | 0e4295f3-05de-4ab8-ad47-b07a9d8ce001 |
| protocol            | HTTP                                 |
| protocol_port       | 80                                   |
| session_persistence |                                      |
| status              | PENDING_CREATE                       |
| status_description  |                                      |
| subnet_id           | 7812029f-3143-415b-84bd-bd86c24906ca |
| tenant_id           | eaf3c06d99a241cbbe7dce24dabdc30f     |
+---------------------+--------------------------------------+
stack@helion-cp1-c0-m1-mgmt:~$ neutron lb-vip-list
+--------------------------------------+------------+--------------+----------+----------------+--------+
| id                                   | name       | address      | protocol | admin_state_up | status |
+--------------------------------------+------------+--------------+----------+----------------+--------+
| 25986c66-7954-4258-a503-e46d708d4ff1 | Apache_VIP | 10.246.77.52 | HTTP     | True           | ACTIVE |
+--------------------------------------+------------+--------------+----------+----------------+--------+
stack@helion-cp1-c0-m1-mgmt:~$
```
11. Now you should be able to curl the IP and get the hostname as response.
```
[chaudric@server ~]$ curl 10.246.77.52
Welcome test-3
^C
[chaudric@server ~]$ curl 10.246.77.52
Welcome test-2
^C
[chaudric@server ~]$ curl 10.246.77.52
Welcome test-3
^C
[chaudric@server ~]$
```

### Known Issues
There seems to be a network issue when binding a third interface to the **external bridge (ext-net)**. Network seems to get slow and all API calls to OpenStack services and MidoNet CLI becomes slow causing the tasks to get queued. This is the main reason **midonet_static.yml** only binds **ext-net bridge** to **controller2** and **controller3**. We still need **controller1** as it is used for the internal networking using the static routing.