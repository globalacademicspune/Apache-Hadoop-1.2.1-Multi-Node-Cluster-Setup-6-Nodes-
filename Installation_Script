# Hadoop 1.2.1 Multi-Node Cluster Installation Guide (6 Nodes)

## Step 1: Configure Host Entries

On all nodes, edit:

```bash
sudo vi /etc/hosts
```

Add:

```text
10.0.47.12   nn
10.0.36.84   snn
10.0.39.117  jt
10.0.33.218  dn1
10.0.38.139  dn2
10.0.47.194  dn3
```

---

## Step 2: Configure Passwordless SSH

Copy the SSH key:

```bash
scp -i hdfs.pem hdfs.pem ubuntu@<ip>:/home/ubuntu/.ssh
```

Move to SSH directory:

```bash
cd ~/.ssh
```

Set permissions:

```bash
chmod 400 hdfs.pem
```

Test login:

```bash
ssh -i hdfs.pem nn
ssh -i hdfs.pem snn
ssh -i hdfs.pem jt
ssh -i hdfs.pem dn1
ssh -i hdfs.pem dn2
ssh -i hdfs.pem dn3
```

Edit profile:

```bash
sudo vi ~/.profile
```

Add:

```bash
eval `ssh-agent`
ssh-add /home/ubuntu/.ssh/hdfs.pem
```

Reload profile:

```bash
source ~/.profile
```

---

## Step 3: Establish Connectivity Between Nodes

Copy profile:

```bash
scp .profile snn:~/
```

Copy SSH key:

```bash
scp .ssh/hdfs.pem snn:/home/ubuntu/.ssh
```

Verify connectivity:

```bash
ssh nn
ssh snn
ssh jt
ssh dn1
ssh dn2
ssh dn3
```

Update `/etc/hosts` on each node.

Set hostname:

```bash
sudo hostname nn
sudo hostname snn
sudo hostname jt
sudo hostname dn1
sudo hostname dn2
sudo hostname dn3
```

Disable strict host key checking:

```bash
sudo vi /etc/ssh/ssh_config
```

Change:

```text
#StrictHostKeyChecking ask
```

to

```text
#StrictHostKeyChecking no
```

---

## Step 4: Install DSH

```bash
sudo apt install dsh
```

Edit machine list:

```bash
sudo vi /etc/dsh/machines.list
```

Add:

```text
nn
snn
jt
dn1
dn2
dn3
```

Edit DSH configuration:

```bash
sudo vi /etc/dsh/dsh.conf
```

Add:

```text
remoteshell = ssh
```

Test:

```bash
dsh -a sudo apt update
```

---

## Step 5: Install Java and Hadoop on All Nodes

Install Java:

```bash
dsh -a sudo apt install openjdk-8-jre -y
dsh -a sudo apt install openjdk-8-jdk-headless -y
```

Download Hadoop:

```bash
dsh -a wget https://archive.apache.org/dist/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz
```

Extract:

```bash
dsh -a tar -xzvf hadoop-1.2.1.tar.gz
```

Create Hadoop directory:

```bash
dsh -a sudo mkdir -p /usr/local/hadoop
```

Move Hadoop:

```bash
dsh -a sudo mv hadoop-1.2.1 /usr/local/hadoop
```

Set ownership:

```bash
dsh -a sudo chown -R ubuntu:ubuntu /usr/local/hadoop
```

---

## Step 6: Configure Hadoop Environment

Edit:

```bash
vi ~/.bashrc
```

(On NN and JT)

```bash
export HADOOP_PREFIX=/usr/local/hadoop
export PATH=$PATH:$HADOOP_PREFIX/bin
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$PATH:$JAVA_HOME
```

Copy `.bashrc`:

```bash
scp .bashrc jt:~/
```

Configure Hadoop environment:

```bash
cd /usr/local/hadoop/hadoop-1.2.1/conf
sudo vi hadoop-env.sh
```

Add:

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_OPTS=-Djava.net.preferIPV4Stack=true
```

---

## Step 7: Configure Hadoop XML Files

### core-site.xml

```bash
sudo vi core-site.xml
```

```xml
<property>
    <name>fs.default.name</name>
    <value>hdfs://localhost:9000</value>
</property>

<property>
    <name>hadoop.tmp.dir</name>
    <value>/usr/local/hadoop/tmp</value>
</property>
```

### mapred-site.xml (JT)

```bash
sudo vi mapred-site.xml
```

```xml
<property>
    <name>mapred.job.tracker</name>
    <value>hdfs://localhost:9001</value>
</property>
```

### slaves (NN)

```bash
sudo vi slaves
```

```text
dn1
dn2
dn3
```

### masters (NN)

```bash
sudo vi masters
```

```text
snn
```

---

## Step 8: Distribute Configuration Files

```bash
scp hadoop-env.sh core-site.xml mapred-site.xml slaves masters snn:/usr/local/hadoop/hadoop-1.2.1/conf
```

---

## Step 9: Create Hadoop Temporary Directory

(On NN)

```bash
ssh nn
```

```bash
dsh -a mkdir /usr/local/hadoop/hadoop-1.2.1/tmp
```

---

## Step 10: Reload Environment Variables

```bash
dsh -a exec bash
```

---

## Step 11: Format NameNode

(On NN)

```bash
hadoop namenode -format
```

---

## Step 12: Start MapReduce Services

(On JT)

```bash
ssh jt
start-mapred.sh
```

---

## Step 13: Start HDFS Services

(On NN)

```bash
start-dfs.sh
```

---

## Step 14: Verify Hadoop Cluster

```bash
dsh -a jps
```

Expected services:

* NameNode
* SecondaryNameNode
* JobTracker
* DataNode
* TaskTracker

```
```
