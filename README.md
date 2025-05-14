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
   openshift-install create install-config
   ```
   - Modify `install-config.yaml` to match AWS infrastructure.

4. **Deploy Bootstrap Node**
   ```sh
   openshift-install create manifests
   openshift-install create ignition-configs
   aws ec2 run-instances --image-id <AMI_ID> --count 1 --instance-type m5.large --key-name <KEY_NAME> --security-group-ids <SG_ID> --subnet-id <SUBNET_ID>
   ```

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


# **If Installation Failed then  Check the logs**

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


===================================


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

## **The End**


