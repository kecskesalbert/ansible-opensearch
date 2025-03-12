# OpenSearch Ansible playbook

## Features
- Cluster deployment on Debian/Ubuntu Linux. Support for multiple clusters (on separate nodes)
- OS configuration management: templates for all OS *.yml config files 
- Security index (users) management
- Admin/Node certificates management
- Index templates and ILM policies management
- Cluster configuration: snapshots, cross-site replication
## Configuration
### Nodes
Add hosts, their roles and optional node attributes to `hosts.ini`
### Cluster settings
Edit cluster configuration in `group_vars/os_cluster_${cluster_name}_backend.yaml`.    
Change os_master_nodes to contain the list of master nodes.
### TLS certificates
TLS certificates are required for transport and API layer. This is done in multiple stages:
1. Root certificate generation
Run playbook with `-e force_certificate=2`. Playbook will stop and print the instructions how to generate root CA private and public keys. 
Open the generated keyfiles. Paste the key contents into root_ca_pem and root_ca_key variables in `group_vars/os_cluster_${cluster_name}_backend.yaml`.
2. Admin certificate generation
Similarly, run playbook with `-e force_certificate=3`. Playbook will stop and print the instructions how to generate admin CA private and public keys. 
Open the generated keyfiles. Paste the key contents into admin_pem and admin_key variables.
3. Node certificate generation
Run playbook with `-e force_certificate=1`. Playbook will (re)generate node certificates. There is no need to make any Ansible configuration edit this time. Each node will have their own certificate delivered.
### Users
Internal (built-in) users are defined in os_internal_users. Other authentication sources can be added to the configuration.
Playbook relies on the existence of 'admin' user for talking to the cluster API. This user should have enough privileges to manage the cluster.
### Load Balancer frontend
Playbook (and probably you also) need a URL to talk to the cluster itself, not directly to the node API ports, especially during node service restart, or when the node API ports are not directly accessible to the Ansible controller. A Load Balancer frontend ensures that API requests are routed to any available node.
Put your LB URL into `cluster_frontend` variable, e.g.: https://os-frontend.entcorp.com:443
In case you don't have a LB frontend, a round-robin DNS will do, e.g. dnsmasq running on localhost. Playbook retries when request times out, so it'll eventually get to an available node. 
## CLI variables
|Variable name|  |
|--|--|
|upgrade|Upgrade mode (one node at a time)|
|force_certificate|Renew root/admin/node certificates. See above.|
|force_deployment|Remove OS binaries, which triggers a reinstall.  |
|force_config|Remove main OS configuration file, which triggers service restart.|

 - OS binaries are deployed when they're not found at the expected location. Otherwise binary installation is skipped, unless forced with force_deployment.
- If there's a change to OS configuration, the service is restarted. Otherwise service restart is skipped, unless forced with force_config.
- By default, playbook runs on all nodes in parallel. Cluster will have some downtime during operation. There's a risk that cluster needs manual intervention when playbook fails. 
In upgrade mode, cluster healthcheck need to pass before the playbook moves to the next node. This ensures cluster operability.
## Tags
|Tag name|  |
|--|--|
|deployment|Deploy OS binaries|
|configuration|Generate certificates and opensearch.yml|
|security|Update internal users|
|management|Update cluster-level settings: Snapshot repositories, ISM policies, index templates|
## Usage
    ansible-playbook -i hosts.ini -e os_cluster_name=rdg_prod opensearch.yml
## Limitations
- Supported OS is Debian/Ubuntu only.
- Multiple clusters are supported, but only on separate nodes. A node can't be part of multiple OS clusters. 
- OS version upgrade works, but not in upgrade mode. In upgrade mode, the node with the new OS version is not accepted to join the cluster. Version upgrade works fine in default (upgrade=0) mode, with downtime.
- Hardcoded admin username
- Hashing internal user passwords with fancy characters might fail. Use \w only to avoid disappointment.
