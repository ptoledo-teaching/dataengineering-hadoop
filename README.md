# Deploying a Hadoop Cluster on AWS Academy

This repository contains a guided procedure for manually deploying a Hadoop cluster for teaching purposes inside AWS Academy. The idea is that students build and configure the cluster from scratch, understanding each infrastructure decision and each configuration file instead of delegating the process to automation scripts.

## Repository Structure

- `config/hadoop/`: base Hadoop configuration files adjusted for educational purposes. They are intended to be copied and edited manually on the Master node
- `scripts/hadoop-env.sh`: Hadoop-specific shell snippet for loading Java 11 and Hadoop paths in the current session, without affecting the system-wide Java configuration.
- `scripts/get_usage.sh`: optional utility to observe CPU and memory usage once the cluster is already running

When refering to the **AWS Academy Console** it corresponds to the regular AWS Management Console, but accessed through the AWS Academy portal. The services and features are the same as in the regular console, but the available resources will be provided and limited by the AWS Academy subscription.

## Initial AWS Preparation

### Create the EC2 key pair

In the AWS Academy console go to **EC2** and then:

- Open **Network & Security** → **Key Pairs**
- Click **Create key pair**
- Name the key `cluster-key`
- Use an **RSA** key
- Use the **.pem** format
- Download the key to your computer

If you want to keep your local material organized, you may store the key inside `.ssh/` for easy access from the terminal. The important thing is to remember where you stored it, because you will need it to connect to the Master machine later.

> ⚠️ Warning: You cannot download the same key pair again. If you lose the `.pem` file, you will not be able to connect to the machines that use that key pair

### Create the security group

You must create a security group that allows:

- `SSH` on port `22` from `Anywhere-IPv4`.
- `Custom TCP` on port `8088` from `Anywhere-IPv4`.
- `Custom TCP` on port `9870` from `Anywhere-IPv4`.
- `Custom TCP` on the range `0-65535` from the same security group.

To create the security group:

- Open **Network & Security** → **Security Groups**
- Click **Create security group**
- Name the group `hadoop-cluster-sg` with a description like "Security group for the Hadoop cluster" and VPC to the default one provided by AWS
- In **Inbound rules** add the rules for SSH, port 8088, and port 9870
- In **Outbound rules** there should be a single rule allowing all traffic configured by default. You must leeave it as is
- Click **Create security group**
- After the process is complete, go again to **Security Groups**, select the new group, and click on the **Inbound rules** tab at the bottom. Click **Edit inbound rules** and add a new rule with:
  - Type: `Custom TCP`
  - Port range: `0-65535`
  - Source: `Custom` and then select the same security group from the dropdown
- Click **Save rules**

## Creating the Cluster Machines

### Create the Master machine

Go to **EC2** → **Instances**, and click **Launch instances**. Configure the instance with:

- Name: `Master`
- Operating system: `Ubuntu Server 24.04`
- Instance type: `t3.medium`
- Key pair: the key created in the previous step
- Network settings: Select existing security group and then select the sg created in the previous step
- Storage: `16 GB` `gp3`

In summary the number of instances must be 1, then click on **Launch instance** and then on **View all instances**. After the Master instance is state running proceed to the next step.

### Elastic IP for the Master

In AWS, each time a machine is stopped and started again, it may receive a different public IP. To be able to have a fixed public IP for the Master, you can associate an Elastic IP to it. This has an additional cost that is covered by the AWS Academy subscription. To associate the elastic IP:

- Open **Network & Security** → **Elastic IPs**
- Click **Allocate Elastic IP address**
- Use the default options and click **Allocate**
- After the Elastic IP is allocated, select it and click **Actions** → **Associate Elastic IP address**
- In the association form, select the Master instance (it is not neccesary to select a specific private IP) 
- Click **Associate**
- Take note of the Elastic IP, which is the new public IP of the Master. You will use it to connect to the Master and to access the web consoles later

### Testing the SSH connection to the Master

Before proceeding with the cluster configuration, test that you can connect to the Master through SSH using the `.pem` key. In the console, first go to the folder that contains the `.pem` file, then run:

On Linux or macOS:

```bash
chmod 400 cluster-key.pem
ssh -i cluster-key.pem ubuntu@<master_public_ip>
```
On Windows you may need to adjust permissions before connecting. First get your username with `whoami`, then run:

```powershell
icacls cluster-key.pem /reset
icacls cluster-key.pem /grant <username>:F
```

Then you can connect with:

```bash
ssh -i cluster-key.pem ubuntu@<master_public_ip>
```

The first time you connect, you must accept the server fingerprint. If the connection is successful, you should see a welcome message from the Ubuntu server and a command prompt. Once you confirm that the SSH connection to the Master is working, you can disconnect with `exit` and proceed to the next steps.

### Create the Worker machines

Create the Worker instances using:

- Name: `Worker`
- Operating system: `Ubuntu Server 24.04`
- Instance type: `t3.small`
- Key pair: the same key used for the Master
- Network settings: the same cluster security group
- Storage: `8 GB` `gp3`

This time, in the summary, set the number of instances to `4` and then click on **Launch instance**. After the Worker instances are state running proceed to the next step.

## Configuring the Cluster

### Base Installation on the Master

Connect to the Master, update packages and install Java 11:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-11-jdk -y
```

Download Hadoop 3.4.3 from the official site or from the Apache archive:

```bash
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.4.3/hadoop-3.4.3.tar.gz
tar xzf hadoop-3.4.3.tar.gz
```

This will leave Hadoop in:

```bash
~/hadoop-3.4.3
```

### Make This Repository Available on the Master

Clone the current repository on the Master machine via:

```bash
git clone https://github.com/ptoledo-teaching/dataengineering-hadoop.git
```

### Configure a Hadoop-Specific Environment on the Master

Hadoop requires Java 11, but you should avoid making Java 11 the machine-wide default. Instead, keep the Java 11 selection scoped to Hadoop itself and to an optional shell snippet used only when working with Hadoop.

The repository includes a support file in `scripts/hadoop-env.sh`. It is intended to be sourced manually in a shell session.

To have a shortcut for loading the Hadoop environment, you can create a symbolic link to that file in your home folder:

```bash
ln -s ~/dataengineering-hadoop/scripts/hadoop-env.sh ~/hadoop-env.sh
```

Then, whenever you want to work with Hadoop in that shell session, load it with:

```bash
source ~/hadoop-env.sh
```

### Configure SSH on the Master

Hadoop uses SSH for communication from the Master node to itself and to the Workers. On the Master you must:

1. Generate an SSH key for the Master itself
  - Use the default options and do not set a passphrase, so that the key can be used non-interactively
2. Install that key into `authorized_keys`
3. Confirm that the Master can connect to itself
  - As in the previous ssh connection test, the first time you connect, you must accept the server fingerprint. If the connection is successful, you should see a welcome message from the Ubuntu server and a command prompt. You can disconnect with `exit` and proceed to the next steps

Run:

```bash
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
ssh localhost
```

### Prepare the Hadoop Configuration Files

The configuration files are already organized in `config/hadoop/`. The idea is to use them as a base and then adapt them to the specific cluster.

### `core-site.xml`

The file is located at `config/hadoop/core-site.xml` and contains the `{master_ip_private}` placeholder. You must replace it with the real private IP of the Master after copying it into Hadoop etc folder:

```bash
cp dataengineering-hadoop/config/hadoop/core-site.xml ~/hadoop-3.4.3/etc/hadoop/core-site.xml
```

### `yarn-site.xml`

It also contains the `{master_ip_private}` placeholder and must be replaced manually after copying it into Hadoop etc folder:

```bash
cp dataengineering-hadoop/config/hadoop/yarn-site.xml ~/hadoop-3.4.3/etc/hadoop/yarn-site.xml
```

### `hdfs-site.xml` and `mapred-site.xml`

These files can be copied directly:

```bash
cp dataengineering-hadoop/config/hadoop/hdfs-site.xml ~/hadoop-3.4.3/etc/hadoop/hdfs-site.xml
cp dataengineering-hadoop/config/hadoop/mapred-site.xml ~/hadoop-3.4.3/etc/hadoop/mapred-site.xml
```

### `hadoop-env.sh`

Besides the XML files, Hadoop also uses `etc/hadoop/hadoop-env.sh` when starting daemons through scripts such as `start-dfs.sh` and `start-yarn.sh`. That file must define `JAVA_HOME`, to do this, add the following line at the end of `hadoop-3.4.3/etc/hadoop/hadoop-env.sh`:

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

### `workers` file

Edit the file named `~/hadoop-3.4.3/etc/hadoop/workers` with the private IPs of the Worker machines, one per line. You can get the private IPs from the AWS Academy console, in the description of each Worker instance.

Example:

```text
172.31.0.11
172.31.0.12
172.31.0.13
172.31.0.14
```

> ⚠️ Important: You should delete the default `localhost` entry from the `workers` file, otherwise the Master will try to connect to itself as a Worker and that may cause issues

## Workers preparation

### Connecting to each worker and configuring SSH connectivity

The master must be able to connect to each Worker through SSH without a password, and each Worker must be able to connect back to the Master. This is required for Hadoop to work properly.

To start, take the `cluster-key.pem` file that you downloaded at the beginning and load it in the `.ssh` folder in the Master machine. While using the cluster key should be enough, we will use the `id_rsa.pub` public key generated on the Master as the main key for SSH connectivity between the Master and the Workers, so the cluster key is only needed for the initial connection to the Master, but not for the Master-Worker communication.

To install the Master public key on each Worker, you can add it to the `authorized_keys` file on each Worker. First, print the Master public key with:

```bash
cat ~/.ssh/id_rsa.pub
```

Then copy the output and paste it at the end of `~/.ssh/authorized_keys` on each Worker. This will allow the Master to connect to each Worker without a password.

In the same way, you must generate an SSH key on each Worker, install it so it can be used to connect to itself and install its public key into the Master's `authorized_keys` file, so that each Worker can connect back to the Master without a password. This is required for Hadoop to work properly. To do this, inside each Worker, run:

```bash
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_rsa.pub
```

Using the output of the second command, add the Worker's public key to the Master's `authorized_keys` file. This will allow each Worker to connect back to the Master without a password.

Once you have done this, you can delete the `cluster-key.pem` file from the Master, as it is no longer needed for the Master-Worker communication.

### Configuring the Workers

Each Worker must be configured manually, and the `hadoop-3.4.3` folder must be copied from the Master. To do this, in the Master machine we will tar and compress the configured hadoop folder for easier deployment. Since `~/hadoop-3.4.3/etc/hadoop/hadoop-env.sh` is inside that folder, the `JAVA_HOME` setting will travel together with the Hadoop installation to every Worker. To do this run:

```bash
tar czf hadoop-3.4.3-deploy.tar.gz hadoop-3.4.3
```

For each Worker machine:

1. Connect through SSH
2. Update packages and install Java with
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-11-jdk -y
```
4. Copy the `hadoop-3.4.3-deploy.tar.gz` file from the Master with `scp` and extract it there with:
```bash
scp ubuntu@<master_private_ip>:~/hadoop-3.4.3-deploy.tar.gz ~/
tar xzf hadoop-3.4.3-deploy.tar.gz
rm hadoop-3.4.3-deploy.tar.gz
```

### Reboot the Machines

Once the configuration has been applied on the Master and Workers, reboot the machines to ensure package updates and background services start from a clean state. You can do this by connecting to each machine and running:

```bash
sudo reboot
```

Or through the AWS Academy console, by selecting each instance and using **Instance state** → **Reboot instance**.

## Initialize the Cluster

To run any Hadoop command, you must be connected to the Master and have the Hadoop environment loaded with `source ~/hadoop-env.sh`.

### Initializing HDFS and YARN

From the Master, the first time HDFS is used the NameNode must be formatted:

```bash
hdfs namenode -format
```

Then start HDFS:

```bash
start-dfs.sh
```

Then start YARN:

```bash
start-yarn.sh
```

You can verify local processes with:

```bash
jps
```

### Check HDFS status

First check the HDFS report:

```bash
hdfs dfsadmin -report
```

Then check the file and block status of the root folder:

```bash
hdfs fsck / -files -blocks
```

### Web consoles

From your computer you can open:

- `http://<master_public_ip>:9870` for the NameNode
- `http://<master_public_ip>:8088` for the ResourceManager

### Run a wordcount test

Create the data folder in HDFS:

```bash
hdfs dfs -mkdir /data
```

The Episode IV test file is available in `slides/code/T01/sw-script-e04.txt`. You may download it directly on the Master or copy it from your computer.

If you want to download it from the Internet:

```bash
wget -O sw-script-e04.txt https://raw.githubusercontent.com/gastonstat/StarWars/master/Text_files/SW_EpisodeIV.txt
```

Upload the file to HDFS:

```bash
hdfs dfs -put sw-script-e04.txt /data/
```

Run the wordcount example:

```bash
hadoop jar ~/hadoop-3.4.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.3.jar \
wordcount /data/sw-script-e04.txt /data/sw-script-e04.wordcount
```

Review the output:

```bash
hadoop fs -cat /data/sw-script-e04.wordcount/part-r-00000 | more
```

## Final notes

Deployment and configuration must be performed manually, but the repository leaves one utility for the post-installation stage.

### `scripts/get_usage.sh`

Shows CPU and memory usage for the Master and the Workers. It requires SSH connectivity between nodes to be already working.

Usage:

```bash
./scripts/get_usage.sh
```

By default, it samples the usage every 5 seconds. To change the interval:

```bash
./scripts/get_usage.sh 10
```

### Shut Down the Cluster

To stop Hadoop services from the Master:

```bash
stop-yarn.sh
stop-dfs.sh
```

### About the AWS Academy Session

Each time the AWS Academy session it starts, it starts a timer of 4 hours. When the timer ends, all the resources are stopped automatically. You can check the remaining time in the AWS Academy Canvas Website. If you need more time, you can click on **Start Lab** again to reset the timer. If the time has already passed, you will need to click on **Start Lab** to start a new session.

When a AWS Academy Session starts, it automatically starts all the EC2 machines, in the same way, when the session ends, it stops all the EC2 machines preventing the usage of resources when they are not needed. 
