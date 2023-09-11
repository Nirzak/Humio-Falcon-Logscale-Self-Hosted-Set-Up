# Humio Falcon Logscale Self Hosted Single Node Server Set Up Guide
## Doc Updated As of: 10th September, 2023

## Requirements

- You need to be able to hold 48 hours of compressed data in 80% of your RAM.
- You want enough hyper-threads/vCPUs (each giving you 1GB/s search) to be able to search 24 hours of data in less than 10 seconds.
- You need disk space to hold your compressed data. Never fill your disk more than 80%.

Humio recommends at least 16 CPU cores, 32 GB of memory, and a 1 GBit Network card for single server set up on production environment.
Disk space will depend on the amount of ingested data per day and the number of retention days.
This is calculated as follows: Retention Days x GB Injected / Compression Factor. That will determine the needed disk space for a single server.
For more information read: <https://library.humio.com/humio-server/installation-provisioning-sizing.html>

**Set Up Used in this Doc:**

**Data:** 200GB partition at /data partition

**RAM:** 32GB

**CPU:** 4 Cores

**Kafka Version:** 2.13-3.3.2

**Zookeeper Version:** 3.7.1

**Humio Falcon Version:** 1.100.0





**Increase Open File Limit:**

For production usage, Humio needs to be able to keep a lot of files open for sockets and actual files from the file system.

You can verify the actual limits for the process using:

```
PID=`ps -ef | grep java | grep humio-assembly | head -n 1 | awk '{print $2}'`
cat /proc/$PID/limits | grep 'Max open files'
```

The minimum required settings depend on the number of open network connections and datasources. **There is no harm in setting these limits high for the Humio process. A value of at least 8192 is recommended**.

You can do that using a simple text editor to create a file named **99-humio-limits.conf** in the **/etc/security/limits.d/** sub-directory. Copy these lines into that file:

```
#Raise limits for files:
humio soft nofile 250000
humio hard nofile 250000
```

Create another file with a text editor, this time in the **/etc/pam.d/** sub-directory, and name it **common-session**. Copy these lines into it:

```
#Apply limits:
session required pam_limits.so
```

**Check noexec on /tmp directory**

Check the filesystem options on /tmp. Humio makes use of the Facebook Zstandard real-time compression algorithm, which requires the ability to execute files directly from the configured temporary directory.

The options for the filesystem can be checked using mount:

```
$ mount
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,nosuid,noexec,relatime,size=1967912k,nr_inodes=491978,mode=755,inode64)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,noexec,relatime,size=399508k,mode=755,inode64)
/dev/sda5 on / type ext4 (rw,relatime,errors=remount-ro)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noexec,seclabel)

```

You can temporarily remove noexec using **mount** to 'remount' the directory:

```
mount -oremount,exec /tmp
```
 

To permanently remove the noexec flag, update **/etc/fstab** to remove the flag from the options:

```
tmpfs /tmp tmpfs mode=1777,nosuid,nodev 0 0
```

## Installing JDK 17

**Humio requires a Java version 17 or later JVM to function properly.** (Doc updated as Humio will drop support for JDK version below 17 by the end of September 2023)

Download the latest JDK 17 x64 compressed archive from here: <https://www.oracle.com/java/technologies/javase/jdk17-archive-downloads.html>

**Installing JDK17 By unextracting the compressed archive:**
Extract the jdk tar archive to /usr/local/jdk directory

```
tar -xf jdk-17.0.8_linux-x64_bin.tar.gz -C /usr/local/jdk
cd /usr/local/jdk
mv jdk-17.0.8/* .

chmod -R o+rx /usr/local/jdk
```

## Installing Kafka

**At first add Kafka system user like the following**

```
$ sudo adduser kafka --shell=/bin/false --no-create-home --system --group
```

Download kafka:

```
$ curl -o kafka_2.13-3.3.2.tgz https://dlcdn.apache.org/kafka/3.3.2/kafka_2.13-3.3.2.tgz
```

**Create the following directories**
```
$ mkdir /data /usr/local/humio
```

**Untar kafka to /usr/local/humio directory**

```
$ sudo tar -zxf kafka_2.13-3.3.2.tgz /usr/local/humio/
```

**Create the following directory**

```
$ sudo mkdir -p /data/kafka/log /data/kafka/kafka-data
$ sudo chown -R kafka:kafka /data/kafka
$ sudo chown -R kafka:kafka /usr/local/humio/kafka_2.13-3.3.2
```

Navigate to `/usr/local/humio/kafka/config` and then edit the following details of the `server.properties` file.

```
broker.id=1
log.dirs=/data/kafka/log
delete.topic.enable = true
```

Now create a file named `kafka.service` under `/etc/systemd/system` directory and paste the following details inside that file.

```
[Unit]
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=kafka
LimitNOFILE=800000
Environment="JAVA_HOME=/usr/local/jdk"
Environment="LOG_DIR=/data/kafka/log/kafka"
Environment="GC_LOG_ENABLED=true"
Environment="KAFKA_HEAP_OPTS=-Xms512M -Xmx4G"
ExecStart=/usr/local/humio/kafka/bin/kafka-server-start.sh /usr/local/humio/kafka/config/server.properties
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**If we look at the \[Unit\] section, we’ll see it requires zookeeper service. So we have to install zookeeper before we can start kafka service.**

## Installing Zookeeper

Download Zookeeper:

```
$ curl -o zookeeper-3.7.1-bin.tar.gz https://dlcdn.apache.org/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz
```

Creating zookeeper system user

```
$ sudo adduser zookeeper --shell=/bin/false --no-create-home --system --group
```

**Untar Zookeeper to `/usr/local/humio` directory**

```
$ sudo tar -zxf zookeeper-3.7.1-bin.tar.gz /usr/local/humio/
$ sudo mkdir -p /data/zookeeper/data
$ sudo ln -s /usr/local/humio/apache-zookeeper-3.7.1-bin /usr/local/humio/zookeeper
$ sudo chown -R zookeeper:zookeeper /data/zookeeper
```

**Create a file named `zoo.cfg` inside `/usr/local/humio/zookeeper/conf` directory**

```
$ sudo vi /usr/local/humio/zookeeper/conf/zoo.cfg
```

**Paste the following lines inside that file**

```
tickTime = 2000
dataDir = /data/zookeeper/data
clientPort = 2181
initLimit = 5
syncLimit = 2
maxClientCnxns=60
autopurge.purgeInterval=1
admin.enableServer=false
4lw.commands.whitelist=*
server.1=<hostip>:2888:3888
admin.enableServer=false
```

Create a `myid` file in the `data` sub-directory with just the number `1` as its contents.

```
$ sudo bash -c 'echo 1 > /data/zookeeper/data/myid'
```

Mentioning jdk path variable so that zookeeper uses that

Create a file named `java.env` inside `/usr/local/humio/zookeeper/conf` directory and add the following line.

```
## Adding Custom JAVA_HOME

JAVA_HOME="/usr/local/jdk"
```

**Then you can start Zookeeper to verify that the configuration is working**

**Give proper permissions to zookeeper directory**

```
$ sudo chown -R zookeeper:zookeeper /usr/local/humio/apache-zookeeper-3.7.1-bin/
```

**Finally test if everything is ok or not by starting the server. ( You have to do it as root user)**

```
$ ./bin/zkServer.sh start
```

**If the server starts successfully it will show the following message**

```
/usr/local/java/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/apache-zookeeper-3.7.1-bin/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```

**Now we have to create a service to file to start zookeeper as a service. Create a `zookeeper.service` file inside `/etc/systemd/system/` directory and paste the following lines**

```
[Unit]
Description=Zookeeper Daemon
Documentation=http://zookeeper.apache.org
Requires=network.target
After=network.target

[Service]
Type=forking
WorkingDirectory=/usr/local/humio/zookeeper
User=zookeeper
Group=zookeeper
ExecStart=/usr/local/humio/zookeeper/bin/zkServer.sh start /usr/local/humio/zookeeper/conf/zoo.cfg
ExecStop=/usr/local/humio/zookeeper/bin/zkServer.sh stop /usr/local/humio/zookeeper/conf/zoo.cfg
ExecReload=/usr/local/humio/zookeeper/bin/zkServer.sh restart /usr/local/humio/zookeeper/conf/zoo.cfg
TimeoutSec=30
Restart=on-failure

[Install]
WantedBy=default.target
```

**After all things are done correctly start zookeeper by executing the below command. But before that check, if there is any log created by the root user inside `/usr/local/humio/zookeeper/logs` directory. If there are any logs, remove them.**

**Now start zookeeper.**

```
$ sudo systemctl start zookeeper
$ systemctl status zookeeper
$ sudo systemctl enable zookeeper
```

**After starting zookeeper we can now start Kafka.**

```
$ sudo systemctl enable kafka
$ sudo systemctl start kafka
$ systemctl status kafka
```

## Setting up Humio

**Create humio user**

```
$ sudo adduser humio --shell=/bin/false --no-create-home --system --group
```

**Creating humio directories**

```
$ sudo mkdir -p /data/humio/log/ /data/humio/data /usr/local/humio/humio_app
$ sudo chown -R humio:humio /etc/humio/ /data/humio
```

**We are now ready to download and install Humio’s software. Download latest humio stable version from here:**

[**https://repo.humio.com/repository/maven-releases/com/humio/server/1.100.0/server-1.100.0.tar.gz**](https://repo.humio.com/repository/maven-releases/com/humio/server/1.100.0/server-1.100.0.tar.gz)

```
$ curl -o server-1.100.0.tar.gz https://repo.humio.com/repository/maven-releases/com/humio/server/1.100.0/server-1.100.0.tar.gz

$ tar -xzf server-1.100.0.tar.gz

$ cd humio

$ mv * /usr/local/humio/humio_app

$ sudo chown -R humio:humio /usr/local/humio/humio_app
```

**Using a simple text editor, create the Humio configuration file, `server.conf` in the `/etc/humio` directory. You will need to enter a few environment variables in this configuration file to run Humio on a single server or instance. Below are those basic settings:**

```
$ sudo vi /etc/humio/server.conf
```

```
BOOTSTRAP_HOST_ID=1
DIRECTORY=/data/humio/data
JAVA_HOME=/usr/local/jdk
HUMIO_AUDITLOG_DIR=/data/humio/log
HUMIO_DEBUGLOG_DIR=/data/humio/log
JVM_LOG_DIR=/data/humio/log
HUMIO_PORT=8080
KAFKA_SERVERS=<hostip>:9092
EXTERNAL_URL=http://<hostip or domain>:8080
PUBLIC_URL=http://<hostip or domain>
HUMIO_SOCKET_BIND=0.0.0.0
HUMIO_HTTP_BIND=0.0.0.0
```

**In the last create the `humio.service` file inside `/etc/systemd/system/` directory and paste the below contents**

```
[Unit]
Description=Humio service
After=network.service

[Service]
Type=notify
Restart=on-abnormal
User=humio
Group=humio
LimitNOFILE=250000:250000
EnvironmentFile=/etc/humio/server.conf
WorkingDirectory=/data/humio
ExecStart=/usr/local/humio/humio_app/bin/humio-server-start.sh

[Install]
WantedBy=default.target
```

**Start the humio server by executing the following command:**

```
$ sudo systemctl start humio
```

**Check for any errors in humio end:**

```
$ journalctl -fu humio
```

If there is no error, you can access the humio site by visiting `http://<serverip>:8080` on your browser.
