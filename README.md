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

 

|                           |
| ------------------------- |
| mount -oremount,exec /tmp |

 

To permanently remove the noexec flag, update **/etc/fstab** to remove the flag from the options:

 

|                                                 |
| ----------------------------------------------- |
| **tmpfs /tmp tmpfs mode=1777,nosuid,nodev 0 0** |

**Installing JDK 11:**

**Humio requires a Java version 11 or later JVM to function properly.**

**Download the latest JDK 11 x64 compressed archive from here **[**https://www.oracle.com/java/technologies/javase-jdk11-downloads.html**](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)** **

**Installing JDK11:**

|                                                       |
| ----------------------------------------------------- |
| $ sudo yum -y install jdk-11.0.15.1_linux-x64_bin.rpm |

**If everything is alright you’ll see the following output**

|                                                                                                                                                            |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Java --versionJava(TM) SE Runtime Environment 18.9 (build 11.0.15.1+2-LTS-10)Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.15.1+2-LTS-10, mixed mode) |

**Installing Kafka**

**At first add Kafka system user like the following**

|                                                                           |
| ------------------------------------------------------------------------- |
| $ sudo adduser kafka --shell=/bin/false --no-create-home --system --group |

**Download kafka: **

|                                                                                               |
| --------------------------------------------------------------------------------------------- |
| $ curl -o kafka_2.13-3.3.1.tgz https&#x3A;//dlcdn.apache.org/kafka/3.2.0/kafka_2.13-3.3.1.tgz |

**Create the following directories**

|                                |
| ------------------------------ |
| $ mkdir /data /usr/local/humio |

**Untar kafka to /usr/local/humio directory**

|                                                        |
| ------------------------------------------------------ |
| $ sudo tar -zxf kafka_2.13-3.3.1.tgz /usr/local/humio/ |

**Create the following directory**

|                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| $ sudo mkdir -p /data/kafka/log/kafka /data/kafka/kafka-data$ sudo chown -R kafka:kafka /data/kafka$ sudo chown -R kafka:kafka /usr/local/humio/kafka_2.13-3.3.1 |

**Navigate to **/usr/local/humio/kafka/config** and then edit the following details of the **server.properties** file.**

|                                                                                                                      |
| -------------------------------------------------------------------------------------------------------------------- |
| **broker.****id****=1********log.****dirs****=/data/kafka/kafka-data********delete.topic.****enable**** = ****true** |

**Now create a file named “kafka.service” under /etc/systemd/system directory and paste the following details inside that file.**

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **\[Unit]********Requires=zookeeper.service********After=zookeeper.service************\[Service]********Type=simple********User=kafka********LimitNOFILE=****800000********Environment=****"LOG_DIR=/data/kafka/log/kafka"********Environment=****"GC_LOG_ENABLED=true"********Environment=****"KAFKA_HEAP_OPTS=-Xms512M -Xmx4G"********ExecStart=/usr/local/humio/kafka/bin/kafka-server-start.sh /usr/local/humio/kafka/config/server.properties********Restart=****on****-failure************\[Install]********WantedBy=multi-user.target** |

**If we look at the \[Unit] section, we’ll see it requires zookeeper service. So we have to install zookeeper before we can start kafka service.**

**Set up Zookeeper**

|                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------ |
| $ curl -o zookeeper-3.7.1-bin.tar.gz https&#x3A;//dlcdn.apache.org/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz |

  


**Creating zookeeper system user**

|                                                                               |
| ----------------------------------------------------------------------------- |
| $ sudo adduser zookeeper --shell=/bin/false --no-create-home --system --group |

**Untar Zookeeper to /opt directory**

|                                                                                                                                                                                                                                        |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| $ sudo tar -zxf zookeeper-3.7.1-bin.tar.gz /usr/local/humio/$ sudo mkdir -p /data/zookeeper/data$ sudo ln -s /usr/local/humio/apache-zookeeper-3.7.1-bin /usr/local/humio/zookeeper$ sudo chown -R zookeeper:zookeeper /data/zookeeper |

**Create a file named zoo.cfg inside /opt/zookeeper/conf directory**

|                                                   |
| ------------------------------------------------- |
| $ sudo vi /usr/local/humio/zookeeper/conf/zoo.cfg |

**Paste the following lines inside that file**

|                                                                                                                                                                                                                                                                                                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **tickTime = 2000********dataDir = /var/zookeeper/data********clientPort = 2181********initLimit = 5********syncLimit = 2********maxClientCnxns=60********autopurge.purgeInterval=1********admin.enableServer=false********4lw.commands.whitelist=\*********server.1=&lt;hostip>:2888:3888********admin.enableServer=false** |

**Create a ****myid**** file in the ****data**** sub-directory with just the number ****1**** as its contents. **

|                                                     |
| --------------------------------------------------- |
| $ sudo bash -c 'echo 1 > /data/zookeeper/data/myid' |

**Then you can start Zookeeper to verify that the configuration is working**

**Give proper permissions to zookeeper directory**

|                                                                                  |
| -------------------------------------------------------------------------------- |
| $ sudo chown -R zookeeper:zookeeper /usr/local/humio/apache-zookeeper-3.7.1-bin/ |

**Finally test if everything is ok or not by starting the server. ( You have to do it as root user)**

|                           |
| ------------------------- |
| $ ./bin/zkServer.sh start |

**If the server starts successfully it will show the following message**

|                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| /usr/local/java/bin/javaZooKeeper JMX enabled by defaultUsing config: /opt/apache-zookeeper-3.7.1-bin/bin/../conf/zoo.cfgStarting zookeeper ... STARTED |

  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  


**Now we have to create a service to file to start zookeeper as a service. Create a zookeeper.service file inside /etc/systemd/system/ directory and paste the following lines**

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \[Unit]Description=Zookeeper DaemonDocumentation=http&#x3A;//zookeeper.apache.orgRequires=network.targetAfter=network.target\[Service]Type=forkingWorkingDirectory=/usr/local/humio/zookeeperUser=zookeeperGroup=zookeeperExecStart=/usr/local/humio/zookeeper/bin/zkServer.sh start /usr/local/humio/zookeeper/conf/zoo.cfgExecStop=/usr/local/humio/zookeeper/bin/zkServer.sh stop /usr/local/humio/zookeeper/conf/zoo.cfgExecReload=/usr/local/humio/zookeeper/bin/zkServer.sh restart /usr/local/humio/zookeeper/conf/zoo.cfgTimeoutSec=30Restart=on-failure\[Install]WantedBy=default.target |

**After all things are done correctly start zookeeper by executing the below command. But before that check, if there is any log created by the root user inside **/usr/local/humio/zookeeper/logs** directory. If there are any logs, remove them.**

**Now start zookeeper.**

|                                                                                               |
| --------------------------------------------------------------------------------------------- |
| $ sudo systemctl start zookeeper$ systemctl status zookeeper$ sudo systemctl enable zookeeper |

**After starting zookeeper we can now start Kafka.**

|                                                                                   |
| --------------------------------------------------------------------------------- |
| $ sudo systemctl enable kafka$ sudo systemctl start kafka$ systemctl status kafka |

**Setting up Humio**

**Create humio user**

|                                                                           |
| ------------------------------------------------------------------------- |
| $ sudo adduser humio --shell=/bin/false --no-create-home --system --group |

**Creating humio directories**

|                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------- |
| $ sudo mkdir -p /data/humio/log/ /data/humio/data /usr/local/humio/humio_app$ sudo chown -R humio:humio /etc/humio/ /data/humio |

**We are now ready to download and install Humio’s software. Download latest humio from here:**

[**https://repo.humio.com/service/rest/repository/browse/maven-releases/com/humio/server/**](https://repo.humio.com/service/rest/repository/browse/maven-releases/com/humio/server/)

  


|                                                                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| $ curl -o server-1.41.jar https&#x3A;//repo.humio.com/repository/maven-releases/com/humio/server/1.41.0/server-1.41.0.jar$ sudo mv server-1.41.jar /usr/local/humio/humio_app/$ sudo ln -s /usr/local/humio/server-1.41.jar /usr/local/humio/humio_app/server.jar$ sudo chown -R humio:humio /usr/local/humio/humio_app |

**Using a simple text editor, create the Humio configuration file, server.conf in the /etc/humio directory. You will need to enter a few environment variables in this configuration file to run Humio on a single server or instance. Below are those basic settings:**

|                                  |
| -------------------------------- |
| $ sudo vi /etc/humio/server.conf |

|                                                                                                                                                                                                                                                                                                                                                             |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BOOTSTRAP_HOST_ID=1DIRECTORY=/var/humio/dataHUMIO_AUDITLOG_DIR=/data/humio/logHUMIO_DEBUGLOG_DIR=/data/humio/logHUMIO_PORT=8080ELASTIC_PORT=9200ZOOKEEPER_URL=&lt;hostip>:2181KAFKA_SERVERS=&lt;hostip>:9092EXTERNAL_URL=http&#x3A;//&lt;hostip or domain>:8080PUBLIC_URL=http&#x3A;//&lt;hostip or domain>HUMIO_SOCKET_BIND=0.0.0.0HUMIO_HTTP_BIND=0.0.0.0 |

**In the last create the humio.service file inside /etc/systemd/system/ directory and paste the below contents**

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \[Unit]Description=Humio serviceAfter=network.service\[Service]Type=notifyRestart=on-abnormalUser=humioGroup=humioLimitNOFILE=250000:250000EnvironmentFile=/etc/humio/server.confWorkingDirectory=/data/humioExecStart=/usr/bin/java -server -XX:+UseParallelOldGC -Xms12G -Xmx12G -XX:MaxDirectMemorySize=12G -Xss2M --add-exports java.base/jdk.internal.util=ALL-UNNAMED -XX:CompileCommand=dontinline,com/humio/util/HotspotUtilsJ.dontInline -Xlog:gc\*,gc+jni=debug:file=/data/humio/log/gc_humio.log:time,tags:filecount=5,filesize=102400 -Dhumio.auditlog.dir=/data/humio/log/ -Dhumio.debuglog.dir=/data/humio/log -jar /usr/local/humio/humio_app/server.jar\[Install]WantedBy=default.target |

On above service file edit the parameter **“-Xms12G -Xmx12G -XX:MaxDirectMemorySize=12G”** according to your server specification. The recommended value of maximum heap and minimum heap size is calculated the by following formula.

**Maximum heap, minimum heap, MaxDirecMemory = (Number of CPU Cores \* 1) + 8GB**

For example, If your CPU have **4 cores** then the value will be **4\*1 + 8 = 12GB**.

Be sure to increase your server’s RAM according to it. 

**Your server’s RAM must have to be >  (Maximum Heap Size + MaxDirectoryMemory)**. In this case 12GB. For more info read here: <https://library.humio.com/humio-server/configuration-jvm.html>

**Start the humio server by executing the following command:**

|                              |
| ---------------------------- |
| $ sudo systemctl start humio |

**Check for any errors in humio end:**

|                        |
| ---------------------- |
| $ journalctl -fu humio |

**If there is no error, you can access the humio site by visiting http&#x3A;//&lt;serverip>:8080 on your browser.**
