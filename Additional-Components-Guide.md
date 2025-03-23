# Installing Additional Components for Hadoop Ecosystem on RHEL 8.10

This guide provides detailed instructions for installing and configuring additional components for your Hadoop ecosystem on Red Hat Enterprise Linux 8.10, integrating with Kerberos authentication.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Apache Ranger Installation](#apache-ranger-installation)
- [Apache Hue Installation](#apache-hue-installation)
- [Trino (PrestoSQL) Installation](#trino-installation)
- [FreeIPA Installation](#freeipa-installation)
- [Apache Airflow Installation](#apache-airflow-installation)
- [Kerberizing Additional Components](#kerberizing-additional-components)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before proceeding with installation of additional components, ensure you have:

1. A functioning Hadoop cluster with HDFS, YARN, and other core services running
2. Kerberos authentication properly configured
3. Red Hat Enterprise Linux 8.10 installed on all nodes
4. Access to repositories or installation packages
5. Administrative privileges on all servers

## Apache Ranger Installation

Apache Ranger provides a centralized security framework to manage fine-grained access control over Hadoop services.

### Install Ranger via Ambari (Recommended Approach)

If you have Ambari set up, the easiest way to install Ranger is through the Ambari UI:

1. **Add Ranger service in Ambari**:
   - Log into Ambari Web UI
   - Click on "Actions" → "Add Service"
   - Select "Ranger" from the list of services
   - Follow the wizard to install Ranger Admin, Ranger UserSync, and Ranger KMS

2. **Configure Ranger with PostgreSQL**:
   ```bash
   # Install PostgreSQL if not already installed
   dnf install -y postgresql-server postgresql-contrib
   
   # Initialize the database if not already done
   postgresql-setup --initdb
   
   # Start and enable PostgreSQL
   systemctl start postgresql
   systemctl enable postgresql
   
   # Create Ranger database and user
   sudo -u postgres psql << EOF
   CREATE DATABASE ranger;
   CREATE USER ranger WITH PASSWORD 'ranger';
   GRANT ALL PRIVILEGES ON DATABASE ranger TO ranger;
   \q
   EOF
   ```

3. **Configure Ranger in Ambari**:
   - In the Ambari Ranger configuration page, provide the following details:
     - DB Flavor: POSTGRES
     - Database Host: localhost (or your database server)
     - Ranger DB name: ranger
     - Ranger DB username: ranger
     - Ranger DB password: ranger
     - JDBC Driver: org.postgresql.Driver
     - JDBC URL: jdbc:postgresql://localhost:5432/ranger

4. **Set up Ranger authentication**:
   - In Ambari, navigate to Ranger → Configs → Advanced → Advanced ranger-admin-site
   - Set ranger.authentication.method to UNIX, LDAP, or ACTIVE_DIRECTORY based on your environment

### Manual Installation of Ranger

If you prefer to install Ranger manually or don't have Ambari:

1. **Download Ranger**:
   ```bash
   curl -O https://archive.apache.org/dist/ranger/2.1.0/apache-ranger-2.1.0.tar.gz
   tar -xzf apache-ranger-2.1.0.tar.gz
   cd apache-ranger-2.1.0
   ```

2. **Prepare environment**:
   ```bash
   # Install dependencies
   dnf install -y gcc python3-devel python3-pip
   
   # Install Maven
   dnf install -y maven
   ```

3. **Configure Ranger installation**:
   ```bash
   vi install.properties
   
   # Set the following properties:
   DB_FLAVOR=POSTGRES
   SQL_CONNECTOR_JAR=/usr/share/java/postgresql-jdbc.jar
   db_root_user=postgres
   db_root_password=<postgres_password>
   db_host=localhost
   db_name=ranger
   db_user=ranger
   db_password=ranger
   ```

4. **Run Ranger Admin setup**:
   ```bash
   ./setup.sh
   ```

5. **Configure Kerberos for Ranger**:
   ```bash
   vi install.properties
   
   # Add Kerberos configuration
   spnego_principal=HTTP/_HOST@EXAMPLE.COM
   spnego_keytab=/etc/security/keytabs/spnego.service.keytab
   token_valid=30
   cookie_domain=example.com
   cookie_path=/
   authentication_method=KERBEROS
   ```

6. **Install and configure each Ranger plugin**:
   - For HDFS:
     ```bash
     cd ranger-<VERSION>-hdfs-plugin
     vi install.properties
     
     # Configure plugin properties
     POLICY_MGR_URL=http://ranger-admin:6080
     REPOSITORY_NAME=hadoopdev
     COMPONENT_INSTALL_DIR_NAME=/usr/hdp/current/hadoop-client
     
     # Run setup
     ./enable-hdfs-plugin.sh
     ```

   - Similar process for other services (Hive, HBase, etc.)

### Verify Ranger Installation

1. **Access Ranger Admin UI**:
   - Navigate to http://ranger-admin-host:6080
   - Default credentials: admin/admin

2. **Add HDFS Service**:
   - Click "+" next to "HDFS"
   - Enter service details:
     - Service Name: hdfs
     - Username: hdfs
     - Password: (leave blank for Kerberos)
     - NameNode URL: hdfs://namenode:8020
     - Select "Kerberos Authentication"

3. **Create a simple policy**:
   - Navigate to the HDFS service
   - Click "Add New Policy"
   - Set resource path (e.g., /user/ranger)
   - Define permissions for users/groups

## Apache Hue Installation

Hue provides a web interface for interacting with Hadoop and other services.

### Install Hue via Ambari (with Management Pack)

1. **Install the Hue Ambari Management Pack**:
   ```bash
   cd /tmp
   curl -O https://archive.cloudera.com/p/HDP/hue-mpack/hue-ambari-mpack-4.6.0.tgz
   ambari-server install-mpack --mpack=/tmp/hue-ambari-mpack-4.6.0.tgz
   ambari-server restart
   ```

2. **Add Hue service through Ambari**:
   - Log into Ambari Web UI
   - Click on "Actions" → "Add Service"
   - Select "Hue" from the list of services
   - Follow the installation wizard

3. **Configure Hue in Ambari**:
   - Database configuration (PostgreSQL recommended)
   - Authentication settings (LDAP or Kerberos)
   - Service dependencies

### Manual Installation of Hue

1. **Install dependencies**:
   ```bash
   dnf install -y gcc gcc-c++ python3-devel python3-pip mariadb-devel openssl-devel krb5-devel openldap-devel libffi-devel nodejs npm
   ```

2. **Download and install Hue**:
   ```bash
   cd /opt
   curl -O https://cdn.gethue.com/downloads/hue-4.10.0.tgz
   tar -xzf hue-4.10.0.tgz
   cd hue-4.10.0
   ```

3. **Configure Hue**:
   ```bash
   vi desktop/conf/hue.ini
   
   # Set PostgreSQL as backend
   [[database]]
   engine=postgresql_psycopg2
   host=localhost
   port=5432
   user=hue
   password=hue
   name=hue
   
   # Configure HDFS
   [[hdfs_clusters]]
   [[[default]]]
   fs_defaultfs=hdfs://namenode:8020
   webhdfs_url=http://namenode:9870/webhdfs/v1
   
   # Configure YARN
   [[yarn_clusters]]
   [[[default]]]
   resourcemanager_host=resourcemanager
   resourcemanager_port=8032
   resourcemanager_api_url=http://resourcemanager:8088
   
   # Configure Hive
   [[hive]]
   hive_server_host=hiveserver
   hive_server_port=10000
   ```

4. **Configure Kerberos in Hue**:
   ```bash
   vi desktop/conf/hue.ini
   
   # Add Kerberos configuration
   [[kerberos]]
   hue_keytab=/etc/security/keytabs/hue.service.keytab
   hue_principal=hue/$(hostname -f)@EXAMPLE.COM
   kinit_path=/usr/bin/kinit
   reinit_frequency=3600
   ```

5. **Build and install Hue**:
   ```bash
   make apps
   
   # Create a database for Hue
   sudo -u postgres psql << EOF
   CREATE DATABASE hue;
   CREATE USER hue WITH PASSWORD 'hue';
   GRANT ALL PRIVILEGES ON DATABASE hue TO hue;
   \q
   EOF
   
   # Initialize the Hue database
   ./build/env/bin/hue syncdb
   ./build/env/bin/hue migrate
   ```

6. **Start Hue**:
   ```bash
   # Create a systemd service file
   cat > /etc/systemd/system/hue.service << 'EOF'
   [Unit]
   Description=Hue Service
   After=network.target
   
   [Service]
   Type=simple
   User=hue
   Group=hue
   ExecStart=/opt/hue-4.10.0/build/env/bin/supervisor
   Restart=on-failure
   
   [Install]
   WantedBy=multi-user.target
   EOF
   
   # Create user for running Hue
   useradd -r hue
   chown -R hue:hue /opt/hue-4.10.0
   
   # Start and enable Hue service
   systemctl daemon-reload
   systemctl start hue
   systemctl enable hue
   ```

### Verify Hue Installation

1. Access Hue web interface at http://hue-host:8888
2. Login with default admin/admin credentials
3. Go to "Hue Administration" to configure users and permissions

## Trino Installation

Trino (formerly PrestoSQL) is a distributed SQL query engine for big data analytics.

### Install Trino on RHEL 8.10

1. **Create a dedicated user**:
   ```bash
   useradd -r -m -d /opt/trino -s /bin/bash trino
   ```

2. **Download and install Trino**:
   ```bash
   mkdir -p /opt/trino
   curl -L https://repo1.maven.org/maven2/io/trino/trino-server/372/trino-server-372.tar.gz -o /tmp/trino-server.tar.gz
   tar -xzf /tmp/trino-server.tar.gz -C /opt/trino --strip-components=1
   chown -R trino:trino /opt/trino
   ```

3. **Create Trino configuration directories**:
   ```bash
   mkdir -p /opt/trino/etc
   mkdir -p /opt/trino/etc/catalog
   ```

4. **Configure Trino**:
   ```bash
   # Create node.properties
   cat > /opt/trino/etc/node.properties << 'EOF'
   node.environment=production
   node.id=$(uuidgen)
   node.data-dir=/opt/trino/data
   EOF
   
   # Create jvm.config
   cat > /opt/trino/etc/jvm.config << 'EOF'
   -server
   -Xmx16G
   -XX:+UseG1GC
   -XX:G1HeapRegionSize=32M
   -XX:+UseGCOverheadLimit
   -XX:+ExplicitGCInvokesConcurrent
   -XX:+HeapDumpOnOutOfMemoryError
   -XX:+ExitOnOutOfMemoryError
   EOF
   
   # Create config.properties for coordinator
   # For a single node setup, set both coordinator and worker to true
   cat > /opt/trino/etc/config.properties << 'EOF'
   coordinator=true
   node-scheduler.include-coordinator=false
   http-server.http.port=8080
   query.max-memory=50GB
   query.max-memory-per-node=1GB
   discovery-server.enabled=true
   discovery.uri=http://trino-coordinator:8080
   EOF
   
   # For worker nodes
   # cat > /opt/trino/etc/config.properties << 'EOF'
   # coordinator=false
   # http-server.http.port=8080
   # query.max-memory=50GB
   # query.max-memory-per-node=1GB
   # discovery.uri=http://trino-coordinator:8080
   # EOF
   ```

5. **Configure catalogs for different data sources**:
   ```bash
   # Create HDFS/Hive connector
   cat > /opt/trino/etc/catalog/hive.properties << 'EOF'
   connector.name=hive-hadoop2
   hive.metastore.uri=thrift://hive-metastore:9083
   hive.config.resources=/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml
   hive.allow-drop-table=true
   EOF
   
   # Create PostgreSQL connector example
   cat > /opt/trino/etc/catalog/postgresql.properties << 'EOF'
   connector.name=postgresql
   connection-url=jdbc:postgresql://postgres:5432/database
   connection-user=user
   connection-password=password
   EOF
   ```

6. **Configure Kerberos for Trino**:
   ```bash
   # Enable Kerberos authentication
   cat > /opt/trino/etc/security.properties << 'EOF'
   http.server.authentication.type=KERBEROS
   http.server.authentication.krb5.service-name=trino
   http.server.authentication.krb5.keytab=/etc/security/keytabs/trino.service.keytab
   http.authentication.krb5.config=/etc/krb5.conf
   internal-communication.kerberos.enabled=true
   internal-communication.kerberos.service-name=trino
   internal-communication.kerberos.keytab=/etc/security/keytabs/trino.service.keytab
   EOF
   
   # Create SPNEGO filter configuration
   cat > /opt/trino/etc/access-control.properties << 'EOF'
   access-control.name=file
   security.refresh-period=1m
   security.config-file=/opt/trino/etc/rules.json
   EOF
   
   # Create basic rules file
   cat > /opt/trino/etc/rules.json << 'EOF'
   {
     "catalogs": [
       {
         "user": ".*",
         "catalog": ".*",
         "allow": true
       }
     ]
   }
   EOF
   ```

7. **Set up Trino CLI**:
   ```bash
   curl -L https://repo1.maven.org/maven2/io/trino/trino-cli/372/trino-cli-372-executable.jar -o /tmp/trino-cli
   chmod +x /tmp/trino-cli
   mv /tmp/trino-cli /usr/local/bin/trino
   ```

8. **Create systemd service file**:
   ```bash
   cat > /etc/systemd/system/trino.service << 'EOF'
   [Unit]
   Description=Trino SQL Engine
   After=network.target
   
   [Service]
   Type=simple
   User=trino
   Group=trino
   ExecStart=/opt/trino/bin/launcher run
   Restart=on-failure
   
   [Install]
   WantedBy=multi-user.target
   EOF
   
   systemctl daemon-reload
   systemctl start trino
   systemctl enable trino
   ```

### Verify Trino Installation

1. **Check service status**:
   ```bash
   systemctl status trino
   ```

2. **Test connection with CLI**:
   ```bash
   trino --server localhost:8080 --catalog hive --schema default
   
   # With Kerberos
   trino --server https://trino-coordinator:8080 --catalog hive --schema default --krb5-config-path=/etc/krb5.conf --krb5-principal=user@EXAMPLE.COM --krb5-keytab-path=/path/to/user.keytab --krb5-remote-service-name=trino
   ```

3. **Access Web UI**:
   - Navigate to http://trino-coordinator:8080
   - For Kerberos-enabled setup, browser must be Kerberos-aware

## FreeIPA Installation

FreeIPA provides integrated security information management combining Linux, 389 Directory Server, MIT Kerberos, NTP, DNS, and more.

### Install FreeIPA Server

1. **Update system and install required packages**:
   ```bash
   dnf update -y
   dnf module enable -y idm:DL1
   dnf install -y ipa-server ipa-server-dns
   ```

2. **Configure firewall**:
   ```bash
   firewall-cmd --permanent --add-service=http --add-service=https --add-service=ldap --add-service=ldaps --add-service=kerberos --add-service=kpasswd --add-service=dns
   firewall-cmd --reload
   ```

3. **Configure server**:
   ```bash
   # Set hostname to FQDN
   hostnamectl set-hostname ipa.example.com
   
   # Run the IPA server installation
   ipa-server-install --setup-dns \
     --hostname=$(hostname -f) \
     --domain=example.com \
     --realm=EXAMPLE.COM \
     --ds-password=adminpassword \
     --admin-password=adminpassword \
     --mkhomedir \
     --no-ntp
   ```

4. **Configure NTP**:
   ```bash
   # Install chrony
   dnf install -y chrony
   
   # Configure chrony
   sed -i 's/^server /# server /g' /etc/chrony.conf
   echo "server $(hostname -f) iburst" >> /etc/chrony.conf
   
   # Start and enable chronyd
   systemctl start chronyd
   systemctl enable chronyd
   ```

5. **Verify FreeIPA installation**:
   ```bash
   kinit admin
   ipa user-find admin
   ```

### Install FreeIPA Client on Hadoop Nodes

1. **Install client packages**:
   ```bash
   dnf install -y ipa-client
   ```

2. **Join the IPA domain**:
   ```bash
   ipa-client-install --domain=example.com \
     --server=ipa.example.com \
     --realm=EXAMPLE.COM \
     --principal=admin \
     --password=adminpassword \
     --mkhomedir \
     --enable-dns-updates
   ```

### Integrate Hadoop with FreeIPA

1. **Create service principals in FreeIPA**:
   ```bash
   # Login as admin
   kinit admin
   
   # Create HDFS principals
   ipa service-add hdfs/$(hostname -f)@EXAMPLE.COM
   
   # Create HTTP principals for WebUI
   ipa service-add HTTP/$(hostname -f)@EXAMPLE.COM
   
   # Create principal for other services
   ipa service-add yarn/$(hostname -f)@EXAMPLE.COM
   ipa service-add hive/$(hostname -f)@EXAMPLE.COM
   ipa service-add hbase/$(hostname -f)@EXAMPLE.COM
   ipa service-add trino/$(hostname -f)@EXAMPLE.COM
   ipa service-add hue/$(hostname -f)@EXAMPLE.COM
   ```

2. **Create keytabs for services**:
   ```bash
   mkdir -p /etc/security/keytabs
   
   # Generate keytabs for each service
   ipa-getkeytab -s ipa.example.com -p hdfs/$(hostname -f)@EXAMPLE.COM -k /etc/security/keytabs/hdfs.service.keytab
   ipa-getkeytab -s ipa.example.com -p HTTP/$(hostname -f)@EXAMPLE.COM -k /etc/security/keytabs/spnego.service.keytab
   
   # Set appropriate ownership
   chown hdfs:hadoop /etc/security/keytabs/hdfs.service.keytab
   chmod 400 /etc/security/keytabs/hdfs.service.keytab
   
   chown hdfs:hadoop /etc/security/keytabs/spnego.service.keytab
   chmod 440 /etc/security/keytabs/spnego.service.keytab
   ```

3. **Configure SSSD for user authentication**:
   ```bash
   # SSSD configuration is managed by ipa-client-install
   # Verify configuration
   cat /etc/sssd/sssd.conf
   
   # Restart SSSD
   systemctl restart sssd
   ```

4. **Configure Hadoop to use IPA for authentication**:
   - In Ambari Web UI, navigate to HDFS → Configs
   - Search for "hadoop.security.authentication" and set to "kerberos"
   - Search for "hadoop.security.authorization" and set to "true"
   - Update Kerberos configurations to point to FreeIPA

### Test FreeIPA Integration

1. **Create a test user in FreeIPA**:
   ```bash
   kinit admin
   ipa user-add testuser --first=Test --last=User --password
   ```

2. **Test user access to HDFS**:
   ```bash
   # Login as testuser
   kinit testuser
   
   # Test access to HDFS
   hdfs dfs -mkdir /user/testuser
   hdfs dfs -ls /user/testuser
   ```

## Apache Airflow Installation

Apache Airflow is a platform for programmatically authoring, scheduling, and monitoring workflows.

### Install Airflow on RHEL 8.10

1. **Install dependencies**:
   ```bash
   # Install Python 3 and development tools
   dnf install -y python3 python3-devel python3-pip gcc gcc-c++
   
   # Install database dependencies (for PostgreSQL)
   dnf install -y postgresql-devel
   ```

2. **Create an Airflow user**:
   ```bash
   useradd -m -d /opt/airflow airflow
   ```

3. **Install Airflow**:
   ```bash
   # Create virtual environment
   python3 -m pip install --user virtualenv
   mkdir -p /opt/airflow
   cd /opt/airflow
   python3 -m virtualenv airflow_env
   
   # Activate virtual environment and install Airflow
   source airflow_env/bin/activate
   pip install wheel
   pip install apache-airflow==2.3.4 --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.3.4/constraints-3.9.txt"
   
   # Install PostgreSQL provider
   pip install apache-airflow-providers-postgres
   
   # Install Hadoop providers
   pip install apache-airflow-providers-apache-hdfs
   pip install apache-airflow-providers-apache-hive
   ```

4. **Configure Airflow**:
   ```bash
   # Set Airflow home
   export AIRFLOW_HOME=/opt/airflow
   
   # Create Airflow configuration
   mkdir -p /opt/airflow/config
   mkdir -p /opt/airflow/logs
   mkdir -p /opt/airflow/dags
   
   # Initialize the configuration
   airflow config list
   
   # Edit airflow.cfg
   vi /opt/airflow/airflow.cfg
   ```

5. **Configure PostgreSQL for Airflow**:
   ```bash
   # Create PostgreSQL database for Airflow
   sudo -u postgres psql << EOF
   CREATE DATABASE airflow;
   CREATE USER airflow WITH PASSWORD 'airflow';
   GRANT ALL PRIVILEGES ON DATABASE airflow TO airflow;
   \q
   EOF
   
   # Update Airflow config to use PostgreSQL
   sed -i 's#sql_alchemy_conn = sqlite:////opt/airflow/airflow.db#sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@localhost/airflow#g' /opt/airflow/airflow.cfg
   
   # Initialize the database
   airflow db init
   
   # Create Airflow admin user
   airflow users create \
       --username admin \
       --firstname Admin \
       --lastname User \
       --role Admin \
       --email admin@example.com \
       --password admin
   ```

6. **Configure Kerberos for Airflow**:
   ```bash
   # Install Kerberos dependencies
   pip install apache-airflow[kerberos]
   
   # Configure Kerberos in Airflow
   vi /opt/airflow/airflow.cfg
   
   # Update security section
   [kerberos]
   ccache = /tmp/airflow_krb5_ccache
   principal = airflow@EXAMPLE.COM
   reinit_frequency = 3600
   kinit_path = /usr/bin/kinit
   keytab = /etc/security/keytabs/airflow.service.keytab
   
   # Create Kerberos principal and keytab
   kinit admin
   ipa service-add airflow/$(hostname -f)@EXAMPLE.COM
   ipa-getkeytab -s ipa.example.com -p airflow/$(hostname -f)@EXAMPLE.COM -k /etc/security/keytabs/airflow.service.keytab
   chown airflow:airflow /etc/security/keytabs/airflow.service.keytab
   chmod 400 /etc/security/keytabs/airflow.service.keytab
   ```

7. **Create systemd service files**:
   ```bash
   # Create script to run Airflow services in virtual environment
   cat > /opt/airflow/run_airflow.sh << 'EOF'
   #!/bin/bash
   export AIRFLOW_HOME=/opt/airflow
   source /opt/airflow/airflow_env/bin/activate
   exec "$@"
   EOF
   
   chmod +x /opt/airflow/run_airflow.sh
   
   # Create webserver service
   cat > /etc/systemd/system/airflow-webserver.service << 'EOF'
   [Unit]
   Description=Airflow webserver
   After=network.target postgresql.service
   Wants=postgresql.service
   
   [Service]
   User=airflow
   Group=airflow
   Type=simple
   ExecStart=/opt/airflow/run_airflow.sh airflow webserver
   Restart=on-failure
   RestartSec=5s
   
   [Install]
   WantedBy=multi-user.target
   EOF
   
   # Create scheduler service
   cat > /etc/systemd/system/airflow-scheduler.service << 'EOF'
   [Unit]
   Description=Airflow scheduler
   After=network.target postgresql.service
   Wants=postgresql.service
   
   [Service]
   User=airflow
   Group=airflow
   Type=simple
   ExecStart=/opt/airflow/run_airflow.sh airflow scheduler
   Restart=on-failure
   RestartSec=5s
   
   [Install]
   WantedBy=multi-user.target
   EOF
   
   # Enable and start services
   systemctl daemon-reload
   systemctl enable airflow-webserver airflow-scheduler
   systemctl start airflow-webserver airflow-scheduler
   ```

8. **Configure Hadoop connection in Airflow**:
   ```bash
   # Create a sample DAG with Hadoop connection
   mkdir -p /opt/airflow/dags
   
   cat > /opt/airflow/dags/hadoop_test.py << 'EOF'
   from datetime import datetime
   from airflow import DAG
   from airflow.providers.apache.hdfs.sensors.hdfs import HdfsSensor
   from airflow.providers.apache.hive.operators.hive import HiveOperator
   
   default_args = {
       'owner': 'airflow',
       'depends_on_past': False,
       'start_date': datetime(2023, 1, 1),
       'email_on_failure': False,
       'email_on_retry': False,
       'retries': 1
   }
   
   dag = DAG(
       'hadoop_test',
       default_args=default_args,
       description='A simple DAG to test Hadoop integration',
       schedule_interval=None,
   )
   
   # Check if a file exists in HDFS
   check_file = HdfsSensor(
       task_id='check_file_exists',
       filepath='/user/airflow/test',
       hdfs_conn_id='hdfs_default',
       dag=dag
   )
   
   # Run a Hive query
   run_query = HiveOperator(
       task_id='run_hive_query',
       hql='SHOW TABLES',
       hive_conn_id='hive_default',
       dag=dag
   )
   
   check_file >> run_query
   EOF
   
   # Set proper permissions
   chown -R airflow:airflow /opt/airflow
   ```

### Configure Airflow Connections

After starting Airflow, log in to the web interface and configure the following connections:

1. **HDFS Connection**:
   - Conn Id: hdfs_default
   - Conn Type: HDFS
   - Host: hdfs://namenode:8020
   - Enable Kerberos Authentication: Yes
   - Kerberos Principal: airflow@EXAMPLE.COM
   - Kerberos Keytab: /etc/security/keytabs/airflow.service.keytab

2. **Hive Connection**:
   - Conn Id: hive_default
   - Conn Type: Hive Server 2 Thrift
   - Host: hiveserver
   - Login: (leave blank for Kerberos)
   - Port: 10000
   - Extra: {"use_kerberos": true, "kerberos_service_name": "hive", "auth_mechanism": "KERBEROS"}

### Verify Airflow Installation

1. **Check service status**:
   ```bash
   systemctl status airflow-webserver
   systemctl status airflow-scheduler
   ```

2. **Access web interface**:
   - Navigate to http://airflow-host:8080
   - Login with admin/admin credentials

3. **Trigger the test DAG**:
   - Go to DAGs
   - Find the 'hadoop_test' DAG
   - Unpause and trigger the DAG
   - Check logs for execution status

## Kerberizing Additional Components

### Create Kerberos Principals for Additional Services

If you're using MIT Kerberos (not FreeIPA), create the necessary principals:

```bash
# Login to kadmin
kadmin.local

# Create principals for each service
addprinc -randkey hue/$(hostname -f)@EXAMPLE.COM
addprinc -randkey trino/$(hostname -f)@EXAMPLE.COM
addprinc -randkey airflow/$(hostname -f)@EXAMPLE.COM

# Create keytabs
ktadd -k /etc/security/keytabs/hue.service.keytab hue/$(hostname -f)@EXAMPLE.COM
ktadd -k /etc/security/keytabs/trino.service.keytab trino/$(hostname -f)@EXAMPLE.COM
ktadd -k /etc/security/keytabs/airflow.service.keytab airflow/$(hostname -f)@EXAMPLE.COM

# Exit kadmin
quit
```

### Update Service Configurations for Kerberos

Ensure each service has the correct ownership and permissions for keytabs:

```bash
# Set proper ownership and permissions
chown hue:hue /etc/security/keytabs/hue.service.keytab
chmod 400 /etc/security/keytabs/hue.service.keytab

chown trino:trino /etc/security/keytabs/trino.service.keytab
chmod 400 /etc/security/keytabs/trino.service.keytab

chown airflow:airflow /etc/security/keytabs/airflow.service.keytab
chmod 400 /etc/security/keytabs/airflow.service.keytab
```

### Create Client-side Configurations

For users accessing these services, provide the necessary configuration files:

1. **Trino client configuration**:
   ```bash
   cat > ~/.trino/config.properties << 'EOF'
   kerberos.config.path=/etc/krb5.conf
   kerberos.principal=user@EXAMPLE.COM
   kerberos.keytab.path=/path/to/user.keytab
   kerberos.service.name=trino
   EOF
   ```

2. **Hue client setup**:
   - Navigate to http://hue-host:8888
   - Login with admin credentials
   - Go to "Hue Administration" > "Configuration"
   - Verify Kerberos configuration is set correctly

## Troubleshooting

### Common Issues with Additional Components

1. **Kerberos Authentication Issues**:
   ```bash
   # Check if keytab is valid
   klist -kt /etc/security/keytabs/service.keytab
   
   # Verify principal can authenticate
   kinit -kt /etc/security/keytabs/service.keytab service/hostname@EXAMPLE.COM
   
   # Look for authentication errors in logs
   grep -i kerberos /var/log/service/service.log
   ```

2. **Service Startup Failures**:
   ```bash
   # Check systemd service status
   systemctl status service-name
   
   # View service logs
   journalctl -u service-name
   
   # Check specific log files
   tail -f /var/log/service/service.log
   ```

3. **Database Connection Issues**:
   ```bash
   # Verify PostgreSQL is running
   systemctl status postgresql
   
   # Check database connectivity
   sudo -u postgres psql -c "SELECT 1;"
   
   # Test specific service database
   sudo -u postgres psql -d database -c "SELECT 1;"
   ```

4. **FreeIPA Integration Issues**:
   ```bash
   # Check if host is enrolled
   ipa-client-install --unattended --no-sudo --no-ntp --force-join
   
   # Test SSSD authentication
   id testuser
   
   # Restart SSSD
   systemctl restart sssd
   ```

### Component-Specific Troubleshooting

1. **Ranger Issues**:
   ```bash
   # Check Ranger admin logs
   tail -f /var/log/ranger/admin/xa_portal.log
   
   # Verify Ranger policies using REST API
   curl -u admin:admin -X GET http://ranger-admin:6080/service/public/v2/api/service
   
   # Restart Ranger services
   ranger-admin restart
   ranger-usersync restart
   ```

2. **Hue Issues**:
   ```bash
   # Check Hue server logs
   tail -f /var/log/hue/supervisor.log
   
   # Verify Hue database
   build/env/bin/hue dbshell
   
   # Restart Hue
   systemctl restart hue
   ```

3. **Trino Issues**:
   ```bash
   # Check Trino server logs
   tail -f /opt/trino/data/var/log/server.log
   
   # Verify Trino service is running
   netstat -tulpn | grep 8080
   
   # Test Trino CLI connectivity without Kerberos
   trino --server localhost:8080 --catalog system --schema runtime
   
   # Restart Trino
   systemctl restart trino
   ```

4. **FreeIPA Issues**:
   ```bash
   # Check IPA server status
   ipactl status
   
   # Test IPA server
   ipa ping
   
   # Check IPA logs
   tail -f /var/log/dirsrv/slapd-EXAMPLE-COM/access
   
   # Restart IPA services
   ipactl restart
   ```

5. **Airflow Issues**:
   ```bash
   # Check Airflow logs
   tail -f /opt/airflow/logs/scheduler/*.log
   tail -f /opt/airflow/logs/dag_processor_manager/*.log
   
   # Verify database connection
   python3 -c "from airflow.models import DagBag; print(DagBag().dags)"
   
   # Restart Airflow services
   systemctl restart airflow-webserver airflow-scheduler
   ```

### SELinux and Firewall Issues

1. **SELinux Troubleshooting**:
   ```bash
   # Check SELinux status
   getenforce
   
   # View SELinux alerts
   ausearch -m avc -ts recent
   
   # Generate SELinux policy for application
   audit2allow -a -M myapp
   semodule -i myapp.pp
   ```

2. **Firewall Troubleshooting**:
   ```bash
   # Check firewall status
   firewall-cmd --state
   
   # List allowed services and ports
   firewall-cmd --list-all
   
   # Add missing port
   firewall-cmd --permanent --add-port=port/protocol
   firewall-cmd --reload
   ```

This guide provides detailed setup instructions for additional Hadoop ecosystem components on RHEL 8.10 with Kerberos integration. Follow each section systematically for a successful deployment of your big data environment. 