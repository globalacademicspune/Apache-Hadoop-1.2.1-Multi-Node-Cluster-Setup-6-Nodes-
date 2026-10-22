# Hadoop 1.2.1 — Complete Six-Node Cluster Installation

A complete step-by-step guide to installing and configuring an Apache Hadoop 1.2.1 cluster on six Ubuntu nodes using passwordless SSH authentication, Distributed Shell (DSH), HDFS, and classic MapReduce.

This guide covers hostname configuration, SSH key generation, Java installation, Hadoop setup, configuration file distribution, and cluster startup and verification.

## Table of Contents

1. [Cluster Architecture](#cluster-architecture)
2. [Prerequisites](#prerequisites)
3. [Step 1: Set Hostnames](#step-1-set-hostnames)
4. [Step 2: Configure `/etc/hosts`](#step-2-configure-etchosts)
5. [Step 3: Configure Passwordless SSH Using `ssh-keygen`](#step-3-configure-passwordless-ssh-using-ssh-keygen)
6. [Step 4: Configure `.profile`](#step-4-configure-profile)
7. [Step 5: Configure SSH Host-Key Checking](#step-5-configure-ssh-host-key-checking)
8. [Step 6: Install and Configure DSH](#step-6-install-and-configure-dsh)
9. [Step 7: Install Java 8](#step-7-install-java-8-on-all-nodes)
10. [Step 8: Download and Install Hadoop 1.2.1](#step-8-download-and-install-hadoop-121)
11. [Step 9: Configure `.bashrc`](#step-9-configure-bashrc)
12. [Step 10: Configure `hadoop-env.sh`](#step-10-configure-hadoop-envsh)
13. [Step 11: Configure `core-site.xml`](#step-11-configure-core-sitexml)
14. [Step 12: Configure `hdfs-site.xml`](#step-12-configure-hdfs-sitexml)
15. [Step 13: Configure `mapred-site.xml`](#step-13-configure-mapred-sitexml)
16. [Step 14: Configure `masters` and `slaves`](#step-14-configure-masters-and-slaves)
17. [Step 15: Copy Configuration Files](#step-15-copy-configuration-files-to-all-nodes)
18. [Step 16: Create Hadoop Directories](#step-16-create-hadoop-directories-on-all-nodes)
19. [Step 17: Format the NameNode](#step-17-format-the-namenode)
20. [Step 18: Start HDFS](#step-18-start-hdfs)
21. [Step 19: Start MapReduce](#step-19-start-mapreduce)
22. [Final Verification](#final-verification)
23. [Troubleshooting](#troubleshooting)
24. [Security Notes](#security-notes)

---

## Cluster Architecture

The cluster consists of six Ubuntu nodes.

| Node | Hostname | Role |
|---|---|---|
| Node 1 | `nn` | NameNode |
| Node 2 | `snn` | Secondary NameNode |
| Node 3 | `jt` | JobTracker |
| Node 4 | `dn1` | DataNode |
| Node 5 | `dn2` | DataNode |
| Node 6 | `dn3` | DataNode |

### Example IP Addresses

| Hostname | IP Address |
|---|---|
| `nn` | `172.31.6.219` |
| `snn` | `172.31.5.143` |
| `jt` | `172.31.3.70` |
| `dn1` | `172.31.11.103` |
| `dn2` | `172.31.14.128` |
| `dn3` | `172.31.5.116` |

Replace these addresses with the actual private IP addresses of your instances.

## Prerequisites

Before starting the installation, ensure the following:

- Six Ubuntu servers or virtual machines.
- Network connectivity between all six nodes.
- SSH access to all nodes for initial configuration.
- A user named `ubuntu` with `sudo` privileges.
- Internet connectivity to download Java and Hadoop.
- Sufficient disk space for Hadoop binaries and HDFS data.
- Required SSH and Hadoop ports permitted by the firewall or cloud security groups.

The commands below assume that the Hadoop installation is performed using the `ubuntu` account.

---

## Step 1: Set Hostnames

Assign a unique hostname to each node.

### On the NameNode (`nn`)

```bash
sudo hostnamectl set-hostname nn
```

### On the Secondary NameNode (`snn`)

```bash
sudo hostnamectl set-hostname snn
```

### On the JobTracker (`jt`)

```bash
sudo hostnamectl set-hostname jt
```

### On DataNode 1 (`dn1`)

```bash
sudo hostnamectl set-hostname dn1
```

### On DataNode 2 (`dn2`)

```bash
sudo hostnamectl set-hostname dn2
```

### On DataNode 3 (`dn3`)

```bash
sudo hostnamectl set-hostname dn3
```

### Verify the hostnames

Run the following command on each node:

```bash
hostname
```

Each node should return its assigned hostname.

---

## Step 2: Configure `/etc/hosts`

The `/etc/hosts` file allows each machine to resolve the other cluster nodes by hostname.

Perform this step on all six nodes.

Open the file:

```bash
sudo vi /etc/hosts
```

Keep the existing localhost entries and add the following:

```text
172.31.6.219 nn
172.31.5.143 snn
172.31.3.70 jt
172.31.11.103 dn1
172.31.14.128 dn2
172.31.5.116 dn3
```

Save and close the file.

### Verify hostname resolution

Run on `nn`:

```bash
getent hosts nn snn jt dn1 dn2 dn3
```

All six hostnames should resolve to the correct IP addresses.

---

## Step 3: Configure Passwordless SSH Using `ssh-keygen`

Hadoop uses SSH to start and manage services on remote cluster nodes. Passwordless SSH authentication allows these operations to run without repeatedly entering a login password.

This setup uses an RSA key pair with no passphrase.

### 3.1 Generate an RSA key pair on the NameNode

Run the following command on `nn` as the `ubuntu` user:

```bash
ssh-keygen -t rsa
```

When prompted for the file location, press Enter to accept the default:

```text
/home/ubuntu/.ssh/id_rsa
```

When prompted for a passphrase, press Enter without typing anything.

When prompted to confirm the passphrase, press Enter again.

The generated files are:

- `~/.ssh/id_rsa` — private key.
- `~/.ssh/id_rsa.pub` — public key.

### 3.2 Set SSH directory and key permissions

Run on `nn`:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```

Verify the generated files:

```bash
ls -l ~/.ssh/id_rsa*
```

### 3.3 Copy the public key to the other nodes

From `nn`, run the following commands:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@snn
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@jt
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@dn1
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@dn2
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@dn3
```

These commands install the NameNode's public key in the remote user's `~/.ssh/authorized_keys` file.

You may be asked to authenticate using the existing access method the first time you connect to each node.

**Important:** Only the public key (`id_rsa.pub`) is copied. Never copy the private key (`id_rsa`) to the other nodes.

### 3.4 Configure SSH access to the NameNode itself

To include `nn` in the DSH machine list, verify that local SSH works:

```bash
ssh nn hostname
```

If `nn` resolves to the NameNode's IP address and local SSH authentication is configured correctly, the command should return:

```text
nn
```

For a local SSH connection that requires public-key authentication, ensure the NameNode's public key is also present in the `ubuntu` user's `~/.ssh/authorized_keys` file on `nn`. If necessary, run:

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

This command appends the public key to the authorization list on `nn`. Avoid repeatedly appending the same key if it is already present.

### 3.5 Test passwordless SSH

From `nn`, run:

```bash
ssh snn hostname
ssh jt hostname
ssh dn1 hostname
ssh dn2 hostname
ssh dn3 hostname
```

Each command should return the corresponding hostname without prompting for the remote account password.

Repeat the test for any node on which local SSH was configured.

---

## Step 4: Configure `.profile`

The `.profile` file can be used to initialize the user's shell environment.

On `nn`, open:

```bash
vi ~/.profile
```

For this no-passphrase SSH key setup, no `ssh-agent` configuration is required. The private key can be used directly by SSH.

If the file already contains other environment settings, leave them unchanged.

Reload the profile:

```bash
source ~/.profile
```

Copy the profile to the remaining nodes if the same profile settings are needed there:

```bash
scp ~/.profile snn:~/
scp ~/.profile jt:~/
scp ~/.profile dn1:~/
scp ~/.profile dn2:~/
scp ~/.profile dn3:~/
```

The SSH private key is not copied by these commands.

---

## Step 5: Configure SSH Host-Key Checking

SSH host-key checking verifies the identity of remote servers.

The recommended approach is to verify each server's host key and keep host-key checking enabled.

Connect to each node from `nn`:

```bash
ssh snn hostname
ssh jt hostname
ssh dn1 hostname
ssh dn2 hostname
ssh dn3 hostname
```

On the first connection, SSH may ask you to confirm the host key. Verify the fingerprint through a trusted channel before accepting it.

Avoid setting `StrictHostKeyChecking no` globally because it weakens protection against server impersonation.

---

## Step 6: Install and Configure DSH

Distributed Shell (DSH) allows commands to be executed on multiple machines from one control node.

Install DSH on `nn`:

```bash
sudo apt update
sudo apt install dsh -y
```

Open the machine list:

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

Configure DSH to use SSH:

```bash
sudo vi /etc/dsh/dsh.conf
```

Set:

```text
remoteshell = ssh
```

Save the file.

### Test DSH

Run:

```bash
dsh -a hostname
```

DSH should execute the hostname command on each configured machine.

---

## Step 7: Install Java 8 on All Nodes

Run the following commands from `nn`.

Update package lists:

```bash
dsh -a sudo apt update
```

Install OpenJDK 8:

```bash
dsh -a sudo apt install openjdk-8-jdk -y
```

Verify Java:

```bash
dsh -a java -version
```

Ensure Java 8 is available on every node.

---

## Step 8: Download and Install Hadoop 1.2.1

Download Hadoop 1.2.1 on all six nodes:

```bash
dsh -a wget https://archive.apache.org/dist/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz
```

Extract the archive:

```bash
dsh -a tar -xzvf hadoop-1.2.1.tar.gz
```

Move the installation into `/usr/local/hadoop`:

```bash
dsh -a sudo mv hadoop-1.2.1 /usr/local/hadoop
```

Set ownership:

```bash
dsh -a sudo chown -R ubuntu:ubuntu /usr/local/hadoop
```

Verify the installation:

```bash
dsh -a ls -lrth /usr/local/hadoop
dsh -a ls -lrth /usr/local/hadoop/conf
```

These commands assume the archive has not already been extracted and the destination directory does not already contain an installation.

---

## Step 9: Configure `.bashrc`

On `nn`, open:

```bash
vi ~/.bashrc
```

Add the following environment variables:

```bash
export HADOOP_PREFIX=/usr/local/hadoop
export HADOOP_HOME=/usr/local/hadoop
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$PATH:$HADOOP_PREFIX/bin:$JAVA_HOME/bin
```

Copy `.bashrc` to the other nodes:

```bash
scp ~/.bashrc snn:~/
scp ~/.bashrc jt:~/
scp ~/.bashrc dn1:~/
scp ~/.bashrc dn2:~/
scp ~/.bashrc dn3:~/
```

Apply the settings on `nn`:

```bash
source ~/.bashrc
```

Verify:

```bash
echo "$HADOOP_HOME"
echo "$JAVA_HOME"
java -version
hadoop version
```

Ensure the configured Java path exists on all nodes.

---

## Step 10: Configure `hadoop-env.sh`

On `nn`, open:

```bash
vi /usr/local/hadoop/conf/hadoop-env.sh
```

Set:

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_OPTS="-Djava.net.preferIPv4Stack=true"
```

Save the file. It will be distributed to the other nodes in Step 15.

---

## Step 11: Configure `core-site.xml`

Open:

```bash
vi /usr/local/hadoop/conf/core-site.xml
```

Use the following complete configuration:

```xml
<?xml version="1.0"?>
<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://nn:9000</value>
    </property>

    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/hadoop/tmp</value>
    </property>
</configuration>
```

Save the file.

- `fs.default.name` specifies the HDFS NameNode URI.
- `hadoop.tmp.dir` specifies the base directory for Hadoop temporary files.

---

## Step 12: Configure `hdfs-site.xml`

Open:

```bash
vi /usr/local/hadoop/conf/hdfs-site.xml
```

Use:

```xml
<?xml version="1.0"?>
<configuration>
    <property>
        <name>dfs.name.dir</name>
        <value>/usr/local/hadoop/hdfs/name</value>
    </property>

    <property>
        <name>dfs.data.dir</name>
        <value>/usr/local/hadoop/hdfs/data</value>
    </property>

    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```

Save the file.

- `dfs.name.dir` stores NameNode filesystem metadata.
- `dfs.data.dir` specifies the DataNode storage directory.
- `dfs.replication` sets the target replication factor to three.

---

## Step 13: Configure `mapred-site.xml`

Open:

```bash
vi /usr/local/hadoop/conf/mapred-site.xml
```

Use:

```xml
<?xml version="1.0"?>
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>jt:9001</value>
    </property>
</configuration>
```

Save the file.

This configuration specifies the JobTracker hostname and port for classic Hadoop 1.x MapReduce.

---

## Step 14: Configure `masters` and `slaves`

On `nn`, configure the `masters` file:

```bash
vi /usr/local/hadoop/conf/masters
```

Add:

```text
snn
```

Configure the `slaves` file:

```bash
vi /usr/local/hadoop/conf/slaves
```

Add:

```text
dn1
dn2
dn3
```

Save both files.

These files are used by the classic Hadoop 1.x startup scripts.

---

## Step 15: Copy Configuration Files to All Nodes

Run the following commands from `nn`.

### Copy to `snn`

```bash
scp /usr/local/hadoop/conf/hadoop-env.sh snn:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/core-site.xml snn:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/hdfs-site.xml snn:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/mapred-site.xml snn:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/masters snn:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/slaves snn:/usr/local/hadoop/conf/
```

### Copy to `jt`

```bash
scp /usr/local/hadoop/conf/hadoop-env.sh jt:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/core-site.xml jt:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/hdfs-site.xml jt:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/mapred-site.xml jt:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/masters jt:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/slaves jt:/usr/local/hadoop/conf/
```

### Copy to `dn1`

```bash
scp /usr/local/hadoop/conf/hadoop-env.sh dn1:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/core-site.xml dn1:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/hdfs-site.xml dn1:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/mapred-site.xml dn1:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/masters dn1:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/slaves dn1:/usr/local/hadoop/conf/
```

### Copy to `dn2`

```bash
scp /usr/local/hadoop/conf/hadoop-env.sh dn2:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/core-site.xml dn2:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/hdfs-site.xml dn2:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/mapred-site.xml dn2:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/masters dn2:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/slaves dn2:/usr/local/hadoop/conf/
```

### Copy to `dn3`

```bash
scp /usr/local/hadoop/conf/hadoop-env.sh dn3:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/core-site.xml dn3:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/hdfs-site.xml dn3:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/mapred-site.xml dn3:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/masters dn3:/usr/local/hadoop/conf/
scp /usr/local/hadoop/conf/slaves dn3:/usr/local/hadoop/conf/
```

Verify the files:

```bash
dsh -a ls -l /usr/local/hadoop/conf/
```

---

## Step 16: Create Hadoop Directories on All Nodes

Run from `nn`:

```bash
dsh -a sudo mkdir -p /usr/local/hadoop/tmp
dsh -a sudo mkdir -p /usr/local/hadoop/hdfs/name
dsh -a sudo mkdir -p /usr/local/hadoop/hdfs/data
```

Set ownership:

```bash
dsh -a sudo chown -R ubuntu:ubuntu /usr/local/hadoop
```

Verify the directories:

```bash
dsh -a ls -ld /usr/local/hadoop/tmp
dsh -a ls -ld /usr/local/hadoop/hdfs/name
dsh -a ls -ld /usr/local/hadoop/hdfs/data
```

The Hadoop user must have the necessary write permissions.

---

## Step 17: Format the NameNode

**Run this step only when initializing a new HDFS filesystem.**

Connect to the NameNode:

```bash
ssh nn
```

Format the filesystem:

```bash
hadoop namenode -format
```

Review the output and confirm that the formatting operation completes successfully.

Do not reformat an existing NameNode containing data unless you deliberately intend to initialize a new filesystem and understand the consequences.

---

## Step 18: Start HDFS

Run on `nn`:

```bash
start-dfs.sh
```

Verify the running processes:

```bash
dsh -a jps
```

Expected HDFS processes:

| Node | Expected Process |
|---|---|
| `nn` | `NameNode` |
| `snn` | `SecondaryNameNode` |
| `dn1` | `DataNode` |
| `dn2` | `DataNode` |
| `dn3` | `DataNode` |

---

## Step 19: Start MapReduce

Connect to the JobTracker:

```bash
ssh jt
```

Start MapReduce:

```bash
start-mapred.sh
```

Verify the processes from `nn`:

```bash
dsh -a jps
```

Expected MapReduce processes:

| Node | Expected Process |
|---|---|
| `jt` | `JobTracker` |
| `dn1` | `TaskTracker` |
| `dn2` | `TaskTracker` |
| `dn3` | `TaskTracker` |

---

## Final Verification

### Check hostname resolution

```bash
getent hosts nn snn jt dn1 dn2 dn3
```

### Check passwordless SSH through DSH

```bash
dsh -a hostname
```

### Check Java

```bash
dsh -a java -version
```

### Check Hadoop installation

```bash
dsh -a ls -ld /usr/local/hadoop
```

### Check running services

```bash
dsh -a jps
```

### Check HDFS status

```bash
hadoop dfsadmin -report
```

Confirm that all three DataNodes are reported and that HDFS is operating as expected.

## Troubleshooting

### SSH authentication fails

- Verify that `id_rsa.pub` was installed in the remote user's `~/.ssh/authorized_keys`.
- Check the ownership and permissions of the SSH directory and authorization file.
- Confirm that the SSH server is running and accessible.
- Check hostname resolution and network connectivity.
- Ensure that the user running DSH has access to the private key.

### Java is not found

- Run `java -version`.
- Verify `JAVA_HOME` in `.bashrc` and `hadoop-env.sh`.
- Confirm that the configured Java path exists on all nodes.

### Hadoop services fail to start

- Check the XML configuration files.
- Verify hostnames, ports, and directory permissions.
- Inspect Hadoop logs for Java, SSH, storage, and networking errors.
- Ensure that all nodes have the correct configuration files.

### DataNodes are missing

- Check the `slaves` file.
- Verify that the DataNode processes are running.
- Inspect DataNode logs.
- Check HDFS storage directory permissions and NameNode connectivity.

## Security Notes

- The RSA private key (`~/.ssh/id_rsa`) must remain private and must never be uploaded to GitHub.
- The generated key has no passphrase, so anyone who obtains the private key may be able to authenticate as the corresponding user. Protect the machine and file permissions carefully.
- Keep SSH host-key verification enabled wherever possible.
- Hadoop 1.2.1 is a legacy release. Use an isolated lab environment and avoid exposing Hadoop services to untrusted networks.
- Never format an existing NameNode or delete HDFS storage directories without a deliberate recovery plan.
