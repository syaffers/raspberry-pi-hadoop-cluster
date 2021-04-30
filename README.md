# Setup a Raspberry Pi Hadoop Cluster

## Prerequisites
- 3 (or more) Raspberry Pi 4 Model B+
- HDMI monitor
- Micro-HDMI cable to HDMI cable
- Keyboard and mouse
- Ethernet cable
- Network switch

### Assumptions
- You already have the latest Raspberry Pi OS installed on the Raspberry Pis.
- You have connected the network switch to the internet by some uplink.

## Setting up the Raspberry Pis
First, we must get the Raspberry Pis up to speed before we download or install anything.

### Turning on the Raspberry Pi
Due some quirks, the recommended way to connect things to the Raspberry Pi is in the following order:

1. micro-HDMI input to HDMI 0 on the Raspberry Pi and then the HDMI output to the screen,
2. Ethernet cable from the Raspberry Pi to the network switch,
3. Keyboard and mouse to USB ports,
4. USB Type-C power output to the Raspberry Pi USB Type-C socket,
5. USB mains input to the mains switch/plug.

Give it some time to fully start up; you will be ready when you see the login screen (if you installed the headless version) or the desktop (if you installed the desktop version).

### Configuring the Raspberry Pi
The very first thing to do is to configure the Raspberry Pis to make it usable for a cluster. Open a terminal and type in the following command:

    sudo raspi-config

A text UI will then allow you to change several options for the Raspberry Pi. We will need to change the following options for each of the Raspberry Pi:

- System options:
    - Change password
    - Change hostname (make sure each Raspberry Pi gets a distinct hostname)
- Interface options:
    - Enable SSH
    - Enable VNC (optional; only if you have the desktop version installed)
- Localization options
    - Language set to en-US UTF-8
    - Timezone to Asia/Dubai (or wherever you are)
    - Keyboard to 104-keys + English (US)
    - WLAN to AE

Once we are done, we can reboot the Raspberry Pis.

## Updating the APT repository and software
A fresh install of Raspberry Pi OS can have outdated libraries and such which may cause trouble down the line. Open up a terminal and type in the following:

```bash
sudo apt update && sudo apt upgrade -y
```

This will update the Raspberry pi with the latest repository packages and software.

## Installing Java Development Kit 8
Hadoop is recommended to be installed with a certain version of Java. In particular, Hadoop 3.2.2 (which we will be using in this tutorial) requires Java version 1.8.0.

### Installing via APT
We can install this version of Java from the command line:

```bash
sudo apt install openjdk-8-jdk -y
```

Once the install is complete, check that you get the following when you run this command in the terminal:

```
$ java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-8u212-b01-1+rpi1-b01)
OpenJDK Client VM (build 25.212-b01, mixed mode)
```

The version number after the `_` might be different depending on how far in the future you may be doing this tutorial but as long as `1.8.0` is installed, you should be good.

### Setting the `JAVA_HOME` environment variable
It is common to set the `JAVA_HOME` environment variable after installing Java. The `JAVA_HOME` environment variable refers to a folder in the filesystem where the Java binaries and libraries are stored. However, it can be confusing as to where this folder is. One way to probe for this folder is to use the `readlink` command to find where the `java` program is really at.

```
(~/) $ which java
/usr/bin/java  # This is a general binaries folder. Not quite...

(~/) $ readlink $(which java)
/etc/alternatives/java  # This is yet another general binaries folder...

(~/) $ readlink $(readlink $(which java))
/usr/lib/jvm/java-8-openjdk-armhf/jre/bin/java  # BINGO!
```

Java should be installed somewhere under the `/usr/lib` folder in Debian distributions. This may be different in other flavors of Linux. In any case, what we need is just the path up to the `jre` folder.

To set the environment variable, edit the `~/.bashrc` file and add the following lines to the bottom of the file.

```bash
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-armhf/jre"
```

## Installing Hadoop
Hadoop can be installed rather simply by downloading the precompiled binaries and moving it to a suitable location so that it can be accessed by the `pi` user.

__To minimize workload:__ run these commands on a single Raspberry Pi and use `rsync` later to sync all the folders with all the Pis.

### Downloading Hadoop
We will be using *Hadoop 3.2.2* for this tutorial. You can find the link to the [download page][1] here. On that site, select the HTTP mirror and download `hadoop-3.2.2.tar.gz`.

Depending on where the file is downloaded, we will need to go into that folder in the terminal and extract the files. This is done with a one-liner:

```bash
cd Downloads  # or wherever it is you downloaded Hadoop
tar xvf hadoop-3.2.2.tar.gz
```

There will be a bunch of outputs; but once everything is done, there should be a folder called `hadoop-3.2.2.tar.gz` in the directory. To make it more easily accessible, we will move it into the `/opt` folder.

```bash
sudo mv hadoop-3.2.2 /opt
```

### Enabling Hadoop binaries in Bash
While Hadoop is installed in a directory where it is workable, the Hadoop binaries in `/opt/hadoop-3.2.2/bin` is not accessible to the command line yet. To remedy this, edit the `~/.bashrc` file and include the following line.

```bash
export PATH='/opt/hadoop-3.2.2/bin:/opt/hadoop-3.2.2/sbin:$PATH'
```

Once you close and reopen the terminal window, you should be able to run the Hadoop commands in the binaries folder.

### Testing that Hadoop is working
If you haven't missed a step until now, chances are you will have Hadoop installed and ready to work with. To check that this is working you can run the command

```bash
hdfs dfs
# Possible outputs:
# a) ERROR: JAVA_HOME is not set and could not be found.
# b) Usage: hdfs [OPTIONS] SUBCOMMAND [SUBCOMMAND OPTIONS]
#    ... etc etc
```

If it returns an error such as in `a)`, check that your `JAVA_HOME` environment variable is set in `~/.bashrc`. You can check that is it set by typing

```bash
echo $JAVA_HOME
```

and ensuring that `JAVA_HOME` is set to the appropriate JRE folder. If you get `b)`, then you're good.

## Standalone Mode

*Refer to the [official Hadoop documentation][2] for more information regarding this section.*

Hadoop should be running in *Standalone Mode* right now. To check this, simply create a new folder and download some text data from the web.

```bash
mkdir input
wget -O input/input.txt https://raw.githubusercontent.com/karpathy/char-rnn/master/data/tinyshakespeare/input.txt
hadoop jar /opt/hadoop-3.2.2/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.2.jar wordcount input output
cat output/* | sort -n -k 2
```

However, this only really useful for checking that Hadoop is working on your Pi.

## Pseudo-Distributed Mode

*Refer to the [official Hadoop documentation][2] for more information regarding this section.*

You can simulate a Hadoop single-node cluster by configuring some of the XML files in the `/opt/hadoop-3.2.2/etc/hadoop` directory. But before going into that, we must first prepare the Pi so that it can do a few things.

### Generating a SSH key
To enable your Pi to log in to... itself (lol) without a password (lmao), just run

```bash
ssh-keygen -t rsa
```

Feel free to accept the defaults for the remaining questions. Now you have a SSH private key and public key. All that is left is to allow your Pi to trust itself as a SSH server.

```bash
ssh-copy-id localhost
```

Enter your password and done.

### Configuring Hadoop
Go into the directory `/opt/hadoop-3.2.2/etc/hadoop`.
We will update the `hadoop-env.sh` file and update the following section to include our `JAVA_HOME` environment variable and our Hadoop installation folder.

```bash
# hadoop-env.sh
...
# Technically, the only required environment variable is JAVA_HOME.
# All others are optional.  However, the defaults are probably not
# preferred.  Many sites configure these options outside of Hadoop,
# such as in /etc/profile.d

# The java implementation to use. By default, this environment
# variable is REQUIRED on ALL platforms except OS X!
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-armhf"

# Location of Hadoop.  By default, Hadoop will attempt to determine
# this location based upon its execution path.
export HADOOP_HOME="/opt/hadoop-3.2.2"
```

Also, we will edit the `core-site.xml` and `hdfs-site.xml` files and include the following lines, respectively

```xml
<!-- core-site.xml -->
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```
and
```xml
<!-- hdfs-site.xml -->
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

The `core-site.xml` configuration tells Hadoop where to serve the filesystem (in this case, it will be served on `localhost` at port `9000`), while the `hdfs-site.xml` configuration tells Hadoop what the replication factor of a file should be. In this case, we only replicate files once (i.e. not replicate files, since we only have node).

Finally, we will update the `mapred-site.xml` and `yarn-site.xml` to enable YARN-based processing.

```xml
<!-- yarn-site.xml -->
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```
and
```xml
<!-- mapred-site.xml -->
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

The only real important property here is the `mapreduce.framework.name` value being set to `yarn`. This allows YARN to control the cluster but since there's only one node at the moment, it's basically a single node YARN at the moment. The `yarn.nodemanager.env-whitelist` and `mapreduce.application.classpath` properties are there to enable YARN to find certain libraries and tools that it needs.

### Setting up HDFS
To ensure that HDFS will be used in the pseudo-distributed mode, we need to format the namenode.

```bash
hdfs namenode -format
```

We are now ready to run the Hadoop as a service (daemon). Simply run

```bash
start-dfs.sh
```

If you now point your browser to `localhost:9870`, you will see the Hadoop server running. You can find logs in the `/opt/hadoop-3.2.2/logs` directory.

Make the HDFS directories required to execute MapReduce jobs:

```bash
hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/<username>
```

### Starting YARN
Simply

```bash
start-yarn.sh
```

### Running a task and saving into HDFS
Copy the input folder from the Pi into HDFS:

```bash
hdfs dfs -put input input
```

Run a word count:

```bash
hadoop jar /opt/hadoop-3.2.2/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.2.jar wordcount input output
```

Look at the results. The `sort` part sorts the output so that the most common word is at the end:

```bash
hdfs dfs -cat output/* | sort -n -k 2
```

Download the file from HDFS:

```bash
hdfs dfs -get output
cat output/* | sort -n -k 2
```

## Distributed Cluster Mode

*Refer to the [official Hadoop documentation][3] for more information regarding this section.*

Now we are ready to set up the Hadoop cluster with workers and everything. Get all your 3 Raspberry Pis and connect them using the network switch with the uplink connected to port number 1 on the switch. Make sure the other Pis have Hadoop installed, follow the instructions of this guide up until (but not including) Standalone Mode.

Set one of the Pis as the master (or Namenode) and the remaining Pis as workers (or Datanodes).

**Note:** it is important that each Pi get a different hostname! We will only work on the master node and use SSH to move and copy files between the Pis.

### Setting up communication
First, let's find out what our IP address is:

```bash
ip addr show eth0 | grep inet
```

The output should be something like

    inet xx.xx.xx.xx/24 brd yy.yy.yy.255 scope global dynamic eth0
    inet6 zzzz::zzzz:zzzz:zzzz:zzzz/64 scope link

The `xx.xx.xx.xx` is your IP address. On the namenode, edit the `/etc/hosts` file and change the IP address of your PI to this value.

```bash
sudo nano /etc/hosts
```

Update the lines as follows.

```
# /etc/hosts

127.0.0.1	localhost
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

xx.xx.xx.xx master  # Would be something like `127.0.1.1 master` before
```

Now, we need to know what the IP address of the other Pis are. An easy way to get this information is to use `ping` on the hostname of the other Pis. For example:

```bash
ping <another-pis-hostname>.local  # e.g. ping worker-01.local
```

The output may look something like

```
PING worker-01.local (aa.aa.aa.aa) 56(84) bytes of data.
64 bytes from (aa.aa.aa.aa): icmp_seq=1 ttl=64 time=1.76 ms
64 bytes from (aa.aa.aa.aa): icmp_seq=2 ttl=64 time=0.751 ms
64 bytes from (aa.aa.aa.aa): icmp_seq=3 ttl=64 time=0.731 ms
```

Therefore `aa.aa.aa.aa` if the IP address of that worker. You should add this entry into `/etc/hosts`. Repeat the same steps for the other Pis to get their IP addresses and add them into the file too.

```
# /etc/hosts

127.0.0.1	localhost
::1		localhost ip6-localhost ip6-loopback
ff02::1		ip6-allnodes
ff02::2		ip6-allrouters

xx.xx.xx.xx	master
aa.aa.aa.aa	worker-01
bb.bb.bb.bb	worker-02
# etc ...
```

Once this file is completed, we must first enable password-less SSH from the master node to the worker nodes.

```bash
ssh-copy-id pi@worker-01
ssh-copy-id pi@worker-02
# etc ...
```

Once that's done, we must copy the `/etc/hosts` to all the other Pis so that all Pis have the same `/etc/hosts` file and is able to communicate with one another. This can be done by through the following command:

```bash
cat /etc/hosts | ssh pi@worker-01 "sudo tee /etc/hosts"
cat /etc/hosts | ssh pi@worker-02 "sudo tee /etc/hosts"
# etc ...
```

### Hadoop configuration for Cluster Mode

*Refer to the [official Hadoop documentation][3] for more information regarding this section.*

First thing we should edit is the `workers` file in the Hadoop configuration directory. This file contains the hostnames of the Pis that we are explicitly asking to be workers. In this case, we are only going to let the workers do the work and the master is going to be a manager.


**Note**: make sure you stop all HDFS and YARN daemons before going forward. An easy way to do this is just to run

```bash
stop-all.sh
```

Let's begin the configuration with the `workers` file. Just list all the hostname of the workers that you have according to the name in `/etc/hosts`.

**Note:** Assume all these files are in `/opt/hadoop-3.2.2/etc/hadoop`

```
# workers
worker-01
worker-02
```

Then we'll need to update a bunch of settings in the configuration folder (i.e. `core-site.xml`, `hdfs-site.xml`, `yarn-site.xml`). Let's start with the `core-site.xml`.

```xml
<!-- core-site.xml -->

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
</configuration>
```

Here we just change the hostname to `master`. Accordingly, you should set this value to the hostname of your master node as it is defined in the `/etc/hosts` file. Next is `hdfs-site.xml`.

```xml
<!-- hdfs-site.xml -->

<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
        <name>dfs.hosts</name>
        <value>/opt/hadoop-3.2.2/etc/hadoop/workers</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/opt/hadoop-3.2.2/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.name.dir</name>
        <value>/opt/hadoop-3.2.2/datanode</value>
    </property>
</configuration>
```

There are quite a single change and a few introductions here. First, the `dfs.replication` value has increased to 3. This value should be equal to the number of workers or datanodes that you plan to have. A good rule of thumb is to make it equal to the number of lines in your `workers` file. Then we added 2 more properties to indicate where the namenode and datanode directories are.

```xml
<!-- yarn.site -->

<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>master</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

Here we only added one property which is the `yarn.resourcemanager.hostname` and set it to the master node. This tells the master node to be the manager and the one to whom the workers will communicate with to perform tasks.

### Syncing the configuration

We've changed quite a few files here. What's the best way to sync all the configurations to all the Pis? `rsync`. Do this only from the master node to the worker nodes. `rsync` works by sending changes in directories; if a file has not been changed, it will not be copied.

```bash
rsync -azP /opt/hadoop-3.2.2/etc/hadoop/* pi@worker-01:/opt/hadoop-3.2.2/etc/hadoop
rsync -azP /opt/hadoop-3.2.2/etc/hadoop/* pi@worker-02:/opt/hadoop-3.2.2/etc/hadoop
# etc ...
```

### Formatting cluster HDFS

Before starting the HDFS, we need to create the namenode and datanode directories in each of the respective nodes.

Now to create the directories.

```bash
mkdir -p /opt/hadoop/namenode
ssh worker-01 "mkdir -p /opt/hadoop/datanode"
ssh worker-02 "mkdir -p /opt/hadoop/datanode"
# etc ...
```

Then we need to format HDFS.

```bash
hdfs namenode -format "whatever"  # Whatever cluster name you want.
```

### Starting the cluster

Same stuff:

```bash
start-dfs.sh
```

Make sure you point to `http://master:9870/dfshealth.html#tab-datanode` in the browser to check that the datanodes are alive. If not, a configuration is incorrect or you might be missing a worker or something. Try retracing back your steps.

Then:

```bash
start-yarn.sh
```

You can check `http://master:8088/cluster` to see if YARN is up and running.

```bash
mapred --daemon start historyserver
```

Finished or stopped jobs are listed here at `http://master:19888/jobhistory`.


### Running a large job
To test that everything is working, we should run a job on a big enough such that we can see all the workers actively processing things.

First, let's get a relatively bigger dataset than the one previously downloaded.

```bash
wget -O ~/wikitext-103-v1.zip https://s3.amazonaws.com/research.metamind.io/wikitext/wikitext-103-v1.zip
unzip wikitext-103-v1.zip
```

Then we need to put it into the HDFS.

```bash
hdfs dfs -put wikitext-103 wikitext
```

Now we are ready to run a word count job.

**Note:** to check that all workers are actually doing the work, open separate windows of terminals and `ssh` into the workers and run `htop` to see the processor used. If the workers' bars are jumping around that means you've got it right.

```bash
hadoop jar /opt/hadoop-3.2.2/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.2.jar wordcount wikitext wikiout
```

Once the job is done, you can view the output in HDFS.

```cat
hdfs dfs -cat wikiout/*
```

## Bonus: Hadoop Streaming

With Hadoop streaming, you can create your own functions rather than use the prepared tasks in the `hadoop-mapreduce-examples-3.2.2.jar`.

Suppose we want not just to get the most common word but rather to find the most common two words (or word bigrams). This functionality is not available in the example MapReduce JAR, however, you can program your own custom functions.

Rather than resorting to Java, which is the preferred programming language of Hadoop's JAR, we can instead use whatever program we want as long at it takes an input in the form of a stream.

### Writing the mapper and reducer

We will use Python to program our functions and we will need two functions: a mapper and a reducer. Let's start with the mapper.

```python
#!/usr/bin/env python3

import sys


for line in sys.stdin:
    line = line.strip()  # Removes any newline characters.
    tokens = line.split(' ')  # The wikitext words are separated by spaces.
    for i in range(len(tokens) - 1):
        bigram = f'({tokens[i]},{tokens[i+1]})'
        print(f'{bigram}')
```

The mapper just returns a tuple of bigrams. Now let's look at the reducer which just counts the bigrams as they appear in a sorted order.

```python
#!/usr/bin/env python3

import sys


prev_bigram = None

for line in sys.stdin:
    curr_bigram = line.strip()  # Removes any newline characters.

    if prev_bigram is None:
        prev_bigram = curr_bigram
        count = 1
        continue

    if prev_bigram == curr_bigram:
        count += 1
    else:
        print(f'{prev_bigram}\t{count}')
        prev_bigram = curr_bigram
        count = 1

print(f'{prev_bigram}\t{count}')
```

The reducer works by reading a sorted list of bigrams and keeping track of counts. When the current bigram changes, it will emit the last bigram and the count.

To test that this works, first enable the scripts to execute as programs.

```bash
chmod +x mapper.py reducer.py
```

Then simply run

```bash
cat wikitext-103/wiki.valid.tokens | ./mapper.py | sort | ./reducer.py
```

### Running the bigram counter on the cluster

To run it on the cluster we must use Hadoop streaming and pass in the file that we want and these mapper and reducer Python files as well.

```bash
mapred streaming -files mapper.py,reducer.py -input wikitext -output bigrams -mapper mapper.py -reducer reducer.py
```

**Note:** sometimes jobs can fail due to bad code or unforeseen edge cases. To debug, you would go to `master:8088` and find the job that failed and go into that worker's web service and go through the logs for the particular application that was running.

Once it is done you can have a look at the outputs as we did before. We'll sort it by the most frequent:

```bash
hdfs dfs -cat bigrams/* | sort -n -k 2
```

[1]: https://www.apache.org/dyn/closer.cgi/hadoop/common/hadoop-3.2.2/hadoop-3.2.2.tar.gz
[2]: https://hadoop.apache.org/docs/r3.2.2/hadoop-project-dist/hadoop-common/SingleCluster.html
[3]: https://hadoop.apache.org/docs/r3.2.2/hadoop-project-dist/hadoop-common/ClusterSetup.html
