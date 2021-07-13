![Intro](./docs/tyche.png)

ServiceNow (MID) Java Service for AWS

# Prerequisites

* AWS Console

# Create MID Server SSH Key

* Create SSH Key (e.g. johnsmith@acme.com)

    ```
    ssh-keygen -t rsa -b 4096 -C "johnsmith@acme.com" -f $HOME/.ssh/johnsmith
    ```

# Create EC2 MID Server

* Create an EC2 VM in your VPC and Subnet using a `Ubuntu 20.04.2 LTS` on a `t2.small` AMI

# Install MID Service

* SSH to MID Server

    ```
    ssh ubuntu@YOURMIDSERVERIP -i ~/.ssh/johnsmith
    ```

* Sudo as root

    ```
    sudo su - 
    ```

* Create MID User

    ```
    useradd -c "midserver" midserver -U -m -s /bin/bash
    ```

* Add MID User to sudoers

    ```
    usermod -aG sudo midserver
    ```

* Install Tools

    ```
    apt-get -y update && \
    apt-get install -qqy \
    wget unzip net-tools iftop && \
    rm -rf /var/lib/apt/lists/*
    ```

* Download Installer

    ```
    URL="https://install.service-now.com/glide/distribution/builds/package/app-signed/mid-linux-installer/2021/06/07/mid-linux-installer.quebec-12-09-2020__patch4-hotfix1-06-03-2021_06-07-2021_1407.linux.x86-64.deb"
    wget --progress=bar:force --no-check-certificate ${URL} -O mid.deb
    ```

* Run Installer

    ```
    dpkg --install mid.deb
    ...
    Setting up agent (19.4.1.1-1885.el7) ...
    MID Server has been installed at /opt/servicenow/mid
    MID Server can be configured using /opt/servicenow/mid/agent/installer.sh script
    ```

* Set Permissions

    ```
    chown -R midserver:midserver /opt/servicenow
    chmod -R 775 /opt/servicenow 
    ```

* Configure MID Server

    ```
    /opt/servicenow/mid/agent/installer.sh -silent \
    -INSTANCE_URL "https://hlastage01.service-now.com/" \
    -MUTUAL_AUTH N \
    -MID_USERNAME midserver \
    -MID_PASSWORD changeit \
    -USE_PROXY N \
    -MID_NAME midserver \
    -APP_NAME midserver \
    -APP_LONG_NAME midserver \
    -NON_ROOT_USER midserver
    ```

 * Fix Kill Mode Bug
 
    > https://support.servicenow.com/kb?id=kb_article_view&sysparm_article=KB0870356&sysparm_rank=3&sysparm_tsqueryId=18c33ef3dba3e850190b1ea6689619ae)

    ```
    vi /etc/systemd/system/midserver.service
    ...
    [Unit]
    Description=midserver
    After=syslog.target 

    [Service]
    Type=forking
    ExecStart=/opt/servicenow/mid/agent/bin/mid.sh start sysd
    ExecStop=/opt/servicenow/mid/agent/bin/mid.sh stop sysd
    PIDFile=/opt/servicenow/mid/agent/work/midserver.pid  << FIX LIKE THIS
    User=midserver
    KillMode=control-group <<  FIX IT LIKE THIS

    [Install]
    WantedBy=multi-user.target
    ...
    ```

* Fix Keystore Bug

    > See: https://community.servicenow.com/community?id=community_question&sys_id=7ca04c41db857f000be6a345ca9619f8#:~:text=go%20to%20your%20server%20where,that%20is%20working%20or%20not.&text=Forum%20Level%201-,go%20to%20your%20server%20where%20mid%20server%20has%20installed%20then,that%20is%20working%20or%20not.)

    ```
    rm /opt/servicenow/mid/agent/keystore/agent_keystore.jks
    ```

* Restart Mid Server with script

    ```
    cd /opt/servicenow/mid/agent
    /opt/servicenow/mid/agent/bin/mid.sh start
    /opt/servicenow/mid/agent/bin/mid.sh status
    ```

* Restart Mid Server with systemd

    ```
    systemctl daemon-reload
    systemctl start midserver
    systemctl status midserver
    ```

# Uninstall MID Service (Optional)

* Remove Mid Server

    ```
    /opt/servicenow/mid/agent/uninstall.sh
    rm -rf /opt/servicenow
    ```