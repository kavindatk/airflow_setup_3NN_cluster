# Part 5: Apache Airflow HA mode

<br/><br/>
<p align="center">
<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/airflow_logo.png" width="300" height="125">
</picture>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/rabbitmq_logo.png" width="300" height="70">
</picture>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/samba_log.png" width="200" height="125">
</picture>
</p>


<br/><br/>

## Next Step: Installing Apache Airflow in HA Mode


So far, we have successfully set up a <b>multi-node Big Data cluster</b> using several open-source components.

As the next step, I plan to <b>install Apache Airflow in High Availability (HA) mode</b> on the cluster.

Airflow will be used for both <b>workflow orchestration</b> and as part of the <b>ETL process</b>, aligning with current industry trends for modern data pipelines.

<br/>

## Prerequisites for Setting Up Apache Airflow in HA Mode

<br/>
Before setting up <b>Apache Airflow in High Availability (HA) mode</b>, we need to configure a few <b>supporting components</b> to ensure a reliable and scalable environment.

<br/>

1. <b>Shared DAG Folder (Using Samba)</b>
<br/>
Since we are using a <b>3-node setup</b>, all Airflow nodes must have access to the <b>same DAG folder</b>.
Deploying separate copies of the DAG files on each node is <b>not recommended</b>, as it increases complexity and risk of inconsistency.

To solve this, I will set up a <b>Linux Samba shared folder</b> that all nodes can mount and use to read/write DAG files centrally.
<br/>

2. <b>Message Broker (RabbitMQ via HAProxy)</b>
<br/>
Airflow also requires a <b>message broker</b> to manage task status and communication between components.

By default, Airflow uses <b>Redis</b>, but in this setup, I will use <b>RabbitMQ</b> for better reliability in an HA environment.
RabbitMQ will be configured in <b>HA mode</b>, and I will place it behind <b>HAProxy</b> to handle failover and load balancing.

The following steps will walk you through the setup:


### Step 01 : Samba shared folder configuration
<br/>
<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/airflow_logo.png" width="200" height="100">
</picture>
<br/>

The following steps will guide you through the samba installation process on an Ubuntu environment.

<br/>
### 1.1 Install samba on ubuntu environment 

On Master 01 Node : 

```bash
sudo apt-get install samba
```

```bash
sudo mkdir /shared
sudo chmod 777 /shared
```


### 1.2 Configure samba 

```bash
# Edit smb.conf
sudo nano /etc/samba/smb.conf
```

#### Add or modify following file

```xml
[shared]
   path = /shared
   browseable = yes
   writable = yes
   guest ok = yes
   force user = nobody
   force group = nogroup
   create mask = 0777
   directory mask = 0777
```


### 1.3 Start samba Service

On Master 01 Node , run following command 

```bash
# Start services
sudo systemctl enable --now smbd
sudo systemctl enable --now nmbd

sudo systemctl start --now smbd
sudo systemctl start --now nmbd
```

On Master 02/03 Nodes : 

```bash
sudo apt-get install cifs-utils
```

```bash
sudo mkdir /mnt/shared_smb
sudo mount -t cifs //mst01/shared /mnt/shared_smb -o guest,uid=1000,gid=1000,dir_mode=0777,file_mode=0777

sudo umount /mnt/shared_smb # unmount shared folder
 
```

<br/>

###  1.4 Verify Installation

You can verify the <b>Samba shared folder</b> setup by creating a sample file on <b>Master 01</b> under the ``` /shared ``` directory.
Then, check if the same file appears on <b> Master 02 </b> or <b> Master 03 </b> under the ``` /mnt/shared_smb ``` path.

You can also test the reverse by creating a file from <b> Master 02  or  Master 03  and confirming its availability on  Master 01 </b>.
This ensures that the shared folder is properly synchronized and accessible across all nodes.

<br/><br/>

### Step 02 : RabbitMQ HA setup and integration with HAProxy

<br/>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/rabbitmq_logo.png" width="200" height="70">
</picture>

<br/>

The following steps will guide you through the rabbitmq installation process on an Ubuntu environment.

<br/>
### 2.1 Install rabbitmq on ubuntu environment 

On Master Nodes : 

```bash
sudo apt-get install rabbitmq-server 
```

### 2.2 Configure rabbitmq 

```bash
nano /etc/rabbitmq/rabbitmq.conf
```


#### Add or modify following file


```xml
cluster_formation.peer_discovery_backend = classic_config
cluster_formation.classic_config.nodes.1 = rabbit@mst01
cluster_formation.classic_config.nodes.2 = rabbit@mst02
cluster_formation.classic_config.nodes.3 = rabbit@mst03

cluster_name = airflow_rmq_cluster
loopback_users.guest = false
```


### 2.3 Start rabbitmq Service

On each Master's  Node , run following command 

```bash
# Start services
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```

On Master 02/03 Nodes : 

```bash
sudo rabbitmqctl stop_app
sudo rabbitmqctl join_cluster rabbit@<first-node-hostname>
sudo rabbitmqctl start_app
```

Enable Queue Mirroring and create airflow user

```bash

sudo rabbitmqctl add_vhost airflow
sudo rabbitmqctl add_user airflow 
sudo rabbitmqctl set_permissions -p airflow airflow ".*" ".*" ".*"


rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}' --apply-to queues
```

#### Special configuration (important)

```bash
# On mast01
sudo cat /var/lib/rabbitmq/.erlang.cookie

# copy this key to other masters 
# Ex : 
echo 'emclO8dcP+EwaSHk1jJZQQ==' | sudo tee /var/lib/rabbitmq/.erlang.cookie
sudo chmod 400 /var/lib/rabbitmq/.erlang.cookie
sudo chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie


mst02 and mst03
sudo systemctl restart rabbitmq-server
```

<br/>

###  2.4 Verify Installation

You can verify the <b>RabbitMQ</b> status using the following commands.
We will then configure <b>HAProxy</b> as a load balancer to connect RabbitMQ to <b>Airflow</b> in high availability mode.

```bash
sudo rabbitmqctl cluster_status

sudo rabbitmqctl list_vhosts
sudo rabbitmqctl list_users
sudo rabbitmqctl list_permissions -p airflow
```

<br/><br/>

