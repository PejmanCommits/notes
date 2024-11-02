A complete high-availability architecture involves a number of components and processes working together to replicate data. Any organization implementing a high-availability solution should define target metrics for database uptime, switchover recovery time, and acceptable data loss.

Some of the most important concepts involving database high availability are as follows:

- **Data Replication**: Data replication generates multiple copies of the original database data. It logs any database additions and updates and transmits them to all nodes in the HA Cluster. These changes can be database data transactions or alterations to the database schema or table structure. Replication can be either synchronous or asynchronous.
- **High Availability Cluster (HA Cluster)**: A HA Cluster is a collection of nodes that each have a copy of the same underlying data. Having multiple copies of the dataset is essential for data redundancy. Any one of the database servers can respond to queries, and any node can potentially become the master node. From the user’s point of view, the HA Cluster appears as a single database. In most cases, users do not know which node responded to their query.
- **Primary Node**: This is the master node for the HA cluster. It is the recipient of all database changes, including writes and schema updates. Therefore, it always has the most current data set. It replicates these changes to the other instances in the HA cluster, sending them the transactions in either list or stream format. Primary nodes can also handle read requests, but these are typically distributed between the different nodes for load-balancing purposes. The primary node is elected through a *primary election*.
- **Replica Node**: Also known as a *secondary node*, a replica receives updates from the primary node. During regular operation, these nodes can handle read requests. However, depending on the HA architecture, the data in the replica data set might not be completely up to date. Each HA cluster can contain multiple replica nodes for added redundancy and load balancing.
- **Failover**: In the event of a primary node failure, a failover event occurs. One of the secondary nodes becomes the primary node and supervises database updates. Administrators can initiate a manual failover for database maintenance purposes. This scheduled activity is sometimes known as a *manual switchover*. A switch back to the original master is known as a *fallback*.
- **Write-ahead log (WAL)**: This log stores a record of all changes to the database. A unique sequence number identifies each WAL record. In PostgreSQL, the WAL is stored in a *segment file*. A segment file typically contains a large number of records.


### **Methods for Implementing Database Replication**

There are two main forms of data replication and two methods of implementing it. The two main approaches are as follows:

- **Synchronous replication**: In this approach, the primary node waits for confirmation from at least one replica before confirming the transaction. This guarantees the database is consistent across the HA cluster in the event of a failure. Consistency eliminates potential data loss and is vital for organizations that demand transactional data integrity. However, it introduces latency and can reduce throughput.
- **Asynchronous replication**: In asynchronous replication, the primary node sends updates to the replicas without waiting for a response. It immediately confirms a successful commit after updating its own database, reducing latency. However, this approach increases the chances of data loss in the event of an unexpected failover. This is the default PostgreSQL replication method.

The following algorithms are used to implement replication:

- **File-based log shipping**: In this replication method, the primary node asynchronously transmits segment files containing the WAL logs to the replicas. This method cannot be used synchronously because the WAL files build up over a large number of transactions. The primary node continually records all transactions, but the replicas only process the changes after they receive a copy of the file. This is a good approach for latency-sensitive loss-tolerant applications.
- **Streaming replication**: A streaming-based replication algorithm immediately transmits each update to the replicas. The primary node does not have to wait for transactions to build up in the WAL before transmitting the updates. This results in more timely updates on the replicas. Streaming can be either asynchronous, which is the default setting, or synchronous. In both cases, the updates are immediately sent over to the replicas. However, in synchronous streaming, the primary waits for a response from the replicas before confirming the commit. Users can enable synchronous streaming on PostgreSQL through the `sychronous_commit` configuration option.

Another relevant set of concepts relates to how the HA cluster handles a split-brain condition. This occurs when multiple segments of the HA cluster are active but are not able to communicate with each other. In some circumstances, more than one node might attempt to become the primary. To handle this situation, the replication manager structures the rules for a primary election or adds a *quorum*. This problem can also be eliminated through the use of an external monitor.

## **Patroni High Availability Solution**

A specialized replication manager application is almost always used to configure PostgreSQL HA Clusters. These applications automatically handle data replication and node monitoring, which are otherwise very difficult to implement. There are a number of different choices. Each alternative has its own series of strengths and drawbacks. This section explains each of the three most common solutions and compares them.

[Patroni](https://patroni.readthedocs.io/en/latest/) is a Python-based software template for enabling high availability in PostgreSQL databases. This framework requires some template customization to work most effectively. It also requires a *distributed configuration store* (DCS) but supports a number of different storage solutions. Patroni works well on a two-node HA cluster consisting of a primary node and a single replica.

Patroni configures a set of nodes into an HA cluster and configures streaming replication to share updates. It runs an agent on each node in the HA cluster to share node health updates between the members. The primary node is responsible for regularly updating the *leader key*, which is stored in the DCS. If it fails to do so, it is evicted as the primary and another node is elected to take over. After a switchover, the replicas coordinate their position with respect to the database updates. The most up-to-date node typically takes over. In the event of a tie, the first node to create a new leader key wins. Only one node can hold the leader key at any time. This reduces any ambiguity about the identity of the primary node and avoids a split-brain scenario.

Patroni can be installed on Linux nodes using `pip`. Mandatory configuration settings can be configured globally, locally using a YAML file, or through environment variables. The global settings are dynamic and are applied asynchronously to all nodes in the HA cluster. However, local configuration always takes precedence over any global settings. Patroni supports a REST API, which is useful for monitoring and automation purposes. This API is used to determine the status and role of each node in the HA cluster.

**Advantages:**

- It is a mature open-source product.
- It performs very well in standard high-availability test scenarios. It is able to handle more failure scenarios than the alternatives.
- In some circumstances, it is able to restore a failed PostgreSQL process. It also includes a fallback function to restore the HA cluster to a healthy state after failures. This involves initializing the affected node as a replica.
- It enables a standard end-to-end solution on all nodes in the HA cluster based on global configuration settings.
- It has a wide set of features and is highly configurable.
- It includes monitoring functionality.
- The associated REST API permits script access to all attributes.
- It includes watchdog support and callbacks for event notifications.
- It can be integrated with HaProxy, a popular high-performance load balancer.
- Patroni works well with Kubernetes as part of an automated pipeline.
- Storing the leader key in the DCS enforces consensus about the primary node and avoids multiple masters.

**Drawbacks:**

- It is unable to detect a misconfigured replica node.
- It requires manual intervention in a few cases, such as when the Patroni process itself fails.
- It requires a separate DCS application, which must be configured by the user. DCS requires two open communications ports in addition to the main Patroni port.
- Configuration is more complex than the other solutions.
- It uses more memory and CPU than the alternatives.

For more information on Patroni, see the [Patroni website and documentation](https://patroni.readthedocs.io/en/latest/) or [Patroni GitHub](https://github.com/zalando/patroni) .




This implementation consists of 9 Nodes, following best practices of concept to have servies externally. 3 vms for HAproxy and PaceMaker, 3 vms for etcd-server and 3 vms for Postgres servers. all nodes operating systems are debian 12. I also use different LVM to use for services on /data path.


| No. | Hostname  |         Role         | CPU (Cores) | RAM (GB) | Disk (GB) | NIC |       IP        |        OS        |
| :-: | :-------: | :------------------: | :---------: | :------: | :-------: | :-: | :-------------: | :--------------: |
|  1  | dc1-psql-node1 | psql, patroni  |     16      |    16    |    100    |  1  | 192.168.34.124 | Debian 12 |
|  2  | dc1-psql-node2 | psql, patroni  |     16      |    16    |    100    |  1  | 192.168.34.136 | Debian 12 |
|  3  | dc1-psql-node3 | psql, patroni  |     16      |    16    |    100    |  1  | 192.168.34.113 | Debian 12 |
|  4  | hap1 | HAProxy, PaceMaker |      4      |    8     |    25     |  1  | 192.168.34.132 | Debian 12 |
|  5  | hap2 | HAProxy, PaceMaker  |      4      |    8     |    25     |  1  | 192.168.34.133 | Debian 12 |
|  5  | hap2 | HAProxy, PaceMaker  |      4      |    8     |    25     |  1  | 192.168.34.134 | Debian 12 |
|  5  | etcd1 | etcd1  |      4      |    8     |    25     |  1  | 192.168.34.137 | Debian 12 |
|  5  | etcd2 | etcd2  |      4      |    8     |    25     |  1  | 192.168.34.138 | Debian 12 |
|  5  | etcd3 | etcd3  |      4      |    8     |    25     |  1  | 192.168.34.139 | Debian 12 |



The final setup will look like this:

![postgres](https://github.com/user-attachments/assets/6fbd2e9e-865a-40fd-b494-761c6a8da3c6)




## LoadBalancing Layer

in the fist step, we begin to configure loadbalancers. so add nodes and ip addresses to /etc/hosts:

`127.0.0.1       localhost`

`192.168.34.132  node1`

`192.168.34.133  node2`

`192.168.34.134  node3`


To access the multiple hosts using a single interface, we need to create a cluster of LoadBalancer nodes and that is managed by PCS. so now install pcs:

`sudo apt-get install pacemaker corosync crmsh pcs haproxy`

`systemctl enable corosync`

`systemctl enable pacemaker`

`systemctl enable pcsd`


while Installing the PCS and other packages, the package manager also creates a user “hacluster” which is used with PCS for configuring the cluster nodes. and Before we can use PCS we need to set the password for user “hacluster” on all nodes:

`passwd hacluster`

Now using the user “hacluser” and its password we need to authenticate the nodes for PCS cluster.

```
[root@HOST-1 ~]# sudo pcs cluster auth node1 node2 node3
```


we can setup cluster by running:

`sudo pcs cluster setup haproxy node1 addr=192.168.34.132 node2 addr=192.168.34.133 node3 addr=192.168.34.134 --force`

`pcs cluster start --all`

`pcs status cluster`

`pcs property set stonith-enabled=false`
`pcs property set no-quorum-policy=ignore`


### Virtual IP (VIP) and Pacemaker

A Virtual IP (VIP) is an IP address that is not tied to a specific physical network interface or device but can be dynamically assigned to one or more nodes in a high-availability (HA) cluster. In the context of using Pacemaker for HA, a VIP allows clients to connect to a single address regardless of which node is currently active or serving requests. Pacemaker manages the VIP by monitoring the health of the nodes in the cluster. If the primary node fails or becomes unavailable, Pacemaker automatically reassigns the VIP to a standby node, ensuring continuous availability of services. This seamless failover process allows developers and applications to interact with a consistent endpoint, minimizing downtime and enhancing reliability in distributed applications.


now we continue to configure PCS to use VIP. my VIP is 192.168.34.85, so i run:

`sudo pcs resource create virtual_ip ocf:heartbeat:IPaddr2 ip=192.168.34.85 cidr_netmask=24 op monitor interval=30s`

`sudo pcs resource create haproxy systemd:haproxy op monitor interval=10s`

`sudo pcs resource group add HAproxyGroup virtual_ip haproxy`

`sudo pcs constraint order virtual_ip then haproxy`

last command ensures the VIP is assigned on the node, then starts haproxy on it.

then, to check the status of the cluster run:

`sudo pcs status resources`

the output displays the current status of the resources managed by the cluster. This includes information about each resource, such as:

1. **Resource Name**: The name of the resource being monitored.
2. **Resource Type**: The type of resource (e.g., virtual IP, service, etc.).
3. **Current State**: The current status of the resource (e.g., started, stopped, or failed).
4. **Node Location**: The node on which the resource is currently active or running.
5. **Resource Stickiness**: If applicable, it shows how much the resource prefers to stay on its current node.
6. **Fail Count**: The number of times the resource has failed and been restarted.

- `Resource Group: HAproxyGroup:`

    `* virtual_ip        (ocf:heartbeat:IPaddr2):         Started node3`

    `* haproxy   (systemd:haproxy):       Started node3`


after successfully setting up PCS, we configure the HAproxy by editing /etc/haproxy/haproxy.cfg file:

```
# Global configuration settings
global
    # Maximum connections globally
    maxconn 4096
    # Logging settings
    log /data/logs/haproxy local0
    user haproxy
    group haproxy
    daemon

# Default settings
defaults
    # Global log configuration
    log global
    # Number of retries
    retries 2
    # Client timeout
    timeout client 30m
    # Connect timeout
    timeout connect 4s
    # Server timeout
    timeout server 30m
    # Check timeout
    timeout check 5s

# Stats configuration
listen stats
    # Set mode to HTTP
    mode http
    # Bind to port 7000
    bind *:7000
    # Enable stats
    stats enable
    # Stats URI
    stats uri /
    stats auth pejman:**************

# Frontend for Write Requests
listen production
    # Bind to port 5000
    bind *:5000
    # Enable HTTP check
    option httpchk OPTIONS/master
    # Expect status 200
    http-check expect status 200
    # Server settings
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    # Define PostgreSQL servers
    server dc1-psql-node1 192.168.34.124:5432 maxconn 100 check port 8008
    server dc1-psql-node2 192.168.34.136:5432 maxconn 100 check port 8008
    server dc1-psql-node3 192.168.34.113:5432 maxconn 100 check port 8008

# Backend for Standby Databases (for read requests)
listen standby
    bind *:5001
    option httpchk OPTIONS/replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server dc1-psql-node1 192.168.34.124:5432 maxconn 100 check port 8008
    server dc1-psql-node2 192.168.34.136:5432 maxconn 100 check port 8008
    server dc1-psql-node3 192.168.34.113:5432 maxconn 100 check port 8008

# Frontend for Primary Read Requests
frontend read_requests
    bind *:5002
    acl is_primary path_beg /read_primary
    use_backend production if is_primary
    default_backend standby

```

Based on the above configuration: 
- each HAProxy node is waiting for connections to the Primary node on port 5000 and for connections to the replica nodes on port 5001. Therefore, write requests can now be sent to the primary node, while read requests can be distributed in a round-robin manner among the three servers defined as back-end at the end of the configuration. 
- We also utilized HAProxy's path-based routing feature, allowing developers to access the most up-to-date data from the leader by using the `/read_primary` path on port 5002 in the last part of config.
- Additionally, according to the settings of the listen stats block, the HAProxy status dashboard will be accessible on port 7000. we also used authentication to secure the dashboard access.
- In the settings of both the listen production and listen standby blocks, parameters are also specified in the default-server section that determine HAProxy's behavior towards the back-end servers: 
    - The interval between health checks is 3 seconds (inter 3s).
    - If the health check fails three times in a row, that node is considered down (fall 3). 
    - If the health check succeeds two times in a row, that node is considered back up (rise 2). 
    - If a server is considered down, HAProxy immediately closes all sessions for that server. This helps to quickly remove faulty servers and redirect traffic to healthy servers (on-marked-down shutdown-sessions).


To validate your configuration syntax, run:

`haproxy -c -f /etc/haproxy/haproxy.cfg`


Then restart the haproxy resource by running:

`sudo pcs resource restart haproxy`


## etcd cluster

Before configuring the patroni nodes, it is necessary to set up the infrastructure required to record and maintain the status of the Postgres cluster. For this purpose, we use etcd, which is a key-value database, as a distributed configuration store (DCS). In order for this store to be resilient against failures and to continue operating in the event of a node failure, it is necessary to set up an instance of etcd on each of the three nodes and to cluster them together. Note that the etcd cluster uses the Raft algorithm for consensus and leader election, so it is essential that the number of nodes is odd. To install etcd and other required packages, we use the following command on etcd nodes:

`sudo apt install etcd-server etcd-client python3-etcd`

- The etcd-server package contains the etcd daemon binaries. 
- The etcd-client package contains the etcd client binaries. 
- The python3-etcd package is a Python client for interacting with etcd, allowing Python programs to communicate with and manage etcd clusters.


To configure each etcd node, it is necessary to first delete the files located in the /var/lib/etcd/ directory. The presence of default files in this path causes all etcd nodes to be created with the same UUID, and for this reason, they cannot recognize each other as members of a cluster.

`sudo systemctl stop etcd`

```
sudo mkdir /data/etcd
sudo chown -R etcd:etcd /data/
```

`sudo rm -rf /var/lib/etcd/*`

Then we open the file located at etc/default/etcd/ in the editor and add the following lines to it. be sure to change the values according to your setup:

```
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/data/etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.34.138:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://192.168.34.138:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.34.138:2380"
ETCD_INITIAL_CLUSTER="etcd-1=http://192.168.34.137:2380,etcd-2=http://192.168.34.138:2380,etcd-3=http://192.168.34.139:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.34.138:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_ENABLE_V2="true"
```

- in the time of writing this, patroni just supports etcd api v2, so we enabled it above.

I also changed `ETCD_DATA_DIR` in etcd service in order to use `/data/etcd` as data dir:

```
[Unit]
Description=etcd - highly-available key value store
Documentation=https://etcd.io/docs
Documentation=man:etcd
After=network.target
Wants=network-online.target

[Service]
Environment=DAEMON_ARGS=
Environment=ETCD_NAME=%H
Environment=ETCD_DATA_DIR=/data/etcd
EnvironmentFile=-/etc/default/%p
Type=notify
User=etcd
PermissionsStartOnly=true
#ExecStart=/bin/sh -c "GOMAXPROCS=$(nproc) /usr/bin/etcd $DAEMON_ARGS"
ExecStart=/usr/bin/etcd $DAEMON_ARGS
Restart=on-abnormal
#RestartSec=10s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd2.service
```

Then restart the etcd-server:

`sudo systemctl restart etcd`


## DataBase Layer

We will use version 17 of PostgreSQL. To install this version, we first need to add the PostgreSQL APT repository to the machines. According to the PostgreSQL documentation, we proceed as follows:

```
# Import the repository signing key:
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Create the repository configuration file:
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```


After creating pdgd.list, we update the APT cache once and install it alongside patroni on all postgres nodes:

```
sudo apt update
sudo apt -y install postgresql-17 postgresql-server-dev-17 patroni etcd-client etcd-python3 python3-psycopg2
```


Since we need to delegate the control of Postgres to Patroni, we will stop the currently running service that has been started automatically:

```
which psql
sudo systemctl stop postgresql
sudo systemctl stop patroni
```


Patroni uses some of the tools that are installed with Postgres, so it is necessary to create a symbolic link (symlink) from its binaries in the /usr/sbin/ directory to ensure that Patroni will have access to them:


`sudo ln -s /usr/lib/postgresql/17/bin/* /usr/sbin/`


Now according to [Patroni Docs](https://github.com/patroni/patroni/blob/master/postgres0.yml) we will config Patroni using `etc/patroni/config.yml/` file. mine looks like this:

```
scope: postgres
namespace: /db/
name: node1

restapi:
    listen: 192.168.34.124:8008
    connect_address: 192.168.34.124:8008

etcd:
    hosts: 192.168.34.137:2379,192.168.34.138:2379,192.168.34.139:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: hot_standby # or replica
        hot_standby: "on"
        wal_keep_segments: 8
        max_wal_senders: 5
        max_replication_slots: 5
        max_connections: 100
        max_worker_processes: 8
        max_locks_per_transaction: 64
        wal_log_hints: "on"
        track_commit_timestamp: "off"
        archive_mode: "on"
        archive_timeout: 1800s
        # Command to archive WAL files. This command creates a directory named 'wal_archive', checks if the file doesn't already exist, and then copies it
        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
      recovery_conf:
        # Command used to retrieve archived WAL files during recovery. It copies files from the 'wal_archive' directory.
        restore_command: cp ../wal_archive/%f %p

  initdb:
  - auth: scram-sha-256
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - host replication replicator 127.0.0.1/32 scram-sha-256
  - host replication replicator 192.168.34.124/0 scram-sha-256
  - host replication replicator 192.168.34.136/0 scram-sha-256
  - host replication replicator 192.168.34.113/0 scram-sha-256
  - host all all 0.0.0.0/0 scram-sha-256

  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 192.168.34.124:5432 # This can remain as is for local connections
  connect_address: 192.168.34.124:5432 # or Point to PgBouncer if you have it externally
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: ***********
    superuser:
      username: postgres
      password: ************
  parameters:
      unix_socket_directories: '.'
      max_connections: 100
      shared_buffers: 512MB
      wal_level: replica
      hot_standby: "on"
      max_wal_senders: 5
      max_replication_slots: 5
      password_encryption: 'scram-sha-256'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

- We added the following configurations to the bootstrap section. In this section, we define what default values and settings the Postgres node should use during the initial startup. We also configure how Patroni interacts with the distributed configuration store (DCS), or etcd, here.
- Next, we determined how to authenticate the clients of the cluster (pg_hba.conf) and allow the user 'replicator', which is used for replication between the nodes of the cluster, to access the PostgreSQL cluster nodes using the scram-sha-256 mechanism, which is the most secure method of authentication via password. We also set the password for the admin user in this section.
- Then, we entered the remaining configurations for Postgres and Patroni, such as the Postgres address on each node, the password for the replicator and postgres users, etc., and saved the changes.


After editing the config.yml file, it is necessary to transfer the ownership of the directory we designated for storing Patroni data (/data/patroni/) to the postgres user and restrict read and write access to only the owner of the directory:

```
sudo mkdir -p /data/patroni

sudo chown -R postgres:postgres /data/patroni

sudo chmod 700 /data/patroni
```

also check patroni service to ensure it is using the correct config file in `ExecStart` section:

`ExecStart=/usr/bin/patroni /etc/patroni/config.yml`


Now we can start the Patroni service on all related nodes:

`sudo systemctl restart patroni`


If it becomes necessary to make changes to the parameters in the bootstrap.dcs section after the initial setup, we should use the command `patronictl edit-config`.

To perform a final check on the PostgreSQL nodes managed by Patroni, execute the following command on one of the nodes:

`patronictl -c /etc/patroni/config.yml list`


the output should be something like this:

```
+ Cluster: postgres (7431525978089455740) ------+----+-----------+
| Member | Host           | Role    | State     | TL | Lag in MB |
+--------+----------------+---------+-----------+----+-----------+
| node1  | 192.168.34.124 | Leader  | running   |  2 |           |
| node2  | 192.168.34.136 | Replica | streaming |  2 |         0 |
| node3  | 192.168.34.113 | Replica | streaming |  2 |         0 |
+--------+----------------+---------+-----------+----+-----------+
```

Finally, we disable the automatic execution of the Postgres service after the system reboots so that the control of the Postgres cluster remains with Patroni. If the execution of this command is successful, there will be no specific output:

`sudo systemctl disable --now postgresql`


now if we go to the VIP address on port 7000, the haproxy dashboard will display the status of nodes.
    - one leader in production
    - two replica nodes listening for read operations in standby section except the leader
    - and read requests to the leader node if is_primary request has been used

![Bildschirmfoto_20241031_113654](https://github.com/user-attachments/assets/80777502-db25-4e43-a984-7c3cf422b2e6)



## sources

- https://www.linode.com/docs/guides/comparison-of-high-availability-postgresql-solutions/
- https://github.com/libremomo/devops-notes/blob/main/HA-Postgres-Cluster/Step-by-Step-Guide-to-Deploy-a-Highly-Available-Postgres-Cluster.md?plain=1
- https://adamtheautomator.com/patroni/
- https://www.linode.com/docs/guides/create-a-highly-available-postgresql-cluster-using-patroni-and-haproxy/
