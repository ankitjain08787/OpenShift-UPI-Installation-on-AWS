# OpenShift-UPI-Installation-on-AWS
Red Hat OpenShift User-Provisioned-Installation on AWS

[Documentation Author: Ankit Jain](https://www.linkedin.com/in/ankitkkjain/)

Setting up a **Red Hat OpenShift cluster** on **AWS** using the **User-Provisioned Infrastructure (UPI)** method requires manual creation of infrastructure components. Here’s a **brief end-to-end guide**:

### **Prerequisites**
- **AWS Account** with necessary permissions.
- **AWS CLI** installed and configured.
- **OpenShift Installer** downloaded.
- **Terraform** (optional) for automation.
- **Networking setup** (VPC, subnets, security groups).
- **IAM roles and policies** for OpenShift components.
- **Route 53** on AWS with Public domain

### **Installation Steps**
1. **Prepare AWS Infrastructure**
   ```sh
   aws configure
   aws ec2 create-vpc --cidr-block 10.0.0.0/16
   aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.1.0/24
   aws ec2 create-security-group --group-name openshift-sg --description "OpenShift Security Group"
   ```
   - Create **VPC**, **subnets**, and **security groups**.
   - Set up **IAM roles** for OpenShift components.

2. **Download and Configure OpenShift Installer**
   ```sh
   curl -O https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/openshift-install-linux.tar.gz
   tar -xvf openshift-install-linux.tar.gz
   mv openshift-install /usr/local/bin/
   ```

3. **Generate Installation Configuration**
   ```sh
   openshift-install create install-config --dir=<Directory>
   ```
   - Modify `install-config.yaml` to match AWS infrastructure.

4. **Deploy Bootstrap Node**
   ```sh
   openshift-install create manifests --dir=<Directory>
   openshift-install create ignition-configs --dir=<Directory>
   aws ec2 run-instances --image-id <AMI_ID> --count 1 --instance-type m5.large --key-name <KEY_NAME> --security-group-ids <SG_ID> --subnet-id <SUBNET_ID>
   ```
[Refer for Image](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/installing_on_aws/user-provisioned-infrastructure#installation-aws-ami-stream-metadata_installing-aws-user-infra)

5. **Deploy Control Plane and Worker Nodes**
   ```sh
   aws ec2 run-instances --image-id <AMI_ID> --count 3 --instance-type m5.large --key-name <KEY_NAME> --security-group-ids <SG_ID> --subnet-id <SUBNET_ID>
   ```

6. **Complete Installation**
   ```sh
   openshift-install wait-for bootstrap-complete
   openshift-install wait-for install-complete
   ```

7. **Verify Cluster**
   ```sh
   oc login --server=https://api.<cluster_name>.<domain>:6443 --username=kubeadmin --password=<password>
   oc get nodes
   ```

For **detailed documentation**, refer to [Red Hat OpenShift Docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/installing_on_aws/installing-aws-user-infra) and [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/red-hat-openshift-on-aws-implementation/installation-options.html).

## **Installation Steps Completed**

##
##

## **If Installation Failed then  Check the logs**

[root@ip-172-31-3-148 ~]# openshift-install gather bootstrap --help
Gather debugging data for a failing-to-bootstrap control plane

Usage:
  openshift-install gather bootstrap [flags]

Flags:
      --bootstrap string     Hostname or IP of the bootstrap host
  -h, --help                 help for bootstrap
      --key stringArray      Path to SSH private keys that should be used for authentication. If no key was provided, SSH private keys from user's environment will be used
      --master stringArray   Hostnames or IPs of all control plane hosts
      --skipAnalysis         Skip analysis of the gathered data

Global Flags:
      --dir string         assets directory (default ".")
      --log-level string   log level (e.g. "debug | info | warn | error") (default "info")

##

## **IPv4 VPC and Subnet CIDR Blocks**
In Amazon AWS, when creating a VPC (Virtual Private Cloud), commonly used IPv4 CIDR blocks follow the private address space defined by RFC 1918. Here are the most frequently used options:

**Commonly Used IPv4 VPC CIDR Blocks in AWS:**
1. **10.0.0.0/16** – Often used because it provides **65,536** available IP addresses.
2. **192.168.0.0/16** – Common in smaller setups; also allows for **65,536** IP addresses.
3. **172.16.0.0/16** – Less frequently used but still an option within the **RFC 1918** range.

When creating **subnets**, their **CIDR block** must be **within the VPC CIDR block**. For example, if your **VPC CIDR block** is `10.0.0.0/16`, you might use these **subnet CIDR blocks**:
- **10.0.1.0/24** (Allows **256** IPs)
- **10.0.2.0/24** (Allows **256** IPs)
- **10.0.3.0/24** (Allows **256** IPs)

Node: Each subnet must stay within the VPC's address space, and AWS reserves **5 IP addresses** in each subnet: the first IP, last IP, and three others for AWS services.
##
##

### **Should the Bootstrap Node Use a Different Subnet?**
**1️⃣ Same Subnet as the Master (`10.0.1.0/24`)**  
✅ If the bootstrap node is tightly integrated with the master and worker nodes, keeping it in the **same subnet** (`10.0.1.0/24`) can simplify networking.  
✅ Reduces complexity in routing and security configurations.  
✅ Easier for direct communication with the master node, reducing latency.  

**2️⃣ Separate Subnet for Bootstrap (e.g., `10.0.5.0/24`)**  
✅ If you want **network isolation** between the bootstrap and master/workers, a separate subnet can add security.  
✅ Better control over Security Groups and Network ACLs.  
✅ Useful if bootstrap runs temporary provisioning processes and doesn’t need long-term persistence.  

### **Best Approach?**
Most deployments **keep the bootstrap node in the same subnet** as the master (`10.0.1.0/24`) for simplicity.  
However, if you have a security-first approach or multi-AZ architecture, a separate subnet may be beneficial.  

**Security Rules for Bootstrap Node**
Since your **master node** is in `10.0.1.0/24`, your bootstrap node will likely be in the same subnet. Here’s how to set up **Security Group rules**:

#### **1️⃣ Inbound Rules (Incoming Traffic)**
| Protocol | Port | Source | Purpose |
|----------|------|--------|---------|
| SSH (Secure Shell) | 22 | Your Admin IP | Only allow your IP for secure access. |
| Kubernetes API | 6443 | Master Nodes | Enables communication with the master node. |
| Internal Communication | Custom | Master & Worker Subnets | Any necessary ports for internal cluster setup. |

#### **2️⃣ Outbound Rules (Outgoing Traffic)**
| Destination | Protocol | Purpose |
|-------------|----------|---------|
| Internet Gateway / NAT Gateway | HTTPS (443) | Allows package downloads & updates. |
| Master Node (`10.0.1.0/24`) | TCP/UDP | Communication for node initialization. |
##
##

It looks like your **security group** and **subnet** belong to different **VPCs**, which is causing the error. Here’s how you can troubleshoot and resolve it:

### **Steps to Fix**
1. **Check the VPC of the Subnet and Security Group**
   ```sh
   aws ec2 describe-subnets --subnet-ids subnet-0d656ccfceb0b1715 --query 'Subnets[*].VpcId' --output text
   aws ec2 describe-security-groups --group-ids sg-07185c8a682bd5b0c --query 'SecurityGroups[*].VpcId' --output text
   ```
   - Ensure both have the same **VPC ID**.

2. **Modify the Security Group to Match the Subnet's VPC**
   - If the **security group** belongs to a different **VPC**, create a new one in the correct **VPC**:
   ```sh
   aws ec2 create-security-group --group-name new-sg --description "New OpenShift SG" --vpc-id <CORRECT_VPC_ID>
   ```
   - Use this new **security group** when launching instances.

3. **Verify Network Configuration**
   - Ensure the subnet and security group are correctly associated with the intended VPC.

4. **Retry Instance Creation**
   ```sh
   aws ec2 run-instances --image-id ami-0c2fcce436b950ba6 --count 1 --instance-type m5.large --key-name k8snodes --security-group-ids <NEW_SG_ID> --subnet-id subnet-0d656ccfceb0b1715
   ```
##

Several Modification in YAML files inside the `manifests/` folder after 'openshift-install create install-config' command to adjust **networking**, **machine settings**, and **security policies**.

### **Key Configuration File Modifications**
Here are the main changes you might need to make:

#### **1️⃣ `install-config.yaml` (Before manifest creation)**
Before running `openshift-install create manifests`, edit `install-config.yaml`:
- Set **Cluster Networking**:
  ```yaml
  networking:
    networkType: OpenShiftSDN
    machineNetwork:
      - cidr: "10.0.0.0/16"
    clusterNetwork:
      - cidr: "10.128.0.0/14"
        hostPrefix: 23
    serviceNetwork:
      - "172.30.0.0/16"
  ```
- Define **Machine Types**:
  ```yaml
  compute:
    - name: worker
      replicas: 3
      platform:
        aws:
          type: "m5.large"
  ```
- Adjust **Platform-specific settings** (for AWS, Azure, etc.).

#### **2️⃣ `cluster-scheduler-02-config.yaml`**
Modify this file if you want **dedicated master nodes** (preventing workload scheduling on masters):
```yaml
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  name: cluster
spec:
  mastersSchedulable: false
```

#### **3️⃣ `cloud-provider-config.yaml` (for Cloud integrations)**
For AWS, configure cloud provider settings:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloud-provider-config
  namespace: openshift-config
data:
  config: |
    [Global]
    Zone = us-east-1a
```

#### **4️⃣ `machine-config.yaml` (for node settings)**
Define node configurations:
```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: worker-config
spec:
  config:
    ignition:
      version: 3.2.0
  osImageURL: "registry.openshift.com/rhcos:latest"
```

### **Best Practices**
✅ Ensure **CIDR blocks** match your planned VPC & subnets.  
✅ Keep master nodes **unschedulable** for workload isolation.  
✅ Customize instance sizes (`m5.large` for workers, `m5.xlarge` for masters).  
✅ Enable **proper IAM roles & policies** for cloud integrations.  

##
The subnet plays a crucial role in defining which nodes are masters and which are workers in the install-config.yaml file. Here’s how it works:

1. Subnet Differentiation
Each node (master or worker) is assigned a subnet in AWS.

The subnet determines the network range and accessibility of the node.

OpenShift uses this information to group nodes accordingly.

2. Master vs. Worker Subnet
Master Nodes are typically placed in a private subnet for security.

Worker Nodes can be in a public or private subnet, depending on workload requirements.

The subnet ID in install-config.yaml helps OpenShift identify and assign roles.

3. Example Configuration
Here’s how you define subnets in install-config.yaml:
```
yaml
platform:
  aws:
    region: us-east-1
compute:
- name: worker
  replicas: 2
  platform:
    aws:
      type: m5.large
      subnet: subnet-12345678
controlPlane:
- name: master
  replicas: 3
  platform:
    aws:
      type: m5.xlarge
      subnet: subnet-87654321
```
The subnet ID (subnet-12345678 for workers, subnet-87654321 for masters) determines where each node is deployed.

OpenShift automatically assigns roles based on these subnet definitions.

4. Validation
To verify subnet assignments, use:
```
sh
aws ec2 describe-subnets --query 'Subnets[*].[SubnetId, VpcId, AvailabilityZone]' --output table
```
This ensures that the correct subnets are being used for masters and workers.
##

##
To specify `bootstrap.ign` in an AWS EC2 instance, follow these steps:

### **1. Prepare Ignition Configuration**
Ensure you have the `bootstrap.ign` file generated using:
```sh
openshift-install create ignition-configs --dir=<Directory>
```
This file contains setup instructions for the bootstrap node.

### **2. Upload `bootstrap.ign` to an S3 Bucket (Optional)**
AWS requires an accessible location for the Ignition config. You can either:
- Store it in an S3 bucket and use an HTTPS link.
- Directly provide it as `user_data` during instance launch.

### **3. Launch AWS EC2 Bootstrap Node**
When creating the bootstrap node, specify `bootstrap.ign` in the **User Data** field. You can do this using the AWS console or CLI.

#### **Using AWS Management Console**
1. Go to EC2 → Launch Instance.
2. Select the appropriate AMI (e.g., RHCOS image).
3. Choose instance type and configure networking.
4. In **Advanced Details**, locate **User Data** and paste:
   ```
   #cloud-config
   user_data: file://bootstrap.ign
   ```

#### **Using AWS CLI**
Run the following command:
```sh
aws ec2 run-instances --image-id <AMI-ID> --count 1 \
--instance-type <Instance-Type> --key-name <Key-Pair> \
--security-groups <Security-Group> --subnet-id <Subnet-ID> \
--user-data file://bootstrap.ign
```
Replace `<AMI-ID>`, `<Instance-Type>`, and other parameters with actual values.

### **4. Verify Bootstrap Process**
After instance creation, monitor the bootstrap process using:
```sh
openshift-install wait-for bootstrap-complete --log-level=debug
```
Once the master nodes stabilize, the bootstrap node shuts down.

Would you like help verifying logs or troubleshooting bootstrap failures?

##

##
To inject an Ignition file stored in an **S3 bucket** directly into an AWS EC2 instance during creation, follow these steps:

### **1. Generate and Upload the Ignition File**
Make sure you have the Ignition configuration file:
```sh
openshift-install create ignition-configs --dir=<Directory>
```
Then, upload it to an S3 bucket:
```sh
aws s3 cp bootstrap.ign s3://<your-bucket-name>/bootstrap.ign --acl private
```
Ensure the bucket permissions allow only your EC2 instances to access the file.

### **2. Create an EC2 Instance with Ignition via User Data**
Use AWS CLI to fetch the Ignition file from S3 and apply it during EC2 instance creation:
```sh
aws ec2 run-instances --image-id <AMI-ID> --count 1 \
--instance-type <Instance-Type> --key-name <Key-Pair> \
--security-groups <Security-Group> --subnet-id <Subnet-ID> \
--iam-instance-profile Name=<IAM-Role> \
--user-data "Content-Type: multipart/mixed; boundary=\"IgnitionBoundary\"
MIME-Version: 1.0

--IgnitionBoundary
Content-Type: text/x-shellscript
Mime-Version: 1.0

#!/bin/bash
curl -o /var/lib/ignition/bootstrap.ign $(aws s3 presign s3://<your-bucket-name>/bootstrap.ign)
--IgnitionBoundary--"
```
Replace:
- `<AMI-ID>` with the appropriate **Red Hat CoreOS (RHCOS)** AMI.
- `<IAM-Role>` with an IAM profile that has **S3 read access**.
- `<your-bucket-name>` with the name of your S3 bucket.

### **3. Ensure IAM Role Has S3 Access**
Create an **IAM Role** with the following permissions:
```json
{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<your-bucket-name>/*"
}
```
Attach this role to the EC2 instance so it can access the Ignition file.

### **4. Verify Bootstrap Process**
After instance creation, check if the Ignition file was applied correctly:
```sh
journalctl -u ignition --no-pager -n 50
```
Also, verify the bootstrap progress:
```sh
openshift-install wait-for bootstrap-complete --log-level=debug
```

This ensures the bootstrap node initializes correctly using the Ignition file from S3.
##

## **The End**


