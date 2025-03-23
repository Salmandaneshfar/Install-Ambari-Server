# Complete Guide to Installing and Kerberizing Hadoop Ecosystem Services with Apache Ambari

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Ambari Server Installation](#ambari-server-installation)
- [Hadoop Cluster Setup](#hadoop-cluster-setup)
- [Kerberos Implementation](#kerberos-implementation)
- [Kerberizing Hadoop Services](#kerberizing-hadoop-services)
- [Kerberizing Additional Ecosystem Components](#kerberizing-additional-ecosystem-components)
- [Post-Kerberos Configuration](#post-kerberos-configuration)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## Introduction

This guide provides comprehensive instructions for installing, configuring, and securing a Hadoop cluster using Apache Ambari with Kerberos authentication. Kerberos is an authentication protocol that provides strong security for services in a Hadoop cluster by ensuring that communications are encrypted and authenticated.

### About Apache Ambari

Apache Ambari is an open-source administration tool that simplifies managing, monitoring, and provisioning Hadoop clusters. It provides a web-based interface for installing and managing Hadoop components like HDFS, YARN, MapReduce, Hive, HBase, and more.

### About Kerberos

Kerberos is a network authentication protocol that uses tickets to allow nodes communicating over a non-secure network to prove their identity securely. In a Hadoop environment, Kerberos ensures that:
- Only authenticated users and services can access the cluster
- Communications between services are encrypted
- Impersonation attacks are prevented

## Prerequisites

### Hardware Requirements
- **Master Nodes**: 16+ GB RAM, 8+ CPU cores, 100+ GB storage
- **Worker Nodes**: 8+ GB RAM, 4+ CPU cores, 1+ TB storage
- **Edge Nodes**: 8+ GB RAM, 4+ CPU cores, 100+ GB storage

### Software Requirements
- Operating System: CentOS/RHEL 7.x (64-bit)
- Java: OpenJDK 1.8
- PostgreSQL (for Ambari and service metadata stores)
- Browser: Chrome, Firefox (latest versions)

### Network Requirements
- All nodes should have static IP addresses
- All nodes must be able to communicate with each other
- Forward and reverse DNS must be properly configured for all hosts
- NTP must be configured to maintain clock synchronization

### Prepare All Nodes

1. **Set up hosts file on all nodes**:
   ```bash
   vi /etc/hosts
   # Add entries for all nodes in your cluster
   192.168.1.100 ambari-server.example.com ambari-server
   192.168.1.101 master1.example.com master1
   192.168.1.102 master2.example.com master2
   192.168.1.103 worker1.example.com worker1
   192.168.1.104 worker2.example.com worker2
   192.168.1.105 worker3.example.com worker3
   ```

2. **Disable SELinux on all nodes**:
   ```bash
   # Edit SELinux config
   vi /etc/selinux/config
   
   # Set SELINUX to disabled
   SELINUX=disabled
   
   # Check current status
   getenforce
   
   # Temporarily disable until next reboot
   setenforce 0
   ```

3. **Disable firewall or configure appropriate ports**:
   ```bash
   # Disable firewall (for testing only, not recommended for production)
   systemctl stop firewalld
   systemctl disable firewalld
   
   # For production, configure specific ports instead of disabling the firewall
   ```

4. **Set up password-less SSH from Ambari server to all nodes**:
   ```bash
   # On Ambari server, generate SSH key
   ssh-keygen -t rsa
   
   # Copy the key to all nodes, including Ambari server itself
   for host in ambari-server master1 master2 worker1 worker2 worker3; do
     ssh-copy-id root@$host
   done
   
   # Test SSH connectivity
   for host in ambari-server master1 master2 worker1 worker2 worker3; do
     ssh root@$host hostname
   done
   ```

5. **Install NTP and synchronize time on all nodes**:
   ```bash
   # Install NTP
   yum install -y ntp
   
   # Start and enable NTP service
   systemctl start ntpd
   systemctl enable ntpd
   
   # Verify time synchronization
   ntpq -p
   ```

6. **Install Java on all nodes**:
   ```bash
   # Install OpenJDK 1.8
   yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel
   
   # Set JAVA_HOME in /etc/profile
   echo 'export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk' >> /etc/profile
   echo 'export PATH=$JAVA_HOME/bin:$PATH' >> /etc/profile
   source /etc/profile
   
   # Verify Java installation
   java -version
   ```

7. **Configure max open file descriptors**:
   ```bash
   # Add to /etc/security/limits.conf
   echo "* soft nofile 10000" >> /etc/security/limits.conf
   echo "* hard nofile 10000" >> /etc/security/limits.conf
   
   # Verify settings
   ulimit -Sn
   ulimit -Hn
   ```

## Ambari Server Installation

### Set Up Local Repository (Optional)

If you're in an environment without internet access, set up a local repository:

```bash
# Create mount point for DVD/ISO
mkdir /mnt/cdrom
mount /dev/cdrom /mnt/cdrom

# Create yum repository file
cat > /etc/yum.repos.d/ambari.repo << EOF
[Ambari]
name=Ambari
baseurl=http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.5.0
gpgcheck=0
enabled=1

[HDP]
name=HDP
baseurl=http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.5.0
gpgcheck=0
enabled=1

[HDP-UTILS]
name=HDP-UTILS
baseurl=http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7
gpgcheck=0
enabled=1
EOF

# Clean and update yum cache
yum clean all
yum repolist
```

### Install Ambari Server

1. **Install Ambari Server package**:
   ```bash
   yum install -y ambari-server
   ```

2. **Set up Ambari Server**:
   ```bash
   # Run setup command
   ambari-server setup
   
   # Select the JDK option (Custom JDK)
   # Provide the path to JAVA_HOME (/usr/lib/jvm/java-1.8.0-openjdk)
   # Accept all other default settings
   ```

3. **Configure PostgreSQL for Ambari (recommended for production)**:
   ```bash
   # Install PostgreSQL
   yum install -y postgresql-server postgresql-contrib
   
   # Initialize the database
   postgresql-setup initdb
   
   # Start and enable PostgreSQL
   systemctl start postgresql
   systemctl enable postgresql
   
   # Configure client authentication
   vi /var/lib/pgsql/data/pg_hba.conf
   
   # Change authentication method to "md5" for all connections
   # Example:
   # host    all             all             127.0.0.1/32            md5
   # host    all             all             ::1/128                 md5
   
   # Restart PostgreSQL
   systemctl restart postgresql
   
   # Create Ambari database and user
   sudo -u postgres psql << EOF
   CREATE DATABASE ambari;
   CREATE USER ambari WITH PASSWORD 'bigdata';
   GRANT ALL PRIVILEGES ON DATABASE ambari TO ambari;
   \q
   EOF
   
   # Configure Ambari to use PostgreSQL
   ambari-server setup --database=postgresql --databasehost=localhost --databaseport=5432 --databasename=ambari --databaseusername=ambari --databasepassword=bigdata
   ```

4. **Start Ambari Server**:
   ```bash
   ambari-server start
   
   # Check Ambari Server status
   ambari-server status
   ```

5. **Access Ambari Web UI**:
   - Open a browser and navigate to: `http://ambari-server.example.com:8080`
   - Default credentials: admin/admin
   - Change the password after first login

## Hadoop Cluster Setup

### Install Cluster Using Ambari Web UI

1. **Launch Install Wizard**:
   - Click "Launch Install Wizard" on the Ambari welcome page

2. **Name Your Cluster**:
   - Provide a name for your cluster (e.g., "HadoopCluster")

3. **Select Stack**:
   - Choose HDP version (recommended: HDP 3.1.5.0)
   - Select "Use Local Repository" if you set up local repositories

4. **Install Options**:
   - Enter the FQDN of all hosts (or upload a hosts file)
   - Provide SSH private key or password for authentication
   - Verify hosts by clicking "Register and Confirm"

5. **Choose Services**:
   - Select services to install. Recommended services:
     - HDFS
     - YARN + MapReduce2
     - Tez
     - Hive
     - HBase
     - Pig
     - Sqoop
     - Oozie
     - ZooKeeper
     - Ambari Metrics
     - SmartSense
     - Kafka
     - Spark
     - Spark2
     - Storm
     - Ranger
     - Atlas
     - Zeppelin Notebook

6. **Assign Masters**:
   - Distribute master components across master nodes
   - Recommended configuration:
     - **Master1**: NameNode, YARN ResourceManager, HBase Master, Spark History Server
     - **Master2**: Secondary NameNode, YARN Timeline Server, Hive Server, Oozie Server

7. **Assign Slaves and Clients**:
   - Select worker nodes for DataNode, NodeManager, and RegionServer
   - Install clients on all nodes where needed

8. **Customize Services**:
   - Configure service-specific settings
   - Pay special attention to:
     - Database settings for Hive, Oozie, Ranger
     - Directory configurations for HDFS, YARN, and HBase
     - Memory allocations based on your hardware

9. **Review**:
   - Review configurations and verify all settings
   - Click "Deploy" to start the installation

10. **Monitor Installation**:
    - The installation process will run for 30-60 minutes depending on cluster size
    - Resolve any issues that arise during installation
    - Click "Complete" when installation finishes

## Kerberos Implementation

### Set Up KDC (Key Distribution Center)

1. **Install Kerberos Server Packages**:
   ```bash
   # On the KDC server (can be the Ambari server or a dedicated node)
   yum install -y krb5-server krb5-libs krb5-workstation
   
   # On all other nodes
   yum install -y krb5-workstation krb5-libs
   ```

2. **Configure Kerberos Server**:
   ```bash
   # Edit KDC configuration
   vi /etc/krb5.conf
   
   # Example configuration:
   [libdefaults]
    default_realm = EXAMPLE.COM
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    udp_preference_limit = 1
    default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac
    default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac
    permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac
   
   [realms]
    EXAMPLE.COM = {
     kdc = kdc.example.com
     admin_server = kdc.example.com
    }
   
   [domain_realm]
    .example.com = EXAMPLE.COM
    example.com = EXAMPLE.COM
   ```

3. **Configure KDC Server**:
   ```bash
   # Edit KDC config
   vi /var/kerberos/krb5kdc/kdc.conf
   
   # Example configuration:
   [kdcdefaults]
    kdc_ports = 88
    kdc_tcp_ports = 88
   
   [realms]
    EXAMPLE.COM = {
     #master_key_type = aes256-cts
     acl_file = /var/kerberos/krb5kdc/kadm5.acl
     dict_file = /usr/share/dict/words
     admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
     supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
     max_renewable_life = 7d
    }
   ```

4. **Create Kerberos Database**:
   ```bash
   # Create KDC database
   kdb5_util create -s -r EXAMPLE.COM
   
   # Enter KDC database master password when prompted
   ```

5. **Configure ACL for Kadmin**:
   ```bash
   # Edit kadm5.acl
   vi /var/kerberos/krb5kdc/kadm5.acl
   
   # Add the following line:
   */admin@EXAMPLE.COM *
   ```

6. **Start Kerberos Services**:
   ```bash
   # Start and enable KDC and kadmin services
   systemctl start krb5kdc
   systemctl enable krb5kdc
   systemctl start kadmin
   systemctl enable kadmin
   
   # Verify services are running
   systemctl status krb5kdc
   systemctl status kadmin
   ```

7. **Create Admin Principal**:
   ```bash
   # Create admin principal
   kadmin.local -q "addprinc admin/admin@EXAMPLE.COM"
   
   # Enter password when prompted
   ```

8. **Verify Kerberos Installation**:
   ```bash
   # Test kinit with admin user
   kinit admin/admin@EXAMPLE.COM
   
   # List existing principals
   kadmin.local -q "listprincs"
   
   # Destroy ticket after testing
   kdestroy
   ```

9. **Distribute Kerberos Configuration to All Nodes**:
   ```bash
   # Copy /etc/krb5.conf to all nodes
   for host in master1 master2 worker1 worker2 worker3; do
     scp /etc/krb5.conf root@$host:/etc/
   done
   ```

## Kerberizing Hadoop Services

### Enable Kerberos Through Ambari

1. **Access Kerberos Wizard**:
   - In Ambari Web UI, go to Admin -> Kerberos
   - Click "Enable Kerberos" to launch the wizard

2. **Configure Kerberos**:
   - Select "Existing MIT KDC" as the KDC type
   - Enter KDC host: kdc.example.com
   - Enter Realm name: EXAMPLE.COM
   - Enter KDC admin account: admin/admin@EXAMPLE.COM
   - Enter admin password
   - Proceed to the next step

3. **Configure Identities**:
   - Review default Kerberos identity settings
   - Make any necessary adjustments to principal names or service users
   - Click "Next" to proceed

4. **Confirm Configuration**:
   - Review the configuration before proceeding
   - Click "Next" to begin Kerberization

5. **Kerberize Cluster**:
   - Ambari will perform the following actions:
     - Create necessary service principals
     - Generate keytab files
     - Distribute keytabs to appropriate hosts
     - Update service configurations
     - Restart services

6. **Complete Kerberization**:
   - Review the completion summary
   - Click "Complete" to finish the Kerberos setup

### Manual Principal and Keytab Creation (Alternative Method)

If you encounter issues with Ambari's Kerberos wizard, you can create principals and keytabs manually:

1. **Create HDFS Principals**:
   ```bash
   # Login to kadmin
   kadmin.local
   
   # Create NameNode principal
   addprinc -randkey nn/master1.example.com@EXAMPLE.COM
   
   # Create DataNode principals for each worker node
   addprinc -randkey dn/worker1.example.com@EXAMPLE.COM
   addprinc -randkey dn/worker2.example.com@EXAMPLE.COM
   addprinc -randkey dn/worker3.example.com@EXAMPLE.COM
   
   # Create HTTP principals (needed for web UIs)
   addprinc -randkey HTTP/master1.example.com@EXAMPLE.COM
   addprinc -randkey HTTP/master2.example.com@EXAMPLE.COM
   addprinc -randkey HTTP/worker1.example.com@EXAMPLE.COM
   addprinc -randkey HTTP/worker2.example.com@EXAMPLE.COM
   addprinc -randkey HTTP/worker3.example.com@EXAMPLE.COM
   
   # Exit kadmin
   quit
   ```

2. **Create Keytab Files**:
   ```bash
   # Login to kadmin
   kadmin.local
   
   # Create NameNode keytab
   ktadd -k /etc/security/keytabs/nn.service.keytab nn/master1.example.com@EXAMPLE.COM HTTP/master1.example.com@EXAMPLE.COM
   
   # Create DataNode keytabs
   ktadd -k /etc/security/keytabs/dn.service.keytab.worker1 dn/worker1.example.com@EXAMPLE.COM HTTP/worker1.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/dn.service.keytab.worker2 dn/worker2.example.com@EXAMPLE.COM HTTP/worker2.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/dn.service.keytab.worker3 dn/worker3.example.com@EXAMPLE.COM HTTP/worker3.example.com@EXAMPLE.COM
   
   # Exit kadmin
   quit
   ```

3. **Distribute Keytabs**:
   ```bash
   # Create directory for keytabs on all nodes
   for host in master1 master2 worker1 worker2 worker3; do
     ssh root@$host "mkdir -p /etc/security/keytabs; chmod 750 /etc/security/keytabs"
   done
   
   # Copy NameNode keytab to master1
   scp /etc/security/keytabs/nn.service.keytab root@master1.example.com:/etc/security/keytabs/
   
   # Copy DataNode keytabs to worker nodes
   scp /etc/security/keytabs/dn.service.keytab.worker1 root@worker1.example.com:/etc/security/keytabs/dn.service.keytab
   scp /etc/security/keytabs/dn.service.keytab.worker2 root@worker2.example.com:/etc/security/keytabs/dn.service.keytab
   scp /etc/security/keytabs/dn.service.keytab.worker3 root@worker3.example.com:/etc/security/keytabs/dn.service.keytab
   
   # Set proper ownership and permissions
   for host in master1 worker1 worker2 worker3; do
     ssh root@$host "chown hdfs:hadoop /etc/security/keytabs/*.keytab; chmod 400 /etc/security/keytabs/*.keytab"
   done
   ```

## Kerberizing Additional Ecosystem Components

### YARN and MapReduce2

1. **Create Principals**:
   ```bash
   # Login to kadmin
   kadmin.local
   
   # Create YARN ResourceManager principal
   addprinc -randkey rm/master1.example.com@EXAMPLE.COM
   
   # Create YARN NodeManager principals
   addprinc -randkey nm/worker1.example.com@EXAMPLE.COM
   addprinc -randkey nm/worker2.example.com@EXAMPLE.COM
   addprinc -randkey nm/worker3.example.com@EXAMPLE.COM
   
   # Create JobHistory Server principal
   addprinc -randkey jhs/master2.example.com@EXAMPLE.COM
   
   # Create keytabs
   ktadd -k /etc/security/keytabs/rm.service.keytab rm/master1.example.com@EXAMPLE.COM HTTP/master1.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/nm.service.keytab.worker1 nm/worker1.example.com@EXAMPLE.COM HTTP/worker1.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/nm.service.keytab.worker2 nm/worker2.example.com@EXAMPLE.COM HTTP/worker2.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/nm.service.keytab.worker3 nm/worker3.example.com@EXAMPLE.COM HTTP/worker3.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/jhs.service.keytab jhs/master2.example.com@EXAMPLE.COM HTTP/master2.example.com@EXAMPLE.COM
   
   # Exit kadmin
   quit
   ```

2. **Configure YARN for Kerberos**:
   In Ambari UI:
   - Go to YARN -> Configs -> Advanced
   - Set `hadoop.security.authentication` to `kerberos`
   - Set `hadoop.security.authorization` to `true`
   - Configure principal and keytab paths:
     - ResourceManager: `/etc/security/keytabs/rm.service.keytab`
     - NodeManager: `/etc/security/keytabs/nm.service.keytab`
     - JobHistory Server: `/etc/security/keytabs/jhs.service.keytab`

### HBase

1. **Create Principals**:
   ```bash
   # Login to kadmin
   kadmin.local
   
   # Create HBase Master principal
   addprinc -randkey hbase/master1.example.com@EXAMPLE.COM
   
   # Create HBase RegionServer principals
   addprinc -randkey hbase/worker1.example.com@EXAMPLE.COM
   addprinc -randkey hbase/worker2.example.com@EXAMPLE.COM
   addprinc -randkey hbase/worker3.example.com@EXAMPLE.COM
   
   # Create keytabs
   ktadd -k /etc/security/keytabs/hbase.master.keytab hbase/master1.example.com@EXAMPLE.COM HTTP/master1.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/hbase.regionserver.keytab.worker1 hbase/worker1.example.com@EXAMPLE.COM HTTP/worker1.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/hbase.regionserver.keytab.worker2 hbase/worker2.example.com@EXAMPLE.COM HTTP/worker2.example.com@EXAMPLE.COM
   ktadd -k /etc/security/keytabs/hbase.regionserver.keytab.worker3 hbase/worker3.example.com@EXAMPLE.COM HTTP/worker3.example.com@EXAMPLE.COM
   
   # Exit kadmin
   quit
   ```

2. **Configure HBase for Kerberos**:
   In Ambari UI:
   - Go to HBase -> Configs -> Advanced
   - Set `hbase.security.authentication` to `kerberos`
   - Set `hbase.security.authorization` to `true`
   - Configure principal and keytab paths:
     - HBase Master: `/etc/security/keytabs/hbase.master.keytab`
     - HBase RegionServer: `/etc/security/keytabs/hbase.regionserver.keytab`

### Hive

1. **Create Principals**:
   ```bash
   # Login to kadmin
   kadmin.local
   
   # Create Hive Server2 principal
   addprinc -randkey hive/master2.example.com@EXAMPLE.COM
   
   # Create Hive Metastore principal
   addprinc -randkey hive/master2.example.com@EXAMPLE.COM
   
   # Create keytabs
   ktadd -k /etc/security/keytabs/hive.service.keytab hive/master2.example.com@EXAMPLE.COM HTTP/master2.example.com@EXAMPLE.COM
   
   # Exit kadmin
   quit
   ```

2. **Configure Hive for Kerberos**:
   In Ambari UI:
   - Go to Hive -> Configs -> Advanced
   - Set `hive.server2.authentication` to `KERBEROS`
   - Configure principal and keytab paths:
     - Hive Server2: `/etc/security/keytabs/hive.service.keytab`
     - Hive Metastore: `/etc/security/keytabs/hive.service.keytab`

### Kafka

1. **Create Principals**:
   ```bash
   # Login to kadmin
   kadmin.local
   
   # Create Kafka Broker principals
   addprinc -randkey kafka/master1.example.com@EXAMPLE.COM
   
   # Create keytabs
   ktadd -k /etc/security/keytabs/kafka.service.keytab kafka/master1.example.com@EXAMPLE.COM HTTP/master1.example.com@EXAMPLE.COM
   
   # Exit kadmin
   quit
   ```

2. **Configure Kafka for Kerberos**:
   In Ambari UI:
   - Go to Kafka -> Configs -> Advanced
   - Set `security.inter.broker.protocol` to `SASL_PLAINTEXT`
   - Set `sasl.kerberos.service.name` to `kafka`
   - Configure principal and keytab paths:
     - Kafka Broker: `/etc/security/keytabs/kafka.service.keytab`

### Spark2

1. **Create Principals**:
   ```bash
   # Login to kadmin
   kadmin.local
   
   # Create Spark History Server principal
   addprinc -randkey spark/master1.example.com@EXAMPLE.COM
   
   # Create keytabs
   ktadd -k /etc/security/keytabs/spark.service.keytab spark/master1.example.com@EXAMPLE.COM HTTP/master1.example.com@EXAMPLE.COM
   
   # Exit kadmin
   quit
   ```

2. **Configure Spark for Kerberos**:
   In Ambari UI:
   - Go to Spark2 -> Configs -> Advanced
   - Set `spark.history.kerberos.enabled` to `true`
   - Configure principal and keytab paths:
     - Spark History Server: `/etc/security/keytabs/spark.service.keytab`

## Post-Kerberos Configuration

### Create HDFS User Directories

After enabling Kerberos, you need to create and set permissions for HDFS user directories:

```bash
# Authenticate as HDFS superuser
kinit -kt /etc/security/keytabs/nn.service.keytab nn/master1.example.com@EXAMPLE.COM

# Create user directories
hdfs dfs -mkdir -p /user/alice
hdfs dfs -mkdir -p /user/bob
hdfs dfs -mkdir -p /user/hive
hdfs dfs -mkdir -p /user/spark

# Set ownership
hdfs dfs -chown alice:alice /user/alice
hdfs dfs -chown bob:bob /user/bob
hdfs dfs -chown hive:hive /user/hive
hdfs dfs -chown spark:spark /user/spark

# Set permissions
hdfs dfs -chmod 755 /user/alice
hdfs dfs -chmod 755 /user/bob
hdfs dfs -chmod 755 /user/hive
hdfs dfs -chmod 755 /user/spark

# Destroy ticket
kdestroy
```

### Create Test Users in KDC

```bash
# Login to kadmin
kadmin.local

# Create user principals
addprinc -pw hadoop alice@EXAMPLE.COM
addprinc -pw hadoop bob@EXAMPLE.COM

# Exit kadmin
quit
```

### Test Kerberized Environment

1. **Test HDFS Access**:
   ```bash
   # Authenticate as user
   kinit alice@EXAMPLE.COM
   
   # Try HDFS operations
   hdfs dfs -ls /
   hdfs dfs -mkdir /user/alice/test
   hdfs dfs -put /etc/hosts /user/alice/test/
   hdfs dfs -ls /user/alice/test/
   
   # Destroy ticket
   kdestroy
   
   # Try access without authentication (should fail)
   hdfs dfs -ls /
   ```

2. **Test Hive Access**:
   ```bash
   # Authenticate as user
   kinit alice@EXAMPLE.COM
   
   # Run Hive commands
   beeline -u "jdbc:hive2://master2.example.com:10000/default;principal=hive/master2.example.com@EXAMPLE.COM"
   
   # In Beeline:
   CREATE TABLE test (id INT, name STRING);
   INSERT INTO test VALUES (1, 'Test');
   SELECT * FROM test;
   DROP TABLE test;
   !quit
   
   # Destroy ticket
   kdestroy
   ```

## Advanced Configuration: Integrating with Active Directory

For enterprise environments, you may want to integrate with Microsoft Active Directory:

1. **Install Required Packages**:
   ```bash
   yum install -y krb5-workstation krb5-libs msktutil
   ```

2. **Configure Kerberos to Use AD**:
   ```bash
   # Edit /etc/krb5.conf
   vi /etc/krb5.conf
   
   # Example configuration:
   [libdefaults]
    default_realm = AD.EXAMPLE.COM
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
   
   [realms]
    AD.EXAMPLE.COM = {
     kdc = ad-server.example.com
     admin_server = ad-server.example.com
    }
   
   [domain_realm]
    .example.com = AD.EXAMPLE.COM
    example.com = AD.EXAMPLE.COM
   ```

3. **Join Cluster to AD Domain**:
   ```bash
   # Use msktutil to join domain and create service principals
   msktutil -c -b "OU=Hadoop,DC=example,DC=com" -s HTTP/master1.example.com -k /etc/security/keytabs/spnego.service.keytab --computer-name master1 --upn HTTP/master1.example.com@AD.EXAMPLE.COM --server ad-server.example.com --user-creds-only
   ```

4. **Configure Ambari to Use AD**:
   - In Ambari UI, go to Admin -> Kerberos
   - Select "Existing Active Directory" as the KDC type
   - Enter AD server details and admin credentials
   - Follow the Kerberos setup wizard

## Troubleshooting

### Common Kerberos Issues

1. **Connection refused to KDC**:
   - Verify KDC service is running: `systemctl status krb5kdc`
   - Check firewall settings: `firewall-cmd --list-all`
   - Ensure port 88 is open: `netstat -tulpn | grep 88`

2. **Clock Skew Too Great**:
   - Synchronize time across all nodes:
     ```bash
     systemctl restart ntpd
     ntpq -p
     ```

3. **Unable to obtain Kerberos ticket**:
   - Verify KDC configuration: `klist -e`
   - Check principal exists: `kadmin.local -q "listprincs" | grep <principal_name>`
   - Ensure password is correct

4. **Service fails to start after Kerberization**:
   - Check service logs:
     ```bash
     tail -f /var/log/hadoop/hdfs/hadoop-hdfs-namenode.log
     ```
   - Verify keytab permissions:
     ```bash
     ls -la /etc/security/keytabs/
     ```
   - Test keytab with kinit:
     ```bash
     kinit -kt /etc/security/keytabs/nn.service.keytab nn/master1.example.com@EXAMPLE.COM
     ```

5. **Invalid keytab error**:
   - Re-create the keytab:
     ```bash
     kadmin.local -q "ktadd -k /etc/security/keytabs/nn.service.keytab nn/master1.example.com@EXAMPLE.COM"
     ```
   - Check keytab content:
     ```bash
     klist -kt /etc/security/keytabs/nn.service.keytab
     ```

6. **Permission denied in HDFS**:
   - Check user authentication:
     ```bash
     klist
     ```
   - Verify file permissions:
     ```bash
     hdfs dfs -ls -d /user/alice
     ```
   - Set proper ownership:
     ```bash
     kinit -kt /etc/security/keytabs/nn.service.keytab nn/master1.example.com@EXAMPLE.COM
     hdfs dfs -chown alice:alice /user/alice
     ```

### Kerberos Debugging

1. **Enable Detailed Logging**:
   ```bash
   # Edit /etc/krb5.conf
   vi /etc/krb5.conf
   
   # Add debug information
   [logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log
   ```

2. **Test Kerberos Configuration**:
   ```bash
   KRB5_TRACE=/dev/stdout kinit alice@EXAMPLE.COM
   ```

3. **Verify Service Principals**:
   ```bash
   klist -kt /etc/security/keytabs/nn.service.keytab
   ```

4. **Check Authentication with KVNO**:
   ```bash
   kvno nn/master1.example.com@EXAMPLE.COM
   ```

## References

1. [Apache Ambari Security Guide](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.5.0/security/index.html)
2. [Apache Hadoop Security Guide](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SecureMode.html)
3. [Kerberos Documentation](https://web.mit.edu/kerberos/krb5-latest/doc/)
4. [HDP Security Administration](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.5/security-reference/index.html)
5. [Kerberizing Hadoop Services](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.5/authentication-with-kerberos/content/kerberizing_hadoop_services.html) 