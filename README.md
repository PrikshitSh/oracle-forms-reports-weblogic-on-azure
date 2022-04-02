# Create High Vailable Oracle Forms and Reports clusters on Azure

This document guides you to create high vailable Oracle Forms and Reports clusters on Azure VMs, including:
- Create Oracle Forms and Reports clusters with 2 replicas.
- Create load balancing with Azure Application Gateway
- Scale up with new Forms and Reports replicas
- Create High Available Adminitration Server
- Troubleshooting

## Contents

* [Prerequisites](#prerequisites)
* [Provision Azure WebLogic admin offer](#provision-azure-weblogic-admin-offer)
* [Create Oracle Database](#create-oracle-database)
* [Create Windows VM and set up XServer](#create-windows-vm-and-set-up-xserver)
* [Install Oracle Fusion Middleware Infrastructure](#install-oracle-fusion-middleware-infrastructure)
* [Install Oracle Froms and Reports](#install-oracle-froms-and-reports)
* [Clone machine for managed servers]()
* [Create schemas using RCU](#create-schemas-using-rcu)
* [Configure Forms and Reports with a new domain](#configure-forms-and-reports-in-the-existing-domain)
* [Create Load Balancing with Azure Application Gateway](#create-ohs-machine-and-join-the-domain)
* [Scale up with new Forms and Reports replicas](#apply-jrf-to-managed-server)
* [Create High Available Adminitration Server]()
* [Troubleshooting]()

## Prerequisites

An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/dotnet).

## Provision Azure WebLogic Virtual Machine

Azure provides s series of Oracle WebLogic base image, it'll save your effor for Oracle tools installation.
This document will setup Oracle Forms and Reports based on the Azure WebLogic admin offer, follow the steps to provison a WebLogic instance:

- Open [Azure portal](https://portal.azure.com/) from your browser.
- Search `WebLogic 12.2.1.4.0 Base Image and JDK8 on OL7.6`, you will find the WebLogic offers, select **WebLogic 12.2.1.4.0 Base Image and JDK8 on OL7.6**, and click **Create** button.
- Input values in the **Basics** blade:
  - Subscription: select your subscription.
  - Resource group: click **Create new**, input a name.
  - Virtual machine name: `adminVM`
  - Region: East US.
  - Image: WebLogic Server 12.2.1.4.0 and JDK8 on Oracle Linux 7.6 - Gen1.
  - Size: select a size with more than 8GiB RAM, e.g. Standard B4ms.
  - Authentication type: Password
  - Username: `weblogic`
  - Password: `Secret123456`
- Networking: you are able to bring your own VNET. If not, keep default settings.
- Keep other blads as default. Click **Review + create**.

It will take 10min for the offer completed. After the deployment finishes, you will have a machine with JDK and WLS installed. Then you are able to install and configure Forms and Reports on the top of the machine.

## Create Oracle Database

You are required to have to database to confiugre the JRF domain for Forms and Reports.This document will use Oracle Database.
Follow this [document](https://docs.microsoft.com/en-us/azure/virtual-machines/workloads/oracle/oracle-database-quick-create) to create an Oracle database

If you are following the document to create Oracle database, write down the credentials to create Forms schema, username and password should be: `sys/OraPasswd1`

## Create Windows VM and set up XServer

Though you have Oracle WebLogic instance running now, to create Oracle Forms and Reports, you still need to install Oracle Forms and Reports.
To simplify the interface, let's provison a Windows machine and leverage XServer to install required tools with graphical user interface.

Follow the steps to provision Windows VM and set up XServer.

- open the resource group.
- Click **Create** button to create a Windows machine.
- Select **Compute**, Select **Virtual machine**.
- Virtual machine name: windowsXServer
- Image: Windows 10 Pro
- Size: Stardard_D2s_v3
- Username: `weblogic`
- Password: `Secret123456`
- Check Licensing.
- Click **Review + create**.

Edit the security to allow access from your terminal.

- Open the resource group that your are working on.
- Select resource `wls-nsg`
- Select **Settings** -> **Inbound security rules**
- Click add
  - Source: Any
  - Destination port ranges: 3389,22
  - Priority: 330
  - Name: Allow_RDP_SSH
  - Click **Save**

After the Windows server is completed, RDP to the server.

- Install the XServer from https://sourceforge.net/projects/vcxsrv/.
- Disable the firewall to allow communication from WebLogic VMs.
  - Turn off Windows Defender Firewall

## Install Oracle Fusion Middleware Infrastructure

Download Oracle Fusion Middleware Infrastructure installer from https://download.oracle.com/otn/nt/middleware/12c/122140/fmw_12.2.1.4.0_infrastructure_Disk1_1of1.zip

Unzip the file and copy `fmw_12.2.1.4.0_infrastructure.jar` to **adminVM**.
Make sure `fmw_12.2.1.4.0_infrastructure.jar` is copied to /u01/oracle/fmw_12.2.1.4.0_infrastructure.jar, owner of the file is `oracle`, you can set the ownership with command `chown oracle:oracle /u01/oracle/fmw_12.2.1.4.0_infrastructure.jar`.

Now let's use the XServer to install Oracle Fusion Middleware Infrastructure on **adminVM**.

Steps to install Oracle Fusion Middleware Infrastructure on adminVM:

- RDP to windowsXServer.
- Click XLaunch from the desktop.
  - Multiple windows, Display number: `-1`, click Next.
  - Select "Start no client"
  - Check Clipboard and Primary Selection, Native opengl, Disable access control. Click Next.
  - Click Finish.

- Open CMD
- SSH to adminVM with command `ssh weblogic@adminVM`
- Use root user: `sudo su`
- Install dependencies
  ```
  # dependencies for XServer access
  sudo yum install -y libXtst
  sudu yum install -y libSM
  sudo yum install -y libXrender
  # dependencies for Forms and Reports
  sudo yum install -y compat-libcap1
  sudo yum install -y compat-libstdc++-33
  sudo yum install -y libstdc++-devel
  sudo yum install -y gcc
  sudo yum install -y gcc-c++
  sudo yum install -y ksh
  sudo yum install -y glibc-devel
  sudo yum install -y libaio-devel
  sudo yum install -y motif
  ```
- Open Port
  ```
  # for XServer
  sudo firewall-cmd --zone=public --add-port=6000/tcp
  # for admin server
  sudo firewall-cmd --zone=public --add-port=7001/tcp
  sudo firewall-cmd --zone=public --add-port=7002/tcp
  # for node manager
  sudo firewall-cmd --zone=public --add-port=5556/tcp
  # for forms and reports
  sudo firewall-cmd --zone=public --add-port=9001/tcp
  sudo firewall-cmd --zone=public --add-port=9002/tcp
  sudo firewall-cmd --runtime-to-permanent
  sudo systemctl restart firewalld
  ```
- Use `oracle` user: `sudo su - oracle`
- Get the private IP address of widnowsXServer, e.g. `10.0.0.8`
- Set env variable: `export DISPLAY=<yourWindowsVMVNetInternalIpAddress>:0.0`, e.g. `export DISPLAY=10.0.0.8:0.0`
- Set Java env: 
  ```
  oracleHome=/u01/app/wls/install/oracle/middleware/oracle_home
  . $oracleHome/oracle_common/common/bin/setWlstEnv.sh
  ```
- Install fmw_12.2.1.4.0_infrastructure.jar
  - Launch the installer
    ```
    java -jar fmw_12.2.1.4.0_infrastructure.jar
    ```
  - Continue? Y
  - Page1
    - Inventory Directory: `/u01/oracle/oraInventory`
    - Operating System Group: `oracle`
  - Step 3
    - Oracle Home: `/u01/app/wls/install/oracle/middleware/oracle_home`
  - Step 4
    - Select "Function Middleware infrastructure"
  - Installation summary
    - picture resources\images\screenshot-ofm-installation-summary.png
  - The process should be completed without errors.
  - Remove the installation file to save space: `rm fmw_12.2.1.4.0_infrastructure.jar`

## Install Oracle Froms and Reports

Following the steps to install Oracle Forms and Reports:
- Download wget.sh from https://www.oracle.com/middleware/technologies/forms/downloads.html#
  - Oracle Fusion Middleware 12c (12.2.1.4.0) Forms and Reports for Linux x86-64 for (Linux x86-64)
  - Oracle Fusion Middleware 12c (12.2.1.4.0) Forms and Reports for Linux x86-64 for (Linux x86-64)
- Copy the wget.sh to `/u01/oracle/wget.sh`
- Use the windowsXServer ssh to adminVM: `ssh weblogic@adminVM`.
- Use `oracle` user
- Set env variable: `export DISPLAY=<yourWindowsVMVNetInternalIpAddress>:0.0`, e.g. `export DISPLAY=10.0.0.8:0.0`
- Edit wget.sh, replace `--ask-password` with `--password <your-sso-password>`
- Run the script, it will download the installer.
  - `bash wget.sh`
  - input your SSO account name.
- Unzip the zip files: ` unzip "*.zip"`, you will get `fmw_12.2.1.4.0_fr_linux64.bin` and `fmw_12.2.1.4.0_fr_linux64-2.zip`
- Remove the zip files to save space
  ```
   rm V983392-01_1of2.zip
   rm V983392-01_2of2.zip
  ```
- Install Forms: `./fmw_12.2.1.4.0_fr_linux64.bin`
  - The installation dialog should prompt up, if no, set `export PS1="\$"`, run `./fmw_12.2.1.4.0_fr_linux64.bin` again.
  - Inventory Directory: `/u01/oracle/oraInventory`
  - Operating System Group: `oracle`
  - Step 3:
    - Oracle Home: `/u01/app/wls/install/oracle/middleware/oracle_home`
  - Step 4:
    - Forms and Reports Deployment
  - Step 5:
    - JDK Home: /u01/app/jdk/jdk1.8.0_291
  - Step 6: if there is error of operation system packages, install the conrresponding package and run `./fmw_12.2.1.4.0_fr_linux64.bin` again.
    - Error like "Checking for compat-libcap1-1.10;Not found", then run `sudo yum install compat-libcap1` to install the `compat-libcap1` package.
  - The installation should be completed without errors.

Now you have Forms and Reports installed in the adminVM. Let's clone the machine for managed servers.

## Clone machine for managed servers

You have Oracle Forms and Reports installed in the adminVM, we can clone adminVM for managed servers.

Follow the steps to clone adminVM and create two VMs for Forms and Reports replicas.

Create the a snapshot from adminVM OS disk:
- Open Azure portal, stop adminVM.
- Create a snapshot from OS disk.


Create VMs for Forms and Reports replicas based on the snapshot:
1. Create a disk from the snapshot.
2. Create a VM with name `mspVM1` on the disk.
3. ssh to the machine, use `root` user.
    - Set hostname: `hostnamectl set-hostname mspVM1`
4. Repeat step1-3 for `mspVM2`, make sure setting hostname with `mspVM2`.


## Create schemas using RCU

- Use the windowsXServer.
- SSH to adminVM
- Use `oracle` user
- Set env variable: `export DISPLAY=<yourWindowsVMVNetInternalIpAddress>:0.0`, e.g. `export DISPLAY=10.0.0.8:0.0`
- `bash /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/bin/rcu`
- Step2: Create Repository -> System Load and Product Load
- Step3: Input the connection information of Oracle database.
- Step4: please note down the prefix, whith will be used in the following configuration.
  - STB
  - OPSS
  - IAU
  - IAU_APPEND
  - IAU_VIEWER
  - MDS
  - WLS
- Step5: Use same passwords for all schemas. Value: `Secret123456`
  - Note: you must use the same password of WebLogic admin account.
- The schema should be completed without error.

## Configure Forms and Reports in the existing domain

- Use the windowsXServer.
- SSH to adminVM
- Use `oracle` user
- Set env variable: `export DISPLAY=<yourWindowsVMVNetInternalIpAddress>:0.0`, e.g. `export DISPLAY=10.0.0.8:0.0`
- `bash  /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/common/bin/config.sh`
- Page1:
  - Update an existing domain
  - location: /01/domains/wlsd
- Page2: 
  - FADS
  - Oracle Forms
  - Oracle Reports Application
  - Oracle Enterprise Manager
  - Oracle Reports Tools
  - Oracle WSM Policy Manager
  - Oracle JRF
  - ORacle WebLogic Coherence Cluster Extension
- Page3:
  - Application location: /u01/domains/applications
- Page4: 
  - RCU Data
  - Host Name: the host name of database
  - DBMS/Service: your dbms
  - Schema Owner: `<the-rcu-schema-prefix>_STB`
  - Schema Password: `Secret123456`
- Page7:
  - Topology
  - System Components
  - Deployment and Services
- Page14: Machines
  - Remove AdminServerMachine
- Page15: Assign Servers to Machine
  - adminVM
    - WLS_FORMS
    - WLS_REPORTS
- Page19: Assign System Component
  - adminVM
    - SystemComonent
      - forms
- Page20: Deployments Targeting
  - AdminServer
    - admin
      - DMS Application#12.2.1.1.0
      - coherence-transaction-rar
      - em
      - fads#1.0
      - fads-ui#1.0
      - opss-rest
      - state-management-provider-menory...
      - wlsm-pm
- The process should be completed withour error.
- Exit `oracle` user
- Start node manager: `sudo systemctl start wls_nodemanager`
- Start weblogic: `sudo systemctl start wls_admin`
- Open ports for Forms and Reports
  ```
  sudo firewall-cmd --zone=public --add-port=9001/tcp
  sudo firewall-cmd --zone=public --add-port=9002/tcp
  sudo firewall-cmd --runtime-to-permanent
  sudo systemctl restart firewalld
  ```

Start Forms and Reports server from admin console.
- Open WebLogic Admin Console from browser, and login
- Select Environment -> Servers -> Control
- Start WLS_FORMS and WLS_REPORTS
- The two servers should be running.

Edit the security to allow access to Forms and Reports:
- Open the resource group that your are working on.
- Select resource wls-nsg
- Select Settings -> Inbound security rules
- Click add
- Source: Any
- Destination：IP Address
- Destination IP addresses/CIDR ranges: IP address of adminVM
- Destination port ranges: 9001,9002
- Priority: 340
- Name: Allow_FORMS_REPORTS
- Click Save

## Apply JRF to managed server

We have to apply JRF to WebLogic dynamic cluster, otherwise, we can not use admin console or em to managed the dynamic cluster.

Stop WebLogic managed servers from admin console.
- Login admin console.
- Select **Environment** -> **Clusters** -> **cluster1** -> **Control** -> **Start/Stop**
- Force stop all the servers that has name starting with "msp".


Pack domain configuration from adminVM.
- ssh to adminVM
- Use oracle user
- Pack domain
  ```
  cd /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/common/bin
  bash pack.sh -domain=/u01/domains/wlsd -managed=true -template=/tmp/cluster.jar -template_name="ofrwlsd"
  ```
- Exit oracle user
- Copy the cluster.jar to mspVM*.
  ```
  sudo scp /tmp/cluster.jar weblogic@mspVM*:/tmp/cluster.jar
  ```

Install libs and unpack domain to managed servers.
- RDP to windowsXServer.
- Setup XLaunch
- SSH to mspVM* with command `ssh weblogic@mspVM*`
- Install depedency: if you are using RHEL, you must install the packages.
  ```
  sudo yum install -y libXtst
  sudo yum install -y libSM
  sudo yum install -y libXrender
  ```
- Add Xport
  ```
  sudo firewall-cmd --zone=public --add-port=6000/tcp
  sudo firewall-cmd --runtime-to-permanent
  sudo systemctl restart firewalld
  ```
- Stop node manager
  ```
  sudo systemctl stop wls_nodemanager
  ```
- Allow the oracle user to access cluster.jar
  ```
  sudo chown oracle:oracle /tmp/cluster.jar
  ```
- Install Oracle Fusion Middleware Infrastructure on mspVM* following stpes in **Install Oracle Fusion Middleware Infrastructure** .
- Install Forms and Reports on mspVM* following steps in **Install Oracle Froms and Reports**
- Unpack the domain
  ```
  cd /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/common/bin
  unpack.sh -domain=/u01/domains/wlsd -template=/tmp/cluster.jar 
  ```
- Append class path for JRF.
  - Edit /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/common/bin/commExtEnv.sh with
  - Append the content after `WEBLOGIC_CLASSPATH="${WL_HOME}/server/lib/postgresql-42.2.8.jar:${WL_HOME}/server/lib/mssql-jdbc-7.4.1.jre8.jar:${WEBLOGIC_CLASSPATH}"`.
  ```
  export JRF_JAR_PATH="${MW_HOME}/oracle_common/modules/oracle.jps/jps-manifest.jar:${MW_HOME}/oracle_common/modules/internal/features/jrf_wlsFmw_oracle.jrf.wls.classpath.jar"
  WEBLOGIC_CLASSPATH="${JRF_JAR_PATH}:${WEBLOGIC_CLASSPATH}"
  ```
- Exit oracle user
- Start node manager
  ```
  sudo systemctl start wls_nodemanager
  ```

Start the managed servers from admin console.
- Login admin console.
- Select **Environment** -> **Clusters** -> **cluster1** -> **Control** -> **Start/Stop**
- Force start all the servers that has name starting with "msp".

## Create OHS machine and join the domain

Create machine from Azure Portal, use image: WebLogic Server 12.2.1.4 and JDK8 on OL7.6 - Gen1.
- Name: ohsVM2.

After the machine is created, ssh to weblogic@ohsVM2, and use `root` user.
- Install denpendencies.
  ```
  # for XServer
  sudo yum install -y libXtst
  sudu yum install -y libSM
  sudo yum install -y libXrender

  # for FORMS and REPORTS
  sudo yum install -y compat-libcap1
  sudo yum install -y compat-libstdc++-33
  sudo yum install -y libstdc++-devel
  sudo yum install -y gcc
  sudo yum install -y gcc-c++
  sudo yum install -y ksh
  sudo yum install -y glibc-devel
  sudo yum install -y libaio-devel
  sudo yum install -y motif
  ```
- Open port

  ```
  # for XServer
  sudo firewall-cmd --zone=public --add-port=6000/tcp
  
  # for WLS cluster
  sudo firewall-cmd --zone=public --add-port=7574/tcp
  sudo firewall-cmd --zone=public --add-port=7574/tcp
  sudo firewall-cmd --zone=public --add-port=7/tcp
  sudo firewall-cmd --zone=public --add-port=5556/tcp

  # for Coherence
  sudo firewall-cmd --zone=public --add-port=42000-42200/tcp
  sudo firewall-cmd --zone=public --add-port=42000-42200/udp  

  # for OHS
  sudo firewall-cmd --zone=public --add-port=7779/tcp
  sudo firewall-cmd --zone=public --add-port=7777/tcp
  sudo firewall-cmd --zone=public --add-port=4444/tcp
  sudo firewall-cmd --runtime-to-permanent
  sudo systemctl restart firewalld
  ```

- Install Oracle Fusion Middleware Infrastructure, see [steps](#install-oracle-fusion-middleware-infrastructure)
- Install Oracle Froms and Reports, see [steps](#install-oracle-froms-and-reports)


Configure domain
- SSH to adminVM
- Use `oracle` user
- Pack domain:
  ```
  cd /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/common/bin
  bash pack.sh -domain=/u01/domains/wlsd -managed=true -template=/tmp/cluster.jar -template_name="ofrwlsd"
  ```
- Exit oracle user.
- Copy the domain package to ohsVM2
  ```
  sudo scp /tmp/cluster.jar weblogic@ohsVM2:/tmp/cluster.jar
  ```
- ssh to ohsVM2.
- Allow the oracle user to access cluster.jar
  ```
  sudo chown oracle:oracle /tmp/cluster.jar
  ```
- Unpack domain
  ```
  cd /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/common/bin
  bash unpack.sh -domain=/u01/domains/wlsd -template=/tmp/cluster.jar 
  ```
- Append class path for JRF.
  - Edit /u01/app/wls/install/oracle/middleware/oracle_home/oracle_common/common/bin/commExtEnv.sh with
  - Append the content after WEBLOGIC_CLASSPATH="${WL_HOME}/server/lib/postgresql-42.2.8.jar:${WL_HOME}/server/lib/mssql-jdbc-7.4.1.jre8.jar:${WEBLOGIC_CLASSPATH}".
    ```
    export JRF_JAR_PATH="${MW_HOME}/oracle_common/modules/oracle.jps/jps-manifest.jar:${MW_HOME}/oracle_common/modules/internal/features/jrf_wlsFmw_oracle.jrf.wls.classpath.jar"
    WEBLOGIC_CLASSPATH="${JRF_JAR_PATH}:${WEBLOGIC_CLASSPATH}"
    ```
- Exit oracle user
- Create service for node manager, use `root` user
  ```
  cat <<EOF >/etc/systemd/system/wls_nodemanager.service
  [Unit]
  Description=WebLogic nodemanager service
  After=network-online.target
  Wants=network-online.target
  [Service]
  Type=simple
  # Note that the following three parameters should be changed to the correct paths
  # on your own system
  WorkingDirectory="/u01/domains/wlsd"
  ExecStart="/u01/domains/wlsd/bin/startNodeManager.sh"
  ExecStop="/u01/domains/wlsd/bin/stopNodeManager.sh"
  User=oracle
  Group=oracle
  KillMode=process
  LimitNOFILE=65535
  Restart=always
  RestartSec=3
  [Install]
  WantedBy=multi-user.target
  EOF
  ```
- Start node manager
  ```
  sudo systemctl enable wls_nodemanager
  sudo systemctl daemon-reload
  sudo systemctl start wls_nodemanager
  ```

Add the machine to existing domain.
- Login to EM portal.
- WebLogic Domain -> Environment -> Machines -> Create
  - Name: ohsVM2
  - Machine OS: Other
  - Listen Address: ohsVM2
  - Listen Port: 5556

Create OHS Server instance.
- Login to EM portal.
- WebLogic Domain -> Administration -> OHS Instances - Create
  - Instance name: ohs
  - Machine name: ohsVM2
  - Click OK

EM will create the OHS instance in ohsVM2.
Once the instance is completed, config Forms, Reports, WLs location.

Config Forms, Reports, WLS location, make sure the WebLogicCluster addresses are correct, may be string like: `mspVM1:8002,mspVM2:8003`
- SSH to ohsVM2
- Use oracle user.
  ```
  cat <<EOF >/u01/domains/wlsd/config/fmwconfig/components/OHS/instances/ohs/mod_wl_ohs.conf
  # NOTE : This is a template to configure mod_weblogic.

  LoadModule weblogic_module   "${PRODUCT_HOME}/modules/mod_wl_ohs.so"

  # This empty block is needed to save mod_wl related configuration from EM to this file when changes are made at the Base Virtual Host Level
  <IfModule weblogic_module>
        WLIOTimeoutSecs 900
        KeepAliveSecs 290
        FileCaching ON
        WLSocketTimeoutSecs 15
        DynamicServerList ON
        WLProxySSL ON
        WebLogicCluster mspVM1:8002,mspVM2:8003
  </IfModule>

  <Location /weblogic>
        SetHandler weblogic-handler
        DynamicServerList ON
        WLProxySSL ON
        WebLogicCluster mspVM1:8002,mspVM2:8003
  </Location>
  <Location /forms/>
        SetHandler weblogic-handler
        WebLogicHost adminVM
        WebLogicPort 9001
  </Location>
  <Location /reports>
      SetHandler weblogic-handler
      WebLogicHost adminVM
      WebLogicPort 9002
  </Location>
  EOF
  ```
- Please double check the content.

Note: if you deploy new applications to your WebLogic cluster, please add an entry to mod_wl_ohs.conf and restart ohs instance.

For an example, sample entry for application with context root `/myapplication`

```
<Location /myapplication>
        SetHandler weblogic-handler
        DynamicServerList ON
        WLProxySSL ON
        WebLogicCluster mspVM1:8002,mspVM2:8003
</Location>
```


Start OHS instance.

Open EM from browser, and start the ohs server.
- Login to EM portal.
- WebLogic Domain -> Administration -> OHS Instances - ohs
  - Start up.

## Validation

- Admin console: `http://<adminvm-ip>:7001/console`
- em: `http://<adminvm-ip>:7001/em`
- forms: `http://<adminvm-ip>:9001/forms/frmservlet` and `http://<ohs-ip>:7777/forms/frmservlet`
  - Please use JRE 32 bit + IE to access Forms.
- reports: `http://<adminvm-ip>:9002/reports/rwservlet` and `http://<ohs-ip>:7777/reports/rwservlet`
- Validate WLS cluster, if you have an application deployed to WLS cluster, you will be able to access the app via `http://<ohs-ip>:7777/<app-path>`
  ```
  curl http://<ohs-ip>:7777/weblogic/ready
  ```

