Install the Ambari Server 
===


This is my documentation install ambari server.

## Software Requirements

- Check OS of your machines and see if it's a 64-bit machine
- Check if we have postgresql package exist


## Setup Yum and Check for packages installed 
---

```gherkin=
==============================
-- Setup Local YUM
==============================
su - root
mkdir /mnt/cdrom

# Linux 7.5
mount /dev/sr0 /mnt/cdrom  

# Linux 7.3
mount /dev/cdrom /mnt/cdrom
mount /dev/sdc1 /mnt/cdrom/

# mount iso
mount -o loop RHEL7.1.iso /mnt/cdrom

vi /etc/yum.repos.d/rhel7dvd.repo

[InstallMedia]
name=Red Hat Enterprise Linux 7.5
mediaid=1521745267.626363
metadata_expire=-1
gpgcheck=1
cost=500
enabled=1
baseurl=file:///mnt/cdrom
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

yum clean all
yum --noplugins list
yum --noplugins update

$ yum --help
$ rpm --help
$ curl --help
$ tar --help
$ wget --help
$ python -V

If not please install above packages using (Yum install <<package name>>)
$ yum install yum rpm curl tar wget python
```


## JDK Requirements

```gherkin=
OpenJDK 1.8 for PPC is the only JDK supported.
You must install the following OpenJDK 1.8 for PPC packages as a prerequisite on all machines in the cluster:
java-1.8.0-openjdk
java-1.8.0-openjdk-devel
java-1.8.0-openjdk-headless

$ yum install java-1.8.0*

Set your JAVA_HOME:
vi .bash_profile
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export PATH=$JAVA_HOME/bin:$PATH


[root@localhost ~]# java -version
openjdk version "1.8.0_161"
OpenJDK Runtime Environment (build 1.8.0_161-b14)
OpenJDK 64-Bit Server VM (build 25.161-b14, mixed mode)
[root@localhost ~]#
```

## Check the Maximum Open File Descriptors
```
The recommended maximum number of open file descriptors is 10000, or more. 
To check the current value set for the maximum number of open file descriptors, 
execute the following shell commands on each host:

$ ulimit -Sn
$ ulimit -Hn

If the output is not greater than 10000, run the following command to set it to a suitable default:

$ ulimit -n 10000
```

## Configuring iptables
```
	For Ambari to communicate during setup with the hosts it deploys to and manages, 
	certain ports must be open and available. The easiest way to do this is to temporarily disable iptables, as follows:

	Disable fire wall on all the nodes using below commands

	$ systemctl stop  firewalld
	$ service firewalld stop
	$ systemctl disable firewalld

	Note: We should restart firewallid once we are done with entire cluster setup
```

## Enable NTP in all the machines (Master + all slave nodes)
```
$ yum install ntp

Enable the service:
$ systemctl enable ntpd

Start NTPD:
$ systemctl start ntpd

```

## Before doing ambari-server setup first disable SELinux status in all the nodes

```gherkin=
$ vi /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

## Make an entry for all the hostnames in all machines like below
```
$ vi /etc/hosts
172.16.9.207   hdpmaster.yogya.com    hdpmaster
172.16.9.206   hdpmaster1.yogya.com   hdpmaster1
172.16.9.211   datanode1.yogya.com    datanode1
172.16.9.212   datanode2.yogya.com    datanode2
```

## Please Reboot Server

## Make sure to repeat the Steps  in all other nodes

## From Master Host 
```
# Install Just in Master host
$ uname -m
$ yum search postgres
$ yum install postgresql-server.x86_64 

# Check Httpd Service in all the machines
$ yum search httpd
$ yum install httpd
$ service httpd start

$ service network restart 
```

## Create SSH key in primary 
```
Create SSH key in primary node and then copy that key to all other slave machines to avoid giving the password 
while connecting from master node. This is required for setting up ambari agents 

From master node enter below command:

$ ssh-keygen

After entering above command press enter and for every prompt just do ENTER then it will automatically create SSH keys

Create authorized_keys file like below

$ cd /root/.ssh/
$ cat id_rsa.pub >> authorized_keys


Check whether we have /root/.ssh/ folder in all other slave nodes if not create .ssh using ssh-keygen 
Then copy the above created authorized keys file to all other slave nodes like below 

scp /root/.ssh/authorized_keys  hdpslave1.yogya.com://root/.ssh/
scp /root/.ssh/authorized_keys  hdpslave2.yogya.com://root/.ssh/

```

## Test http
```
Enter master node IP address in browser: http://172.16.9.211/  . You will see Testing 123 page
```

## Download Hadoop Stack:
```
http://docs.hortonworks.com/HDPDocuments/Ambari-2.4.1.0/bk_ambari-installation/content/hdp_25_repositories.html

-- Download Ambari
nohup wget -nv http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.5.0.0/HDP-2.5.0.0-centos7-rpm.tar.gz &
nohup wget -nv http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.21/repos/centos7/HDP-UTILS-1.1.0.21-centos7.tar.gz &
nohup wget -nv http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.4.1.0/ambari-2.4.1.0-centos7.tar.gz & 

Extract Amabari Tar file

$ cd /root/source/hortownwork
$ cp HDP* ambari* /var/www/html/
$ cd /var/www/html
$ tar xvf ambari-2.4.1.0-centos7.tar.gz
$ tar xvf HDP-2.5.0.0-centos7-rpm.tar.gz
$ mkdir HDP-UTILS-1.1.0.21
$ mv HDP-UTILS-1.1.0.21-centos7.tar.gz HDP-UTILS-1.1.0.21
$ cd HDP-UTILS-1.1.0.21
$ tar xvf HDP-UTILS-1.1.0.21-centos7.tar.gz
```

## Create local repository
```
Create amabari repo file
$ cd /etc/yum.repos.d

$ vi /etc/yum.repos.d/ambari.repo

[Ambari]
name=Ambari
enabled=1
baseurl=http://172.16.9.211/AMBARI-2.4.1.0/centos7/2.4.1.0-22/
gpgcheck=0

vi /etc/yum.repos.d/hdp.repo

[HDP-2.5.0.0]
name=HDP Version - HDP-2.5.0.0
baseurl=http://172.16.9.211/HDP/centos7/
gpgcheck=0
gpgkey=http://172.16.9.211/HDP/centos7/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1


[HDP-UTILS-1.1.0.21]
name=HDP-UTILS Version - HDP-UTILS-1.1.0.21
baseurl=http://172.16.9.211/HDP-UTILS-1.1.0.21/
gpgcheck=0
gpgkey=http://172.16.9.211/HDP-UTILS-1.1.0.21/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1

Check  ambari is showing in repo list

$ yum repolist

Ambari                             Ambari                                                   12
HDP-2.5.0.0                        HDP Version - HDP-2.5.0.0                               200
HDP-UTILS-1.1.0.21                 HDP-UTILS Version - HDP-UTILS-1.1.0.21                   64
repol

```

## Installing ambari server 
```
Install ambari server
$ yum install ambari-server

Download Postgres Driver
$ wget https://jdbc.postgresql.org/download/postgresql-42.2.5.jar
$ cp postgresql-42.2.5.jar /root/source/hortownwork/

Initial Driver postgres JDBC
$ ambari-server setup --jdbc-db=postgres --jdbc-driver=/root/source/hortownwork/postgresql-42.2.5.jar
$ echo $JAVA_HOME
$ ambari-server setup

Using python  /usr/bin/python
Setup ambari-server
Checking SELinux...
SELinux status is 'enabled'
SELinux mode is 'enforcing'
Temporarily disabling SELinux
WARNING: SELinux is set to 'permissive' mode and temporarily disabled.
OK to continue [y/n] (y)? y
Customize user account for ambari-server daemon [y/n] (n)?
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Oracle JDK 1.7 + Java Cryptography Extension (JCE) Policy Files 7
[3] Custom JDK
==============================================================================
Enter choice (1): /usr/lib/jvm/java-1.8.0-openjdk/
Invalid number.
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Oracle JDK 1.7 + Java Cryptography Extension (JCE) Policy Files 7
[3] Custom JDK
==============================================================================
Enter choice (1): 3
WARNING: JDK must be installed on all hosts and JAVA_HOME must be valid on all hosts.
WARNING: JCE Policy files are required for configuring Kerberos security. If you plan to use Kerberos,please make sure JCE Unlimited Strength Jurisdiction Policy Files are valid on all hosts.
Path to JAVA_HOME: /usr/lib/jvm/java-1.8.0-openjdk/
Validating JDK on Ambari Server...done.
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)?
Configuring database...
Default properties detected. Using built-in database.
Configuring ambari database...
Checking PostgreSQL...
Running initdb: This may take up to a minute.
Initializing database ... OK


About to start PostgreSQL
Configuring local database...
Connecting to local database...done.
Configuring PostgreSQL...
Restarting PostgreSQL
Extracting system views...
ambari-admin-2.4.1.0.22.jar
...........
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.
[root@localhost ~]#
```

## Start Ambari Server
```
$ ambari-server start

Using python  /usr/bin/python
Starting ambari-server
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Ambari database consistency check started...
No errors were found.
Ambari database consistency check finished
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start....................
Ambari Server 'start' completed successfully.
```

## Create Database Hive and oozie user
```
$ su - postgres
$ psql

drop database hivemeta;
create database hivemeta;
create user hive with password 'hive';
grant all privileges on database hivemeta to hive;

drop database ooziemeta;
create database ooziemeta;
create user oozie with password 'oozie';
grant all privileges on database ooziemeta to oozie;

drop database rangermeta;
create database rangermeta;
create user ranger with password 'rangeradmin';
grant all privileges on database rangermeta to ranger;

drop database rangerkmsmeta;
create database rangerkmsmeta;
create user rangerkms with password 'rangerkmsadmin';
grant all privileges on database rangerkmsmeta to rangerkms;

Then enter \q
postgres=# exit

After that edit hba_conf file to access Metadata database

$ cd /var/lib/pgsql/data
$ ls -l
$ vi pg_hba.conf

Add below entries at the end of file
 host  all  all   0.0.0.0/0  trust

$ su - root
$ service postgresql restart
```

## Stop Ambari server
```
$ ambari-server stop
$ service httpd stop
```

## Setup  Cluster From Ambari
```
# Mount all cdrom after restart 
$ mount /dev/sr0 /mnt/ 
$ service httpd start
$ ambari-server setup --jdbc-db=postgres --jdbc-driver=/root/source/hortownwork/postgresql-42.2.5.jar
$ ambari-server start


Open Ambari UI to install required hadoop services 
Go to http://172.16.7.115:8080/ and enter username admin and password as admin

Setting up Hadoop cluster

1. Landing page
Click on “Launch Install Wizard” to start cluster setup

2. Cluster Name
Give you cluster a good name.
Note: This is just a simple name for cluster, it is not that significant, 
so don’t worry about it and choose any name for it.
ie : YOGYA_DEV_CLUSTER

3. Stack selection
stack selection in Hadoop Ambari
Use Local Repository

Choose 
    redhat7
                            Baseurl
    HDP-2.5					http://172.16.9.211/HDP/centos7/

    HDP-UTILS-1.1.0.21		http://172.16.9.211/HDP-UTILS-1.1.0.21/


This page will list stacks available to install. Each stack is pre-packaged with Hadoop ecosystem component. 
These stacks are from Hortonworks. (We can install plain Hadoop too. That we will cover in later posts).

4. Hosts Entry and SSH key entry
Prior moving further this step we should have password less SSH setup for all the participating nodes.

Target Host
    hdpmaster.yogya.com
    hdpslave1.yogya.com
    hdpslave2.yogya.com


Private Key
    On Master
    $ cd .ssh
    $ cat id_rsa
      copy paste 

    -----BEGIN RSA PRIVATE KEY-----
    MIIEogIBAAKCAQEAxBtkMQBAlKYeJpbR8OmFMip6oNkTZqFBc8b6csQmw9guGYAt
    Qw3R6klIa2Slr6WAOVUbxR044EnuEjIi3dR6+c97xMQV5NUNdVu6eltJTDNrf8/1
    8VEx/1ECagqCANWdLM5P+XCBh/8iaKpekBvmgEBmEBPeSrA/COyStFNluGlh5AxE
    +fu3aubO+2NBMAcv1xgmWYKjX7GSOZ3J661CbA1p0FTAIBne/mpXxGujTLcCSiFw
    D6QVCr6TdXfni18e7StltUfrrKqXYiSHctEV3oZjzKfJJnSlPLlKaNWdKuURDj2q
    dj3r9ZFUYMqUJHhN0WwGoyT97cOrAsofnIgMxwIDAQABAoIBAF7abNyypYoAy2aY
    3pTrLoy0NTolpRen+bOZU7w9Gg7yOmIFOF9NiPIMXiXruaQ6pcmVW+g8mS0LNUbB
    z1GCm7TG4bOrsHdNgcP5CTpzewGLgXyBxxDg2BEJSuSljnn+2JY6eD5LZ6uzAR6l
    ATYs+JGiFHvEUGnJ08NqQg9Mo0/NeE63Qiw8lMK0UPSf/WJM7VaPEa+FvuOPiulM
    1r9nfH5gJxTq2ZxtQvWhOlceDTRnDgDApvFvqa5jlC+MVVS3BYDOaDjJ7Qoj8Ttw
    +DOCQpGpnUkN1nsogGm48lo5mdHOOSxafv8qOAeDEO7P/D45l4sGhNE/42QfWYCK
    GwfZdekCgYEA49HVHDbF86oO5EBJMwjtNbq2JlSjaabgzfqBV/QOI3KJ+Vvpa8mC
    3fL74h426m2/3S7u/etb+uRaEnrPO5NOAe9uDap2O57Bzz5FM1GmfT/I6AeitfZK
    9SsqZjk3pHkc700eNeKUFa2bXu+MfUiO++2R//Qo+t9l1Rsav6pFiB0CgYEA3F1X
    cqVCHOKqOdG7J5DBve7Uwe8vPSK2TlCCPglvbNgfRwhW1BDCDc445FXrbA6J8EAn
    y872VxMa5yWq0ijscEOKeSLPxFtVbsz7QAhelJ1yUBTRQ2jlicVC+ge0vs17Ytme
    tJ0mpDRHZKhOHI8YGxbyooTrSh3PVINHnkboezMCgYBPyEEwk0H5lhmG97hqxfqE
    cXGutL9RlZ5upAa97XsyEL+e8wAovjY1Ug3B30DuEic552DMzaq1j1i31ghS3cBY
    zyekY4jqUiufTzhew35hqH/MOjhSLwGLIGXFzM3erIdkioZE6qdffB/IPG3fxhRE
    x6r0juX3DVsVKVvRuWiGRQKBgHdSt8plf+IiPmimj4ACG1acX2pP+LVS+YJ4h73N
    4B8A/Ba7hkC4fkt5ckb520ucp8aHBsWOYMePmc62D8RS0oyLlgy38+bwSdeAeUAY
    CBbTFpYUX6fvwfMS9Ixs5cs3eutwyUYlnknl4Q65L+q49SWeDG5CKSHt+Flb6Mjk
    vngPAoGAHFAOoamRBCQUD+4rfwMi8rmal87nVZn+OanQkCA/uUOslZzFgVxmoTPu
    kpAadVdD/xbcP2MMPJADpsA+nPrk/SsKg/U2N05Mf/86xC5FrPdh0jDP3MoNNg8Z
    l8CYOKRmk4uXpJGYr22bxOuogcIWy+BOFZtbCrbrSOc4fz1UP98=
    -----END RSA PRIVATE KEY-----


=================================	  
ERROR WHEN Confirm Hosts
=================================

Note : Error
ERROR 2019-01-03 17:06:31,815 NetUtil.py:88 - [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:579)
ERROR 2019-01-03 17:06:31,816 NetUtil.py:89 - SSLError: Failed to connect. Please check openssl library versions

Edit in All Node
  $ vi /etc/ambari-agent/conf/ambari-agent.ini
      [security]
      force_https_protocol=PROTOCOL_TLSv1_2

  $ vi /etc/python/cert-verification.cfg
      [https]
      verify=disable
```

## Assign Slaves and Clients

In the next screen, we will designate where data nodes and node managers will run. We will run data nodes and node managers on all nodes, we will also install clients on all nodes. This would mean we can run HDFS, YARN, pig and Hive commands from any nodes since we have installed clients on all nodes. NFSGateway allows HDFS to be mounted as a drive so you can browse HDFS as if you are browsing a local file system. We will install this on all nodes as well.

Note : CheckList All 


## Customize Services
```
- Hive 
Database : hivemeta
username : hive
password : hive
jdbc url : jdbc:postgresql://hadoopmastar.yogya.local:5432/hivemeta

- Ozie
Database : ooziemeta
username : oozie
password : oozie
jdbc url : jdbc:postgresql://hadoopmastar.yogya.local:5432/ooziemeta

- Accumuio
username root and password root
password admin

- Ambari Metric
user.    : admin
password : admin

- Knok
Knox Gateway host : hdpmaster.yogya.com
pasword : admin

- SmartSense
password : admin

- Ranger
rangerhost : hdpmaster.yogya.com
Database : rangermeta
username : ranger
password : rangeradmin
jdbc url : jdbc:postgresql://hadoopmastar.yogya.local:5432/rangermeta

- Ranger
rangerhost : hdpmaster.yogya.com
Database : rangerkmsmeta
username : rangerkms
password : rangerkmsadmin
jdbc url : jdbc:postgresql://hadoopmastar.yogya.local:5432/rangerkmsmeta
KMS Master Secret Password  : admin


==================================================
Review
Please review the configuration before installation
==================================================

Admin Name : admin

Cluster Name : YOGYA_DEV

Total Hosts : 3 (3 new)

Repositories:

redhat7 (HDP-2.5): 
http://172.16.7.115/HDP/centos7/
redhat7 (HDP-UTILS-1.1.0.21): 
http://172.16.7.115/HDP-UTILS-1.1.0.21/
Services:

HDFS
DataNode : 3 hosts
NameNode : hdpmaster.yogya.com
NFSGateway : 0 host
SNameNode : hdpslave1.yogya.com
YARN + MapReduce2
App Timeline Server : hdpmaster.yogya.com
NodeManager : 3 hosts
ResourceManager : hdpmaster.yogya.com
Tez
Clients : 3 hosts
Hive
Metastore : hdpmaster.yogya.com
HiveServer2 : hdpmaster.yogya.com
WebHCat Server : hdpmaster.yogya.com
Database : Existing PostgreSQL Database
HBase
Master : hdpmaster.yogya.com
RegionServer : 3 hosts
Phoenix Query Server : 0 host
Pig
Clients : 3 hosts
Sqoop
Clients : 3 hosts
Oozie
Server : hdpmaster.yogya.com
Database : Existing PostgreSQL Database
ZooKeeper
Server : 3 hosts
Falcon
Server : hdpmaster.yogya.com
Storm
DRPC Server : hdpmaster.yogya.com
Nimbus : hdpmaster.yogya.com
UI Server : hdpmaster.yogya.com
Supervisor : 3 hosts
Flume
Flume : 3 hosts
Accumulo
GC : hdpmaster.yogya.com
Master : hdpmaster.yogya.com
Monitor : hdpmaster.yogya.com
Tracer : hdpmaster.yogya.com
TServer : 3 hosts
Ambari Infra
Infra Solr Instance : hdpmaster.yogya.com
Ambari Metrics
Metrics Collector : hdpmaster.yogya.com
Grafana : hdpmaster.yogya.com
Atlas
Metadata Server : hdpmaster.yogya.com
Kafka
Broker : hdpmaster.yogya.com
Knox
Gateway : hdpmaster.yogya.com
SmartSense
Activity Analyzer : hdpmaster.yogya.com
Activity Explorer : hdpmaster.yogya.com
HST Server : hdpmaster.yogya.com
Spark
Livy Server : 3 hosts
History Server : hdpmaster.yogya.com
Thrift Server : 3 hosts
Spark2
History Server : hdpmaster.yogya.com
Thrift Server : 3 hosts
Zeppelin Notebook
Notebook : hdpmaster.yogya.com
Mahout
Clients : 3 hosts
Slider
Clients : 3 hosts
```

## Start Service Manualy
```
- Start Ozie Server
- Start Atlas	
- Start other Service
- SmartSensse Still not working -- On progress
```

## Enabling Spark2
```
In order to use Spark2, you must first install Spark2 on your cluster. 
If both Spark1 and Spark2 are installed, you must modify cdap-env to 
set SPARK_MAJOR_VERSION and SPARK_HOME:

$ vi .bash_profile
export SPARK_MAJOR_VERSION=2
export SPARK_HOME=/usr/hdp/2.5.0.0-1245/spark2

When Spark2 is in use, Spark1 programs cannot be run. Similarly, when Spark1 is in use, Spark2 programs cannot be run.

```

## Modified .bash_profile
```
vi .bash_profile
export HADOOP_USER_NAME=hdfs
```

## Spark Master/Slave installation in Multi Node
```

# hdpmaster.yogya.com
=====================

$ vi /usr/hdp/2.5.0.0-1245/spark2/conf/spark-env.sh

export SPARK_MASTER_HOST=hdpmaster.yogya.com
export SPARK_LOCAL_IP=172.16.9.207
export SPARK_MASTER_PORT=7078
export SPARK_MASTER_WEBUI_PORT=8180
export SPARK_WORKER_MEMORY=1g
export SPARK_WORKER_INSTANCES=2
#export SPARK_WORKER_DIR=/usr/hdp/2.5.0.0-1245/spark-data


$ vi /usr/hdp/2.5.0.0-1245/spark2/conf/slaves

hdpmaster.yogya.com
hdpmaster1.yogya.com
datanode1.yogya.com
datanode2.yogya.com


=========================
Importan In All Node
=========================
$ mkdir -p /var/run/spark2/work
$ chmod -R 777 /var/run/spark2/work

Do this in all Node Slave1 and Slave2

====================
Run Spark Cluster
====================
$ cd /usr/hdp/2.5.0.0-1245/spark2/
$ sbin/start-master.sh
$ sbin/start-slaves.sh
$ jps
master
worker
worker

In Machine-2
============
$ jps
worker
worker

In Machine-3
============
$ jps
worker
worker


====================
# Spark Cluster Master
====================
http://hdpmaster.yogya.com:8180/

URL: spark://node-master.yogya.com:7078
REST URL: spark://node-master.yogya.com:6066 (cluster mode)
Alive Workers: 0
Cores in use: 0 Total, 0 Used
Memory in use: 0.0 B Total, 0.0 B Used
Applications: 0 Running, 0 Completed
Drivers: 0 Running, 0 Completed
Status: ALIVE


=============================================
Setting di R client 
=============================================

cd /yogyaapp/spark-2.0.0-bin-hadoop2.7/conf
vi hive-site.xml
<configuration>
<property>
<name>javax.jdo.option.ConnectionURL</name>
<value>jdbc:derby:memory:databaseName=metastore_db;create=true</value>
</property>
<property>
<name>javax.jdo.option.ConnectionDriverName</name>
<value>org.apache.derby.jdbc.EmbeddedDriver</value>
</property>
</configuration>

```

## Web UI Hadoop
```
The default Hadoop ports are as follows: (HTTP ports, they have WEB UI):
- Hadoop
http://172.16.7.115:50070/
```

## YARN
```
check Memory
# free -m

how to check what all jobs are running in my default queue for resource manager and how to flush them. I
need to do this through command line

$ yarn application -list

This will give you list of all SUBMITTED, ACCEPTED or RUNNING applications.
From this you can filter applications of default queue by running 
$ yarn application -list | grep default

To kill the application you can run yarn application -kill <Application ID>

# Kill all yarn ACCEPTED
$ for app in `yarn application -list | awk '$6 == "ACCEPTED" { print $1 }'`; do yarn application -kill "$app";  done


# Kill all yarn ACCEPTED
$ for app in `yarn application -list | awk '$6 == "RUNNING" { print $1 }'`; do yarn application -kill "$app";  done

```

## Hive
```
sometimes hive not running running out memory you have to kill YARN first
Workarround :	
$ hive -hiveconf hive.log.file=hivecli_debug.log -hiveconf hive.log.dir=/tmp/hivecli -hiveconf hive.root.logger=DEBUG,DRFA
$ check running Yarn JOB
$ yarn application -list | grep default
$ yarn application -kill appication_id
# check memory 
$ free -m
$ export HADOOP_USER_NAME=hdfs
$ hive
$ hive -e "show databases;"

====================================================
*********** HIVE TABLE NOT SHOWING  *********
=====================================================

Go to Ambari - Hive - Config

Search 
javax.jdo.option.ConnectionURL
jdbc:postgresql://hdpmaster.yogya.com:5432/hivemeta
Replace With 
jdbc:postgresql://hdpmaster.yogya.com:5432/hivemeta?createDatabaseIfNotExist=true

====================================================
*********** HIVE TABLE NOT SHOWING IN SPARK *********
=====================================================

go to Ambari hive - config
Search  -- Optional try to to true just alter table 
hive.compactor.initiator.on
false

-- Must 
ALTER TABLE dim_store COMPACT 'major';

===============
Explanation
===============

Compactor 
•Each transaction (or batch of transactions in streaming ingest) creates a new delta file
•Too many files = NameNode !
•Need a way to
–Collect many deltas into one delta – minor compaction
–Rewrite base and delta to new base – major compaction

Minor Compaction 
•Run when there are 10 or more deltas (configurable)
•Results in base + 1 delta
Minor Compaction
/hive/warehouse/purchaselog/ds=201403311000/base_0028000
/hive/warehouse/purchaselog/ds=201403311000/delta_0028001_0028100
/hive/warehouse/purchaselog/ds=201403311000/delta_0028101_0028200
/hive/warehouse/purchaselog/ds=201403311000/delta_0028201_0028300
/hive/warehouse/purchaselog/ds=201403311000/delta_0028301_0028400
/hive/warehouse/purchaselog/ds=201403311000/delta_0028401_0028500
/hive/warehouse/purchaselog/ds=201403311000/base_0028000
/hive/warehouse/purchaselog/ds=201403311000/delta_0028001_0028500 


Run when deltas are 10% the size of base
(configurable)
•Results in new base
Major Compaction
/hive/warehouse/purchaselog/ds=201403311000/base_0028000
/hive/warehouse/purchaselog/ds=201403311000/delta_0028001_0028100
/hive/warehouse/purchaselog/ds=201403311000/delta_0028101_0028200
/hive/warehouse/purchaselog/ds=201403311000/delta_0028201_0028300
/hive/warehouse/purchaselog/ds=201403311000/delta_0028301_0028400
/hive/warehouse/purchaselog/ds=201403311000/delta_0028401_0028500
                Become On file
/hive/warehouse/purchaselog/ds=201403311000/base_0028500 


Compactor Continued 
• Metastore thrift server will schedule and execute compactions
–No need for user to schedule
–User can initiate via ALTER TABLE COMPACT statement
• No locking required, compactions run at same time as select and DML
–Compactor aware of readers, does not remove old files until readers have finished with them
• Current compactions can be viewed using new
SHOW COMPACTIONS statement

Yes it is transnational table; See below *show create table. *
Table hardly have 3 records , and after triggering minor compaction on
tables , it start showing results on spark SQL.

SHOW COMPACTIONS;
ALTER TABLE hivespark COMPACT 'major';
Alter table par_buk partition(dat='2017-10-09_12') compact 'major';
hadoop fs -ls /apps/hive/warehouse/par_buk/dat=2017-10-09_12/


So far we have seen build activities to make updates/deletes happen on Hive. To enable CRUD on Hive configuration files also needs some modification.

Below properties has to be changed in Hive-site.xml .

<property>
<name>hive.support.concurrency</name>
<value>true</value>
</property>
<property>
<name>hive.enforce.bucketing</name>
<value>true</value>
</property>
<property>
<name>hive.exec.dynamic.partition.mode</name>
<value>nonstrict</value>
</property>
<property>
<name>hive.txn.manager</name>
<value>org.apache.hadoop.hive.ql.lockmgr.DbTxnManager</value>
</property>
<property>
<name>hive.compactor.initiator.on</name>
<value>true</value>
</property>
<property>
<name>hive.compactor.worker.threads</name>
<value>2</value>
</property>
```

## HOW TO ADD Datanode in Cluster
```
Go to Ambari 
Host - Pilih Host/Mesin - Host Action - Stop All
<property>
  <name>dfs.datanode.data.dir</name>
  <value>/hadoop/hdfs/data,/hadoop/hdfs/data2</value>
  <final>true</final>
</property>

Warning : Please not user home directoory
```

## Install R
```
Install R first

Update: Finally I've resolved myself the problem updating the RHEL repo: 
cd /etc/yum.repos.d/
vi CentOS-base.repo
[base]
name=CentOS-$releasever – Base
baseurl=http://buildlogs.centos.org/centos/7/os/x86_64-20140704-1/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
priority=1
exclude=php mysql

And after that: 
$ yum update
$ cd /root/source/R
$ yum install R \
texinfo-tex-5.1-4.el7.x86_64.rpm  \
texlive-epsf-svn21461.2.7.4-43.el7.noarch.rpm \
texinfo-5.1-4.el7.x86_64.rpm \
tre-devel-0.8.0-10.el7.art.x86_64.rpm  \
blas-devel-3.4.2-4.el7.x86_64.rpm \
blas-3.4.2-4.el7.x86_64.rpm \
lapack-devel-3.4.2-4.el7.x86_64.rpm  \
lapack-3.4.2-4.el7.x86_64.rpm \
lapack-devel-3.4.2-4.el7.x86_64.rpm  \
tre-0.8.0-10.el7.art.x86_64.rpm

yum remove R-devel.x86_64 \
R.x86_64 \
R-java-devel.x86_64	3.2.3-4.el7 \
R-java.x86_64 \
R-core.x86_64 \
R-core-devel.x86_64	\
libRmath-devel.x86_64 \
libRmath.x86_64


You'll also need to install the Shiny R package and some other dependencies before installing Shiny Server:

$ yum install libcurl-devel.x86_64
$ su - -c "R -e \"install.packages(c('shiny', 'rmarkdown', 'devtools', 'RJDBC'), repos='http://cran.rstudio.com/')\""

$ R
q()

Uninstall R
Suppose you install R using yum, then you can use the following commands to totally uninstall R:

yum remove R
yum remove R-core
yum remove R-devel
yum remove R-core-devel
```

## Install Shiny Server
```
All the dependencies required for the Shiny server are installed now and we are ready to install it on CentOS 7. So let's download the latest stable release of the Shiny server using the following command.

$ wget https://download3.rstudio.org/centos6.3/x86_64/shiny-server-1.5.9.923-x86_64.rpm

Next, install the downloaded shiny server using the following command.
$ yum install --nogpgcheck shiny-server-1.5.9.923-x86_64.rpm

Note Error :
/var/tmp/rpm-tmp.9F5O5V: line 14: initctl: command not found
/var/tmp/rpm-tmp.9F5O5V: line 17: initctl: command not found

Solution
$ sudo cp /opt/shiny-server/config/systemd/shiny-server.service /etc/systemd/system/
$ sudo systemctl enable shiny-server
$ sudo systemctl restart shiny-server

Web Interface

Open up your favorite web browser and visit http://172.16.7.115:3838/ 
then you'll see a welcome webpage of Shiny server like this:

If you see the above-given output then the Shiny server is successfully installed on your CentOS 7 system. Now you'll need to start and enable the shiny server services. Execute the following commands and they'll do the job for you.

$ systemctl start shiny-server
$ systemctl enable shiny-server

Finally, you'll need to modify the firewall rules for your Shiny server to access it. Shiny server listen to port number 3838 by default.
firewall-cmd --permanent --zone=public --add-port=3838/tcp
firewall-cmd --reload

```

## Install RStudio
```
Installing R Studio Server
Installing R Studio on the same local network as the Spark cluster that we want to connect to - in our case directly on the master node - is the recommended approach for using R Studio with a remote Spark Cluster. Using a local version of R Studio to connect to a remote Spark cluster is prone to the same networking issues as trying to use the Spark shell remotely in client-mode (see part 2).

First of all we need the URL for the latest version of R Studio Server. Preview versions can be found here while stable releases can be found here. At the time of writing Sparklyr integration is a preview feature so I’m using the latest preview version of R Studio Server for 64bit RedHat/CentOS (should this fail at any point, then revert back to the latest stable release as all of the scripts used in this post will still run). Picking-up where we left-off in the master node’s terminal window, execute the following commands,

Can look for the latest available version here.
$ wget https://download2.rstudio.org/rstudio-server-rhel-1.1.463-x86_64.rpm
$ yum install rstudio-server-rhel-1.1.463-x86_64.rpm -y
$ yum install openssl-devel
$ yum install libxml2-devel.x86_64
$ yum install libcurl-devel.x86_64

Add some R libraries (could be done after Rstudio install)
$ R -e "install.packages('devtools', repos = 'http://cran.us.r-project.org')"
$ R -e "install.packages('ggplot2', repos = 'http://cran.us.r-project.org')"


Creating RStudio User
It is not advisable to use the root account with RStudio, instead, create a normal user account just for RStudio. The account can be named anything, and the account password will be the one to use in the web interface.
(requires a UID > 500)

$ adduser rstudio
$ passwd rstudio

Open Browser 
http://172.16.7.115:8787
Login : rstudio
passw : rstudio

Changing the default port for Rstudio Server
• By default it is set to 8787, but this is not exported from
• So, create/edit /etc/rstudio/rserver.conf and add the

single line www-port = 8090

Startup Rstudion
$ ps -ef | grep rstudio
root      9331     1  0 Jan07 ?        00:00:00 /usr/lib/rstudio-server/bin/rserver
$ kill -9 9331
$ rstudio-server start

Automating the starting of RStudio Server
• Create a script Rstudio in /etc/init.d
#!/bin/sh
/usr/lib/rstudio-server/extras/init.d/redhat/rstudio-server start
Don’t forget to run
chmod +x /etc/init.d/RStudio

Stop 
$rstudio-server stop

/*=================================*/
/* Error R initializing */
/*=================================*/

cd /home/akrian
sudo chown -R akrian:akrian .rstudio

1) mkdir /home/biology/.rstudio  
2) mkdir /home/biology/.rstudio/graphics-r3  
3) sudo chown -R biology:my_group .rstudio  

```

## Install Sparklyr
```
Open RStudio http://172.16.7.115:8787/

# Sys.setenv(http_proxy="172.16.13.62:8080")
# Sys.setenv(https_proxy="172.16.13.62:8080")

# Please make sure yum install openssl-devel libxml2-devel.x86_64 libcurl-devel.x86_64
install.packages(c('devtools'), repos='http://cran.rstudio.com/')
devtools::install_github("hadley/devtools")
devtools::install_github("r-lib/xml2")
devtools::install_github("rstudio/sparklyr")
devtools::install_github("tidyverse/dplyr")

# Start RSutdio
$ rstudio-server restart
$ ps aux | grep myuser
```
