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


So far, we have successfully set up a <b>multi-node Big Data cluster</b> using several open-source components.As the next step, I plan to <b>install Apache Airflow in High Availability (HA) mode</b> on the cluster.Airflow will be used for both <b>workflow orchestration</b> and as part of the <b>ETL process</b>, aligning with current industry trends for modern data pipelines.

<br/>

## Prerequisites for Setting Up Apache Airflow in HA Mode

<br/>
Before setting up <b>Apache Airflow in High Availability (HA) mode</b>, we need to configure a few <b>supporting components</b> to ensure a reliable and scalable environment.

<br/>

1. <b>Shared DAG Folder (Using Samba)</b>
<br/>
Since we are using a <b>3-node setup</b>, all Airflow nodes must have access to the <b>same DAG folder</b>.Deploying separate copies of the DAG files on each node is <b>not recommended</b>, as it increases complexity and risk of inconsistency.To solve this, I will set up a <b>Linux Samba shared folder</b> that all nodes can mount and use to read/write DAG files centrally.
<br/>

2. <b>Message Broker (RabbitMQ via HAProxy)</b>
<br/>
Airflow also requires a <b>message broker</b> to manage task status and communication between components.By default, Airflow uses <b>Redis</b>, but in this setup, I will use <b>RabbitMQ</b> for better reliability in an HA environment.RabbitMQ will be configured in <b>HA mode</b>, and I will place it behind <b>HAProxy</b> to handle failover and load balancing.

The following steps will walk you through the setup:


## Step 01 : Samba shared folder configuration
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

### csync2 Service

However, <b>csync2</b> is also another good tool to achieve the same purpose.
In this setup, I have used both options, and you can choose whichever you prefer.

One advantage of csync2 is that when one node goes down, the remaining nodes still have the data.
In contrast, with Samba, if the main node goes down, file access will be lost.

The following steps show the installation and configuration of csync2.


```bash
sudo apt-get -y install csync2
```


```bash
nano /etc/csync2.cfg
```

```xml
# Please read the documentation:
# http://oss.linbit.com/csync2/paper.pdf
nossl * *;
tempdir /tmp/;
lock-timeout 30;
  group DAGS
  {
     host node1;
     host node2;
     host node3;
     key /home/airflow/csync2.key_airflow_dags;
     include /home/airflow/airflow/dags;
     auto younger;
  }
```

```bash
csync2 -k csync2.key_airflow_dags
csync2 -xv # Service start
````

```xml
airflow@mst01:~$ crontab -l # every 5 min sync

*/5 * * * * /usr/sbin/csync2 -x > /home/airflow/csync2.log 2>&1

```


<br/>

###  1.4 Verify Installation

You can verify the <b>Samba shared folder</b> setup by creating a sample file on <b>Master 01</b> under the ``` /shared ``` directory. Then, check if the same file appears on <b> Master 02 </b> or <b> Master 03 </b> under the ``` /mnt/shared_smb ``` path.

You can also test the reverse by creating a file from <b> Master 02  or  Master 03  and confirming its availability on  Master 01 </b>. This ensures that the shared folder is properly synchronized and accessible across all nodes.

<br/><br/>

## Step 02 : RabbitMQ HA setup and integration with HAProxy

<br/>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/rabbitmq_logo.png" width="200" height="70">
</picture>

<br/><br/>

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

You can verify the <b>RabbitMQ</b> status using the following commands. We will then configure <b>HAProxy</b> as a load balancer to connect RabbitMQ to <b>Airflow</b> in high availability mode.

```bash
sudo rabbitmqctl cluster_status

sudo rabbitmqctl list_vhosts
sudo rabbitmqctl list_users
sudo rabbitmqctl list_permissions -p airflow
```

<br/><br/>

## Step 03 : Apache Airflow HA setup and integration with HAProxy

<br/>

<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/airflow_logo.png" width="200" height="100">
</picture>

<br/><br/>

The following steps will guide you through the Airflow installation process on an Ubuntu environment.

### 3.1 User creation on Ubuntu and MYSQL

On Each Master's nodes :

```bash
# Add 'airflow' to the 'wheel' group to allow sudo access 
sudo su 
useradd -m -s /bin/bash airflow 
echo "airflow:airflow" | chpasswd 
usermod -aG sudo airflow

su - airflow
```

In this case i am using my pre setuped Mariadb Galera Cluster

```sql
CREATE DATABASE airflow CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'airflow'@'%' IDENTIFIED BY 'airflow';
GRANT ALL PRIVILEGES ON airflow.* TO 'airflow'@'%';
FLUSH PRIVILEGES;
EXIT;
```

### 3.2 Download and Install Airflow on Ubuntu

Airflow required python for running

```bash
sudo apt update
sudo apt install python3 python3-pip python3.12-venv pkg-config default-libmysqlclient-dev build-essential

# Create and activate the python environement

python3 -m venv ~/airflow_venv
source ~/airflow_venv/bin/activate
```

Install Airflow 

```bash
nano ~/.bashrc

export AIRFLOW_HOME=~/airflow
export AIRFLOW__DATABASE__SQL_ALCHEMY_CONN="mysql+mysqldb://airflow:airflow@bigdataproxy:3307/airflow"

source ~/.bashrc

pip install "apache-airflow[celery,mysql,apache.hive]==3.0.3" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-3.0.3/constraints-3.9.txt"
pip install graphviz
pip install flower # This use for celery monitor
```

### 3.3 Configure Airflow

Once Airflow is installed, it does not include any initial configuration files by default.Therefore, run the following command to generate the default configuration files.After that, edit the configuration file and apply the following modifications:

```bash
cd ~/airflow
airflow db migrate

nano $AIRFLOW_HOME/airflow.cfg
```

```xml
sql_alchemy_conn=$AIRFLOW__DATABASE__SQL_ALCHEMY_CONN
port = 3333
broker_url = amqp://airflow:airflow@bigdataproxy:5672/airflow
```

Then rebuild the metastore and start service

```bash
airflow db migrate
```

For full configuration , please check the config file


<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/db_migrate.JPG" width="800" height="200">
</picture>


### 3.4 Start and Verify installation 

You can use the following commands to start and verify the Airflow services.Once the services are up and running, you can access the Airflow UI using the following link:

```bash
nohup airflow api-server > airflow-webserver.log 2>&1 &
nohup airflow scheduler > airflow-scheduler.log 2>&1 &
nohup airflow celery worker > airflow-worker.log 2>&1 &
nohup airflow celery flower > airflow-flower.log 2>&1 &

# kill the service 
pkill -f airflow 

#Other useful comments  
airflow dags list
airflow celery worker

airflow dags reserialize
airflow dags delete data_pipeline_tutorial_v3 # Delete Dag Log
airflow tasks test data_pipeline_tutorial_v3 generate_data 2024-01-01 # Test Dag
```

Web Link :

Login password in the "simple_auth_manager_passwords.json.generated" file , you can modify the password 

http://Node IP:3333/


### Airflow Web
 
<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/airflow_web.JPG" width="800" height="400">
</picture>


### Airflow Flower Web

Airflow Flower is a web-based tool for monitoring and managing Celery workers in Apache Airflow. It provides a real-time dashboard that shows:

Task progress and status
Worker status and resource usage
Queues and routing information
Ability to revoke or retry tasks
Monitoring of task execution times and failures
It's especially useful when you're running Airflow in CeleryExecutor mode, which distributes tasks across multiple workers.

<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/flower_web_1.JPG" width="800" height="400">
</picture>
<br/>
<picture>
  <img alt="docker" src="https://github.com/kavindatk/airflow_setup_3NN_cluster/blob/main/images/flower_web_2.JPG" width="800" height="400">
</picture>
