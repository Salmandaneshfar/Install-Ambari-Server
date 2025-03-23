# Complete Guide to Installing and Kerberizing Hadoop Ecosystem Services with Apache Ambari on RHEL 8.10

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [RHEL 8.10 Specific Preparations](#rhel-810-specific-preparations)
- [Ambari Server Installation](#ambari-server-installation)
- [Hadoop Cluster Setup](#hadoop-cluster-setup)
- [Kerberos Implementation](#kerberos-implementation)
- [Kerberizing Hadoop Services](#kerberizing-hadoop-services)
- [Post-Kerberos Configuration](#post-kerberos-configuration)
- [Troubleshooting RHEL 8.10 Specific Issues](#troubleshooting-rhel-810-specific-issues)
- [References](#references)

## Introduction

This guide provides comprehensive instructions for installing, configuring, and securing a Hadoop cluster using Apache Ambari with Kerberos authentication on Red Hat Enterprise Linux 8.10. It accounts for the specific changes and requirements in RHEL 8.10 compared to earlier versions.

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
- Operating System: Red Hat Enterprise Linux 8.10 (64-bit)
- Java: OpenJDK 11 (RHEL 8.10 default) or OpenJDK 1.8
- PostgreSQL 10+ (included in RHEL 8.10 AppStream)
- Browser: Chrome, Firefox (latest versions)

### Network Requirements
- All nodes should have static IP addresses
- All nodes must be able to communicate with each other
- Forward and reverse DNS must be properly configured for all hosts
- Chronyd must be configured to maintain clock synchronization

## RHEL 8.10 Specific Preparations

### Register RHEL System

Before beginning, ensure your RHEL 8.10 systems are registered and subscribed to the appropriate repositories:

```bash
# Register system with Red Hat Subscription Management
subscription-manager register --username=your_username --password=your_password

# Attach a subscription
subscription-manager attach --auto

# Enable required repositories
subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
subscription-manager repos --enable=codeready-builder-for-rhel-8-x86_64-rpms
```

### Configure Module Streams

RHEL 8.10 uses the concept of module streams for package management. Configure the appropriate module streams for your environment:

```bash
# Check available module streams
dnf module list

# Enable PostgreSQL 12 stream (recommended for Ambari)
dnf module enable -y postgresql:12

# Enable Java 11 (default in RHEL 8.10) or Java 8
dnf module enable -y java:11
# Or for Java 8
# dnf module enable -y java:1.8
```

### Install Required Base Packages

```bash
# Update system packages
dnf upgrade -y

# Install base required packages
dnf install -y wget curl tar zip unzip python3 python3-pip bind-utils net-tools
```

### Create Symbolic Links for Python

RHEL 8.10 uses Python 3 by default. Ambari may require Python 2, so create appropriate symbolic links:

```bash
# Create alternatives group for python
alternatives --set python /usr/bin/python3

# Create symlink if python command is not available
ln -sf /usr/bin/python3 /usr/bin/python
```

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

2. **Configure SELinux**:
   ```bash
   # Set SELinux to permissive (recommended for initial setup)
   sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
   
   # Apply change immediately
   setenforce 0
   
   # Verify status
   getenforce
   ```

3. **Configure Firewall**:
   ```bash
   # Install firewalld if not already installed
   dnf install -y firewalld
   
   # Start and enable firewalld
   systemctl start firewalld
   systemctl enable firewalld
   
   # Add Hadoop services to firewalld
   firewall-cmd --permanent --add-service=ssh
   
   # Common Hadoop ports
   firewall-cmd --permanent --add-port=8080/tcp  # Ambari Web UI
   firewall-cmd --permanent --add-port=8443/tcp  # Ambari Web UI (HTTPS)
   firewall-cmd --permanent --add-port=8440/tcp  # Ambari API
   firewall-cmd --permanent --add-port=8441/tcp  # Ambari API (HTTPS)
   
   # HDFS Ports
   firewall-cmd --permanent --add-port=8020/tcp  # NameNode
   firewall-cmd --permanent --add-port=9870/tcp  # NameNode Web UI
   firewall-cmd --permanent --add-port=9864/tcp  # DataNode Web UI
   firewall-cmd --permanent --add-port=9000/tcp  # HDFS
   
   # YARN Ports
   firewall-cmd --permanent --add-port=8088/tcp  # ResourceManager Web UI
   firewall-cmd --permanent --add-port=8032/tcp  # ResourceManager
   firewall-cmd --permanent --add-port=8030/tcp  # Scheduler
   firewall-cmd --permanent --add-port=8031/tcp  # Resource Tracker
   firewall-cmd --permanent --add-port=8042/tcp  # NodeManager Web UI
   
   # HBase Ports
   firewall-cmd --permanent --add-port=16000/tcp # HBase Master
   firewall-cmd --permanent --add-port=16010/tcp # HBase Master Web UI
   firewall-cmd --permanent --add-port=16020/tcp # HBase RegionServer
   firewall-cmd --permanent --add-port=16030/tcp # HBase RegionServer Web UI
   
   # Hive Ports
   firewall-cmd --permanent --add-port=10000/tcp # HiveServer2
   firewall-cmd --permanent --add-port=9083/tcp  # Hive Metastore
   
   # Kerberos Ports
   firewall-cmd --permanent --add-port=88/tcp    # Kerberos
   firewall-cmd --permanent --add-port=88/udp    # Kerberos
   firewall-cmd --permanent --add-port=749/tcp   # Kerberos admin
   firewall-cmd --permanent --add-port=754/tcp   # Kerberos kpasswd
   
   # Apply changes
   firewall-cmd --reload
   ```

4. **Set up password-less SSH from Ambari server to all nodes**:
   ```bash
   # On Ambari server, generate SSH key
   ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
   
   # Copy the key to all nodes, including Ambari server itself
   for host in ambari-server master1 master2 worker1 worker2 worker3; do
     ssh-copy-id -i ~/.ssh/id_rsa.pub root@$host
   done
   
   # Test SSH connectivity
   for host in ambari-server master1 master2 worker1 worker2 worker3; do
     ssh root@$host hostname
   done
   ```

5. **Install chronyd and synchronize time on all nodes**:
   ```bash
   # Install chrony (modern replacement for NTP in RHEL 8)
   dnf install -y chrony
   
   # Start and enable chrony service
   systemctl start chronyd
   systemctl enable chronyd
   
   # Verify time synchronization
   chronyc sources
   chronyc tracking
   ```

6. **Install Java on all nodes**:
   ```bash
   # Install OpenJDK (Java 11 is RHEL 8.10 default)
   dnf install -y java-11-openjdk java-11-openjdk-devel
   
   # For Java 8 (if required)
   # dnf install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel
   
   # Set JAVA_HOME in /etc/profile.d
   echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk' > /etc/profile.d/java.sh
   # Or for Java 8
   # echo 'export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk' > /etc/profile.d/java.sh
   
   echo 'export PATH=$JAVA_HOME/bin:$PATH' >> /etc/profile.d/java.sh
   chmod +x /etc/profile.d/java.sh
   source /etc/profile.d/java.sh
   
   # Verify Java installation
   java -version
   ```

7. **Configure max open file descriptors**:
   ```bash
   # Add to /etc/security/limits.conf
   cat >> /etc/security/limits.conf << 'EOF'
   * soft nofile 10000
   * hard nofile 10000
   * soft nproc 65536
   * hard nproc 65536
   EOF
   
   # Verify settings (after relogin)
   ulimit -Sn
   ulimit -Hn
   ```

## Ambari Server Installation

### Set Up Ambari Repository

For RHEL 8.10, you'll need to use the appropriate Ambari repository:

```bash
# Download Ambari repository file
wget -O /etc/yum.repos.d/ambari.repo https://archive.apache.org/dist/ambari/ambari-2.7.5/centos7/ambari-2.7.5.0-centos7.repo

# Edit the repository file to point to the correct path
sed -i 's/centos7/centos8/g' /etc/yum.repos.d/ambari.repo

# Add HDP repository
wget -O /etc/yum.repos.d/hdp.repo https://archive.apache.org/dist/ambari/ambari-2.7.5/centos7/hdp-3.1.5.0.repo

# Modify the repo file for RHEL 8
sed -i 's/centos7/centos8/g' /etc/yum.repos.d/hdp.repo

# Clean and update repository cache
dnf clean all
dnf repolist
```

### Install PostgreSQL for Ambari

RHEL 8.10 comes with PostgreSQL 12 in the AppStream repository:

```bash
# Install PostgreSQL 12
dnf install -y postgresql-server postgresql-contrib

# Initialize the database
postgresql-setup --initdb

# Configure PostgreSQL for remote connections
sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/g" /var/lib/pgsql/data/postgresql.conf

# Configure client authentication
cat > /var/lib/pgsql/data/pg_hba.conf << 'EOF'
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
host    all             all             0.0.0.0/0               md5
EOF

# Start and enable PostgreSQL
systemctl start postgresql
systemctl enable postgresql

# Create Ambari database and user
sudo -u postgres psql << EOF
CREATE DATABASE ambari;
CREATE USER ambari WITH PASSWORD 'bigdata';
GRANT ALL PRIVILEGES ON DATABASE ambari TO ambari;

CREATE DATABASE hive;
CREATE USER hive WITH PASSWORD 'hive';
GRANT ALL PRIVILEGES ON DATABASE hive TO hive;

CREATE DATABASE oozie;
CREATE USER oozie WITH PASSWORD 'oozie';
GRANT ALL PRIVILEGES ON DATABASE oozie TO oozie;

CREATE DATABASE ranger;
CREATE USER ranger WITH PASSWORD 'ranger';
GRANT ALL PRIVILEGES ON DATABASE ranger TO ranger;

CREATE DATABASE rangerkms;
CREATE USER rangerkms WITH PASSWORD 'rangerkms';
GRANT ALL PRIVILEGES ON DATABASE rangerkms TO rangerkms;
\q
EOF
```

### Install Ambari Server

```bash
# Install Ambari Server package
dnf install -y ambari-server

# Download PostgreSQL JDBC driver
wget https://jdbc.postgresql.org/download/postgresql-42.2.18.jar -O /tmp/postgresql-42.2.18.jar
mv /tmp/postgresql-42.2.18.jar /usr/share/java/postgresql-jdbc.jar
ambari-server setup --jdbc-db=postgres --jdbc-driver=/usr/share/java/postgresql-jdbc.jar

# Set up Ambari Server
ambari-server setup

# During setup:
# 1. Accept the default settings when prompted
# 2. Choose Custom JDK and provide the path to your JAVA_HOME
# 3. When asked about database, choose option 3 (PostgreSQL)
# 4. Provide database details:
#    - hostname: localhost
#    - port: 5432
#    - database name: ambari
#    - username: ambari
#    - password: bigdata

# Start Ambari Server
ambari-server start

# Check Ambari Server status
ambari-server status
```

## Hadoop Cluster Setup

### Access Ambari Web UI

After starting Ambari Server, access the web UI:
- Open a browser and navigate to: `http://ambari-server.example.com:8080`
- Default credentials: admin/admin
- Change the password after first login

The UI experience and cluster setup process is similar to the original guide, but some specific configurations for RHEL 8.10 include:

### Install Cluster Using Ambari Web UI

1. **Launch Install Wizard**:
   - Click "Launch Install Wizard" on the Ambari welcome page

2. **Name Your Cluster**:
   - Provide a name for your cluster (e.g., "HadoopCluster")

3. **Select Stack**:
   - Choose HDP 3.1.5.0
   - Select "Use Local Repository" if you set up local repositories

4. **Install Options**:
   - Enter the FQDN of all hosts (or upload a hosts file)
   - Provide SSH private key or password for authentication
   - Make sure to select "Perform manual registration on hosts and do not use SSH" if you've already prepared the nodes
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
     - Database settings for Hive, Oozie, Ranger (use PostgreSQL 12)
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

RHEL 8.10 uses updated Kerberos packages:

1. **Install Kerberos Server Packages**:
   ```bash
   # On the KDC server (can be the Ambari server or a dedicated node)
   dnf install -y krb5-server krb5-libs krb5-workstation
   
   # On all other nodes
   dnf install -y krb5-workstation krb5-libs
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

The rest of the Kerberization process, including manual principal and keytab creation (if needed), follows the same pattern as the original guide.

## Post-Kerberos Configuration

### Test Kerberized Environment on RHEL 8.10

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

2. **Verify SELinux Status**:
   ```bash
   # Check SELinux status
   getenforce
   
   # If in enforcing mode and experiencing issues, temporarily set to permissive
   setenforce 0
   
   # For persistent change in production, configure SELinux policies instead of disabling it
   ```

## Troubleshooting RHEL 8.10 Specific Issues

### Common Issues in RHEL 8.10

1. **Python Version Mismatch**:
   - Issue: Ambari requires Python 2.7, but RHEL 8.10 uses Python 3 by default
   - Solution:
     ```bash
     # Install Python 2.7 if needed
     dnf install -y python2
     
     # Create appropriate alternatives
     alternatives --set python /usr/bin/python2
     
     # Restart Ambari components after making the change
     ambari-server restart
     ```

2. **PostgreSQL 12 Compatibility**:
   - Issue: Connection issues with PostgreSQL 12
   - Solution: 
     ```bash
     # Check PostgreSQL logs
     less /var/lib/pgsql/data/log/postgresql-*.log
     
     # Ensure correct JDBC driver is used
     ambari-server setup --jdbc-db=postgres --jdbc-driver=/usr/share/java/postgresql-jdbc.jar
     ```

3. **Firewall Blocking Required Ports**:
   - Issue: Services unable to communicate due to firewall rules
   - Solution:
     ```bash
     # Check firewall status
     firewall-cmd --list-all
     
     # Add any missing ports
     firewall-cmd --permanent --add-port=<port>/<protocol>
     firewall-cmd --reload
     ```

4. **SELinux Issues**:
   - Issue: SELinux blocking service operations even in permissive mode
   - Solution:
     ```bash
     # Install SELinux troubleshooting tools
     dnf install -y setroubleshoot setools
     
     # Analyze SELinux issues
     sealert -a /var/log/audit/audit.log
     
     # Create custom policy for Hadoop services
     audit2allow -a -M hadoop-services
     semodule -i hadoop-services.pp
     ```

5. **Java Version Compatibility**:
   - Issue: Services requiring a specific Java version
   - Solution:
     ```bash
     # Install multiple Java versions
     dnf install -y java-1.8.0-openjdk java-11-openjdk
     
     # Switch between versions as needed
     alternatives --config java
     
     # Update JAVA_HOME in Ambari configuration
     ambari-server setup
     ```

6. **Chronyd Synchronization Issues**:
   - Issue: Clock skew affecting Kerberos authentication
   - Solution:
     ```bash
     # Check chrony status
     chronyc tracking
     
     # Force time synchronization
     chronyc makestep
     
     # Check time synchronization sources
     chronyc sources
     ```

## References

1. [RHEL 8.10 Official Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/8.10_release_notes/index)
2. [Apache Ambari Security Guide](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.5.0/security/index.html)
3. [Apache Hadoop Security Guide](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SecureMode.html)
4. [Kerberos Documentation](https://web.mit.edu/kerberos/krb5-latest/doc/)
5. [RHEL 8 SELinux User's and Administrator's Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/index)
6. [RHEL 8 Managing Networking](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/index)
7. [HDP Security Administration](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.5/security-reference/index.html) 