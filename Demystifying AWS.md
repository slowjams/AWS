## Demystifying AWS

# Animals4Life

BNE HQ:   `192.168.10.0/24`, `10.0.0.0/16` (AWS Pilot),
                             `172.31.0.0/16` (Azure Pilot)
London:   `192.168.15.0/24`
New York: `192.168.20.0/24`
Seattle:  `192.168.24.0/24`
Google:   `10.128.0.0/9`  (previous vendor's usage)

Design:

* Reserve at least networks per region per account for buffering
* 3 × US regions, 1 × Europe region, 1 × Australia region, 4 × AWS accounts: (3 + 1 + 1) × 2 × 4 = 40 ranges (ideally)
* 4 AZs: AZ-A, AZ-B, AZ-C, AZ-Future
* 4 Tiers: **Web**, **App**, **Db**, **Spare**

# VPC (Virtual Private Cloud)

```bash
# private address ranges
10.0.0.0     -   10.255.255.255  (CIDR block: 10.0.0.0/8)
172.16.0.0   -   172.31.255.255  (CIDR block: 172.16.0.0/12)
192.168.0.0  -   192.168.255.255 (CIDR block: 192.168.0.0/16)
```

```bash
# default VPC
172.31.0.0/16  # <-------------------------------------------------------------------------default VPC is not recommended to use, seel later in the course and come back edit the reason

# default ap-southeast-2 subnets   last two octets         each network/subnet's host range        each subnet maps to an AZ
172.31.0.0/20                     0000|0000,00000000       start 172.31.0.0  end 172.31.15.255          ap-southeast-2a
172.31.16.0/20                    0001|0000,00000000       start 172.31.16.0 end 172.31.31.255          ap-southeast-2b
172.31.32.0/20                    0010|0000,00000000       start 172.31.32.0 end 172.31.47.255          ap-southeast-2c
#... extend in the future

# default us-west-2 subnets    each subnet maps to an AZ
172.31.0.0/20                       us-west-2c  # not sure why 'c' map for lower addreeses
172.31.16.0/20                      us-west-2a
172.31.32.0/20                      us-west-2b
172.31.48.0/20                      us-west-2d
```

You can connect and peer two VPCs (e.g  connect `10.0.0.0/16` with `10.1.0.0/16`)

## VPC Facts:

* Minimum `28` (16 IPs), maximum `16` (65536 IPs)
* One default VPC (dVPC) per region - can be deleted & recreated. So a region in theory could have zero default VPC (if you delete it), but some AWS services assume default vpc will be present, so you should leave the default vpc active (do not delete or deactive it), but don't use default VPC for production.
* CIDR for dVPC is always `172.31.0.0/16` which is the same for all different regions, it can contain upto  **16** subnets/AZs for `/20` (like cutting an cake into multiple pieces)
* `/20` for every subnet
* Subnets assign serivces public ip address


# EC2

```bash
ssh -i "A4L.pem" ec2-user@ec2-18-234-163-215.compute-1.amazonaws.com
```


# S3

* Bucket name are **globally unique**
* An object can vary from 0 bytes to 5TB
* You can have only 100 (soft limit) buckets per account, and with support request, up to 1000 (hard limit) buckets per account. This is designed to not let you design a system such as "one user per bucket" as you are supposed to use prefix.



## CloudFormation

```yml
AWSTemplateFormatVersion: version date

Description:
  String

Metadata:
  template metadata

Parameters:
  set of parameters

Rules:
  set of rules

Mappings:
  set of mappings

Conditions:
  set of conditions

Transform:
  set of transforms

Resources:
  set of resources

Outputs:
  set of outputs
```

* if `AWSTemplateFormationVersion` is used then `Description` needs to be dicretly follow it


## IAM

It is not recommended to use access key for root users. You can create at most 2 access keys for an IAM user (rotating purpose).

* inline policy is for a single IAM user, so inline policy cannot be searched in the console UI (because it cannot be assigned to other IAM users)
* An IAM user can be a member of maximum **10** groups and there is a **5,000** IAM user limit for an account, and there is no limit on how many users can be in a single group, 
so you could have all 5,000 users in a single user group 
* There is a limit of **300** groups per account but it can be increased via support ticket
* There is no nesting in group (group within group) and groups (just a container for users only, that's it) are not a true identity so they can't be referenced as a principal in a policy


IAM role contains two kinds of polilcies: 

**Trust Policy** (control who can assume the role), below is a trust policy, only trust policy has "Principal" section and the "Action" has to be "sts:AssumeRole"

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { // <----------------------- trust policy has Principal, while permission policy doesn't
        "AWS": "arn:aws:iam::794303841582:user/Paul",  // this is ARN for user Paul
      },
      "Action": "sts:AssumeRole"  // sts is short for "security token service"
    }
  ]
}
```

**Permissions Policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "ec2:*",
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

**AWS Security Token Service** (STS) involves when an identity **assumes** a IAM role (`sts:AssumeRole` API call), 
STS create temporary (short-term) security credentials (like user's long-live access key pair except STS access key pairs don't belong to identity and it has expire time)
for EC2 instance to access resource. 

So the steps are:

1. Create user Paul and copy the arn
2. Create a role:
  a. (click custom policy) configure trust policy by copying Paul's arn
  b. add  permission policy
  c. Click Create the role and name it e.g EC2-Full-Access and copy the link of this role
3. Login as Paul in incognito mode of your browser, and click Switch role, now Paul can have full EC2 operations.

When you check upon on a Role, you can click "Trust relationships" tab to see the Trust Policy attached to it.

The **default** time for an STS session is **12 hours** and  the duration of a session can be customized to a range of **15 minutes to 36 hours**

**STS checks the role's TRUST Policy first and if the identity is allowed to assume the rule then STS reads the permission policy**, if permission allows, then STS creates a  security credential 
thats consists of `AccessKeyId`, `Expiration`, `SecretAccessKey` (used to sign request), and `SessionToken` (needs to be included in every request) and credential is returned to the identity that request them

==========================================================================


## AWS Organization and Service Control Policy

```yml
# Organizational structure, contains two Organizational units (OUs) 

Root
  |
  DEV (OU)
    |
    shoppigsau-development (memember account)  
    890742603224 | shoppigsau+development@gmail.com
# created directly (of course you have to provide unique email address but no credit card info is needed) in management account and automatically become part of this organization
# note that you cannot access this account directly (also there is no root user) and it can only be accessed by switching role from management account
  |
  PROD (OU)
     |
     shoppigsau-production (memember account)  
     221082202222 | shoppigsau+production@gmail.com
# received invition from management account and accept it then become part of this organization
  | 
  shoppigsau-general (management account)  
  242201314143 | shoppigsau@gmail.com
# Organization is created in management account, so this account becomes the management/master account
   
```

When you create an new aws account in a organization, `OrganizationAccountAccessRole` (OAAR) that has admin rights in that account is automatically created, if you invite an existing aws account (note that you will be using management account to send invitation to the account) to join the organization, then you have to manually create the role so that management account can assume it. Management account doesn't have any role such as OAAR being created.

When you assume an account's OAAR, it is like you login in to the account manually.

**Service Control Policy** (SCP) can be attached on Organizational unit level. However SCP cannot be applied on management account, AWS will popup error message *You can apply SCPs to only member accounts in an organization. They have no effect on users or roles in the management account*  if you try to attach a SCP to management account.

Note that when SCP is applied on a member account, **it can have an effect such as limit the account rooter user's permissions**, it is true that you cannot directly restrict root user, but SCP can indirectly restrict root user.

When you enable SCP (on management account), AWS apply a default policy called `FullAWSAccess` which has full AWS access as:

```json
// default SCP, this is call "Deny List" (as you need to explicit add Deny) against "Allow List"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

then you can apply a SCP on PROD with SCP being:

```json
// SCP
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```

remember when you switch to PROD from management account, you are assuming the `OrganizationAccountAccessRole` in PROD account and this OAAR has full admin access, but since this SCP denies S3 operation, you won't be able to use S3 service

====================================================


## CloudWatch Logs

A **Log Stream** is a series log events from the **same source/instance**, a **Log Group** consists multiple Log Stream


## CloudTrail

* Logs AWS's own Product's event
* **90** days stored by default in **Event History**, enabled by default, which has no cost for 90 days
* Log **Management Event** (Terminate an EC2 instance etc) and **Data** Events (an obejct is uploaded to S3), by default only logs Management Events
































=========================================================================================================================================================================================

AWS operates 24 **Regions** around the world, each region is composed of at least two, usually three **Availability Zones** (AZ). Each AZ is composed of one or more physically separated data centers. 

A `VPC` is a logically isolated portion within a region. A VPC can contain multiple AZs and each AZ is physically isolated. 


#### IAM




================================================================================================================================================


## CLI

`C:\Users\xxx\.aws` folder contains two files to save content when you run `aws configure`:

* `config` file contains profiles:

```yml
 [default]
region = us-east-1
output = json

[profile ec2-full-access]    # <----------manully add this profile without running `aws configure`, there is no [ec2-full-access] counterpart in credentials file
region = us-east-1
role_arn = arn:aws:iam::794303841582:role/EC2-Full-Access  # still only Paul can assume the role by the trust policy defined previously
source_profile = default

[profile Paul]          # <----------create by `aws configure --profile Paul`
region = us-east-1
```

* `credentials` file

access key and secret key are created under user
```yml
[default]  # default user is Paul we created previously
aws_access_key_id = xxx  # access key and secret below are generated in UI by clicking Creae Access Key of the User
aws_secret_access_key = xxx 

[Neil]        # <----------create by `aws configure --profile Paul`
aws_access_key_id = xxxx                                              
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  
```

note that when you do `aws ec2 describe-instances --profile ec2-full-access`, it uses the `[default]` credentials since we copy and paster the profile in the config file and therefore there is no counterpart in credentials file. But when you do `aws ec2 describe-instances --profile Paul`, it uses both `[profile Neil]` in the config file and `[Neil]` in the credentials file.

So if your default users is not Paul, when you try assue the role by `aws ec2 describe-instances --profile ec2-full-access` you will get an error

============================================================================================================================================================


## VPC

A **VPC** (analogous to having your own data center inside AWS) is a logically isolated (not physically isolated) portion of the AWS cloud within a region. You can create mutiple VPC in a region. Inside a VPC, there are multiple **Availability Zone** (AZ) which are *physically isolated* (different data centres).

`Subnets` are created within AZs. A subnet can only belong to one AZ, but you can span workload into different subnet in different AZ.

A **VPC Router** takes care of routing within the VPC and outside of the VPC. However, if a VPC wants to access Internet, an **Internet Gateway** is needed to attached to a VPC to connect to the Internet

**Network ACL** (NACL) apply at the **Subnet level**:

Inbound rules
--------------------------------------------------
Rule # |   Type    |   Protocol    | Port range  |   Source    |  Allow/ Deny |
100         SSH           TCP            22          0.0.0.0/0       Deny

Outbound rules
--------------------------------------------------
Rule #  |  Type    |   Protocol    | Port range  |   Source    |  Allow/ Deny |
100        HTTP          TCP            443          0.0.0.0/0       Allow



**Security Group** apply at the **Instance level**, an example of rules is:

Inbound rules
--------------------------------------------------
Type    |   Protocol    | Port range  |   Source  |
SSH           TCP            22          0.0.0.0/0

Outbound rules
--------------------------------------------------
Type    |   Protocol    | Port range  |   Source   |
HTTP          TCP            443          0.0.0.0/0

Unlike Network ACL that supports both allow and deny,  Security groups supports "allow" rules only, there is no "deny" rule, so any type which is not defined in the inbound rules is explicitly denied. For example, your instance cannot access HTTPS since it is not defined above. Also, Security Group are stateful, if a type is allowed in Outbound then it is automatically allowed Inbound



# Create VPC
Name: MyVPC
IPv4 CIDR Block: 10.0.0.0/16

# Create Subnets

Name: Public-1A
Availability Zone: us-east-1a
IPv4 CIDR Block: 10.0.1.0/24

Name: Public-1B
Availability Zone: us-east-1b
IPv4 CIDR Block: 10.0.2.0/24

Name: Private-1A
Availability Zone: us-east-1a
IPv4 CIDR Block: 10.0.3.0/24

Name: Private-1B 
Availability Zone: us-east-1b
IPv4 CIDR Block: 10.0.4.0/24

# Create private route table

Name: Private-RT
VPC: MyVPC
Subnet associations: Private-1A, Private-1B

# Create Internet Gateway

Name: MyIGW
VPC: MyVPC

--------------------------------------------------
0. Create a VPC and then tick "Enable DNS hostnames" which allow our EC2 instances to have hostname

1. When a new VPC is created, a route table is automatically created in the Route tables for the VPC

`Route Table`
```bash
| Route table ID        | VPC                   | Main | Name |
| rtb-0dd7233d23fcd8e6f | vpc-0253ab4d25d6c6b3e | Yes  |      |   # let's rename it "Main"
```

```bash
#-----------------------------------------V extra info for the automatically created public route table rtb-0dd7233d23fcd8e6f
Routes for rtb-0dd7233d23fcd8e6f
| Destination | Target | 
| 10.0.0.0/16 | local  |
#-----------------------------------------Ʌ
```

2. Create four subnets under `MyVPC` and enable **auto-assign public IPV4** address (receive public IP addresses from the Amazon pool) for the two public subnets

3. Create the private route table, now you have:

```bash
| Route table ID        | VPC                   | Main | Name       |
| rtb-0dd7233d23fcd8e6f | vpc-0253ab4d25d6c6b3e | Yes  |   Main     |
| rtb-08015053a2faf19c2 | vpc-0253ab4d25d6c6b3e | No   | Private-RT |
```

Note that the extra info is the same as rtb-0dd7233d23fcd8e6f for this Private-RT:

```bash
#-----------------------------------------V extra info for the newly created private route table rtb-08015053a2faf19c2
Routes for rtb-08015053a2faf19c2
| Destination | Target | 
| 10.0.0.0/16 | local  |

Explicit subnet associations (0)

Subnets without explicit associations (4)
| Name       | Subnet ID                 | 
| Public-1A  | subnet-08f53ac471b6fa9a3  |
| Public-1B  | subnet-01d55e6b1ca805701  |
| Private-1A | subnet-0c3952e88f3ee2df4  |
| Private-1B | subnet-0c1df9c2b47906f90  |
#-----------------------------------------Ʌ
```

4. Associate `Private-1A` and `Private-1B` with `Private-RT`, then you have

```bash
#-----------------------------------------V new extra info for the route table rtb-08015053a2faf19c2
Routes for rtb-08015053a2faf19c2
| Destination | Target | 
| 10.0.0.0/16 | local  |

Explicit subnet associations (2)
| Private-1A | subnet-0c3952e88f3ee2df4 |
| Private-1B | subnet-0c1df9c2b47906f90 |

Subnets without explicit associations (2)
| Name       | Subnet ID                | 
| Public-1A  | subnet-08f53ac471b6fa9a3 |
| Public-1B  | subnet-01d55e6b1ca805701 |
#-----------------------------------------Ʌ
```

once you assocate the private subnet to the private route table, the original public route become

```bash
#-----------------------------------------V extra info updated for route table rtb-0dd7233d23fcd8e6f   
Routes for rtb-0dd7233d23fcd8e6f            
| Destination | Target | 
| 10.0.0.0/16 | local  |

Explicit subnet associations (0)

Subnets without explicit associations (2)
The following subnets have not been explicitly associated with any route tables and are therefore associated with the main route table:
| Name       | Subnet ID                 |  # Private-1A and Private-1B are no longer associated with the public route
| Public-1A  | subnet-08f53ac471b6fa9a3  |
| Public-1B  | subnet-01d55e6b1ca805701  |
#-----------------------------------------Ʌ
```

5. Create a new Internet gateways `MyIGW` and attach it to MyVPC

6. Since we create the Internet gateway, we need to add a new route in the **public route table** `rtb-0dd7233d23fcd8e6f` as below so it can access the outside internet

```bash
Routes for rtb-0dd7233d23fcd8e6f  # Main 
| Destination | Target                        | 
| 10.0.0.0/16 | local                         |
| 0.0.0.0/0   | igw-0ea30a8ccd7c91ed8 (MyIGW) |
```

7. Create a NAT Gateway (`MyNATGW`) under `Public-1A` subnet (you can only place NAT Gateway in public subnet), you will need to assign Elastic IP address for the NAT Gateway

8. Add a route with `MyNATGW` in the private route table `Private-RT`

```bash
Routes for rtb-08015053a2faf19c2  # Private-RT
| Destination | Target                          | 
| 10.0.0.0/16 | local                           |
| 0.0.0.0/0   | nat-0e4589425f512051a (MyNATGW) |
```

this step is important because our private subnets will have access to the internet as when our instance launches, we're going to run some user data. And that's going to connect to the internet to download some binaries for HTTPD.


The definition of a Public Subnet in an Amazon VPC is:
   *The Route Table attached to the Subnet has a Route with a destination of 0.0.0.0/0 that points to an Internet Gateway*

If a VPC does not have an Internet Gateway, then the resources in the VPC cannot be accessed from the Internet. NAT Gateway does something similar to Internet Gateway, but it only works one way: Instances in a private subnet can connect to services outside your VPC but external services cannot initiate a connection with


```yml
aws ec2 run-instances --image-id <value> --instance-type <value> --security-group-ids <value> --subnet-id <value> --key-name cloud-labs-nv --user-data

aws ec2 terminate-instances --instance-ids <value> <value>
```


## Elastic Block Store (EBS) VS Instance Store

*EBS* volume is attached overa network, not within EC2 host. EBS volumn still persist when the EC2 instance get terminated. An EBS volumn can only be attached to **only one EC2 instance at a time**.

**Instance Store** volumns are physically attached to EC2 host (offer high-performance), and they are non-persistent so if you stopped, hibernated, or terminated your EC2 instance, the instance store data is gone (reboot is fine).

Every EC2 instance has a root EBS volumn (to store operating systmes) created when the EC2 instance is created. Note that root volumn will be deleted when you delete the corresponding EC2 instance but any addtional volumns you attach to the instance will still be available

There are 3 main category of EBS:

* Hard disk drive (HDD) volumes: `st1`, `sc1` (Cold HDD volumes), HDD cannot be used as boot volumes

* General Purpose SSD volumes: `gp2`, `gp3`

* Provisioned IOPS SSD volumes (support multi-attach): `io1`, `io2`, great for database workload

*multi-attach* can attach a EBS volume to multiple EC2 instances (maximum 16 EC2 instances) in the **same AZ**



## EFS ( Elastic File System)

EFS is a managed NFS (Network File System) and it can be attached to multiple EC2 instances in **multiple AZs** (like a network shared drive), while EBS can only be attached to one EC2 instance (there is only one version io1 supports multi-attach), and EBS needs to be in the same AZ as the EC2 instance.

There are 3 main category of EFS:

* EFS Standard
* EFS Infrequent Access
* EFS Archive

When an EFS is created, a DNS name (e.g fs-0f435408205fb6916.efs.us-east-1.amazonaws.com) is created and and if this file system is attached to an EC2 instance, two security groups are created:

1.  `instance-sg-1` attched to the EC2 instance (allow NFS on port 2049 as outbound rules) 
2.  `efs-sg-1` attached to the EFS (allow NFS on port 2049 as inbound rules), yes EFS can contain security group

```yml
sudo mount -t efs -o tls fs-0f435408205fb6916.efs.us-east-1.amazonaws.com:/ ~/efs-mount-point
```
====================================================================================================


## Load Balacners

**Application Load Balances (ALB)**: operates at application layer felxibile of forwarding traffic in many ways: host, path, query/string etc;
you can only select HTTP or HTTPS as listener. ALB uses static DNS name and it cannot be assigned an Elastic IP address

you can create both an ALB and a security group (say sg1) and attach sg1 to that ALB, and lauch an EC2 instance whose own security group's (say sg2) inbound rule is set to sg1 with HTTP. In this way, we can forbid directly access to the EC2 instance from public and public users have to access the ALB only to be able to send request to the EC2 instance, and the ALB will redirect (adding  a header `X-Forwarded-For` that includes the original client's ip address) the request to the EC2 instance.

**Network Load Balances (NLB)**: operates at network layer, not flexible as ALB, just forwards based on the protocol (TCP, UDP, TLS only), but NLB is high-performance that can handles millions of requests per second. NLB uses both static DNS name and static IP.


Cross-Zone Load Balancing:

`ALB`: Enable by default, no charges for inter AZ data
`NLB`: disable by default, you pay charges for inter AZ data


===================================================================================================


## S3

After enabling versioning,  existing object before versioning will have null Version Id, if you delete a object whose Version Id is null, you are not really delete it, instead, S3 will add a deletion mark record

* Standard-General Puporse: $0.023 per GB $0.005 per 1,000 requests
* Standard-Infrequent Access (`S3 Standard-IA`): $0.0125 per GB, longer storage time than in the case of S3 Standard 
* One Zone-Infrequent Access : Unlike other S3 Storage Classes which store data in a minimum of three Availability Zones (AZs), S3 One Zone-IA stores data in a single AZ and costs 20% less than `S3 Standard-IA`
* Glacier Instant Retrieval: $0.004 per GB, $0.02 per 1,000 requests
* Glacier Flexible Retrieval: Expedited (1min to 5 min), Standard (3 to 5 hours), Bulk (5 to 12 hours)
* Glacier Deep Archive: Standard (12 hours), Bulk (48 hours)
* Intelligent Tiering: move objects automatically between Access Tiers based on usage, default tier is Frequent Access tier, then objects not accessed for 30 days are moved to Infrequent Access tier (tiers are exclusive in Intelligent Tiering, ie not S3 Standard-IA) etc

=======================================================================================================


## ECS

A **Cluster** is logical grouping of tasks or services. When creating a cluster, you can choose Infrastructure to be "AWS Fargate" (automatically selected for you and you cannot de-select it), "Amazon EC2 instances", and "External Instances using ECS Anywhere".

A **Task** is a running docker container (I guess aws "task" instead of "container" to differential EC2 contianer)

A **Task definition** is like a "docker compose file" where you specify multiple docker containers. You can specify *Lauch Type* to be *Fargate*, *EC2 Instances*,
or *External* (run containers on on-premises servers, you can only sepcify it in Service)
Note that *Fargate* is serveless so you don't need to manager the underlying EC2 containers (you won't see new EC2 instance in your AWS), while *External* mode,  EC2 Instances you have manage the EC2 instances and install "agent" to register your EC2 instance with the underlying cluster, for *EC2 Instances* mode, AWS will install agents and regiser with cluster for you.

A **Service** choose a Task definition and defines long running tasks-can control task count with Auto Scaling and attach an ELB. You can multi-select  *Fargate*, *EC2 Instances* or *External*.

Note that *IAM instance role* (for underlying EC2 instasnces in non-fargate mode) is different to the* IAM Task Role*, and container instance have access to all of the permissions that are supplied to the container instance role through instance metadata. For Farget mode, there is only IAM Task Role, of course, because it doesn't have underlyhing EC2 instance.

**Task Placement Strategy**: define how tasks are distributed onto EC2 instances in the cluster


The step to Create an ECS Cluster in details: 

choose EC2 Lauch Type to be *External* (best learning example where you immediately know how AWS manages *EC2 Instances* mode for you):

0. Create a Role (trusted entity to be EC2) that contains `AmazonEC2ContainerServiceforEC2Role` policy (contains many ecs related Allow statement e.g ecs:CreateCluster)
0. Launch an EC2 instance (AMI needs to be an ECS-optimized AMI which pre-installed with ECS agent), also attach the role created above in the setting and enter User Data below:

```bash
#!/bin/bash
echo ECS_CLUSTER=myecs-cluster >> /etc/ecs/ecs.config  # <---------------------this will "register" the EC2 instance with your ECS cluster
```

1. Create a Cluster, note that we choose Infrastructure to be *External* as explained above.

2. Create a **Task Definition** to specify what docker image/s you want to use to run a container on EC2. Note that an app might need multiple different docker instances (e.g webapp itself and database instance), that why you can define multiple container in Task Definition by clicking "Add container".

3. Click **Run Task** to chose the Lauch Type (EC2 in this case) and specify the VPC, subnet etc and the task definition in step 1. Then the nginx docker instance will be running inside the registered EC2 instance. Note that a Task can consit of multiple running docker instances.

Differebce between Task and Service is:

A Task is created when you run a Task directly, which launches container(s) (defined in the task definition) until they are stopped or exit on their own, at which point they are not replaced automatically. Running Tasks directly is ideal for short-running jobs, perhaps as an example of things that were accomplished via CRON.

A Service is used to guarantee that you always have some number of Tasks running at all times using ASG. If a Task's container exits due to an error, or the underlying EC2 instance fails and is replaced, the ECS Service will replace the failed Task. 


**Dynamic Host Port Mapping**:

For EC2 Lauch Type, an EC2 instance is assigned multiple random port numbers (host ports) that mapped to container ports, so you must allow on the EC2 instance's Security Group any port from the ALB's Security Group.

For Fargate: each task gets a unique private ip adddress, there is no host port since we don't manage underlying EC2 instance. Each task is assigned with an ENI whose expose port is the same as the task's expose port, so you only need to allow port 80 (443 is not needed since the traffic is already in VPC) from ALB (security group allow port 80/443).


**ECS Task Placement Strategies** in Service setup (only apply to EC2 Instance, not Fargate):

* `Binpack`: minimizes the number of EC2 instance in use by placing tasks on least usage of CPU, memeory etc
```json
"placementStrategy" : [
  {
     "field": "memory",
     "type": "binpack"
  }
]
```
* `Random`: pretty straightforward
* `Spread`:
```json
"placementStrategy" : [
  {
     "field": "attributes:ecs.availability-zone",
     "type": "spread"
  }
]
```
* `AZ balanced spread`
* `AZ balanced binpack`
* ...

**ECS Task Placement Constraints** in Task Definition setup

* `distinctInstance`: place each task on different EC2 instance
* `memberof`:

```json
"placementConstraints" : [
  {
     "field": "attributes:ecs.instance-type =~ t2.*",
     "type": "memberof"
  }
]
```

## CloudFormation

**Service Role or Service-linked Role**:

Image current user Paul doesn't have permission to create an EC2 instance but the senior developer still assign a CloudFormation template for him to execute. That's how "Service Role" comes into play when Paul is createing a stack he can specify a service role for CloudFormation to perform tasks. Below is the steps:

1. Create a **service role** e.g  **CloudFormationServiceRole** that specify CloudFormation has permission to create EC2 instance. Its trust entity is like:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
        "Sid": "",
        "Effect": "Allow",
        "Principal": {
            "Service": "cloudformation.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
    }
  ]
}
```

then attach e.g "AmazonEC2FullAccess" permission


2. Create a **user role** where you specify a trust policy that Paul can assume rule

 ```json
 {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::794303841582:user/Paul", 
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
 * a permission that is attached to the user role above

 ```json
 {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "iam:PassRole"  // <----------------PassRole is needed, what it means is "Paul can attach 'CloudFormationServiceRole' as service role to CF
                            // also note that Paul doesn't have permission to create EC2 instances, however when he uses CF (assumes that Paul has permission to use CF), CF can create EC2 instance
        ],
        "Resource": "arn:aws:iam::<account-id>:role/CloudFormationServiceRole"
    }]
}
 ```



 ## SQS

SQS scales automatically.
 
A message's default rention is **4** days, and maxinum of 14 days. A message is persisted in SQS until a consumer deletes it (uses **DeleteMessageAPI**).

Consumer polls SQS for messages (receive up to **10** messages at a time, default is 1)

**Message Visibility Timeout** is the time that if a consumer polls a message, other consumers won't see/poll this message until specified interval (default is 30 secs) collapese, and if the first consumer doesn't delete the message via DeleteMessageAPI, then this messasge goes back to the queue and will be consumer by other consumers. Note that a consumer could call 

**ChangeMessageVisibilityAPI** (for that message only, not all messages) to exent the time.

**Dead Letter Queue** is a special queue for messages that reach `MaximumReceives` threshold, so that a problemtic message (or caused by consumer bug that didn't delete the message) won't be in and out the queue again and again and again. Good to set a retention of 14 days for debugging. **Redrive allow policy** let us redrive the messages from the DLQ back into the source queue e.g after we fix our consumer code.

Say you have two queue, QueueA (as source queue) and QueueB (DLQ), on AWS UI, you go into QueueA's setting and set the QueueB as its DLQ. Note that AWS UI is a little bit confusing, because the wording of "Set this queue to receive undeliverable messages" radio boxes's text, "this queue" doesn't refer to QueueA, it refer to QueueB as after choose "Enabled", you select QueueB in the dropdown, so "this queue" from "Set this queue to receive undeliverable messages" mean the queue you are going to specify in the dropdown.

And if you go to QueueB's UI setting in the **Redrive allow policy** you can choose Enable first (Disabled as default means allowing all source queues to use this queue as the dead-letter queue) then choose:
* Allow All: (a little bit different than the "default allowing all", that's why it is under Enable option), 
* By Queue: Allow a list of source queues from the same account and in the same region to use this queue as the dead-letter queue
* Deny All: QueueB "refuse" QueueA's "request", so even QueueA set QueueB as its DLQ, but because QueueB refuses it, messages supposed to go to DLQ still stay in QueueA like nothing happen


**Short Polling**: (`ReceiveMessageWaitTimeSeconds` being 0)  queries only a **subset of the servers** (based on a weighted random distribution) to find messages that are available to include in the response. AWS sends the response right away, even if the query found no messages.

**Long Polling**: (`ReceiveMessageWaitTimeSeconds` greater than 0) queries **all of the servers** for messages. Amazon SQS sends a response after it collects at least one available message, up to the maximum number of messages specified in the request (note that it hangs around when there is no messages but it then also can return immedately if one of servers has a message and look like in this case only a single message are return due to its listener, need to dig into SQS architecture). An empty response is sent only if the polling wait time expires.

Short Polling is default, it is faster and the messages can be processed quickly, but on the other hand, your server does need to make an additional calls. In the worst case, no message is returned from the request, even though messages are in the queue (because of a weighted random distribution, see https://flofuchs.com/taking-a-look-at-aws-sqs-short-and-long-polling for details). This is even more noticeable for queues that only have a few messages


**SQS Extended Client** (Java Llibrary): send large messges by only sending real large to S3 and small metadata message (like a pointer) to SQS queue, once consumer receives this metadata message as pointer, it knows how to retreve the real large message from S3.


**FIFO** queue: message ordering is perserved (comapred to Standard Queue who only do best-effort ordering):

* **Limited throughput**, 300 msgs without batching, 3000 msgs with batching, compared to **Standard Queue** who has unlimited throughput
* **Exact-once** (by removing duplicateds)  compared to **Standard Queue** who is **at-least once**

FIFO queue also supports De-duplication interval (5mins):

* **Content-based duplication**: will do a SHA-256 hash of the message body
* **Message deduplication ID**: explicitly provide a **Deduplication ID**

you can also specify "Message group ID" so messages that belong to the same message group are always processed in a strict order relative to the same message group (but not cross different message group)


## SNS 

Create a topic and add subscription such as SQS, Lambda etc. It allows **Message Filtering** so a subscription can only recieve messages that falls into the filter policy



## Kinesis

has a kafka partition alike concept called **Shard** and one shard can only be consumed by one consumer. If one shard receive too many data becomes a "hot" shard, you can spilt it into two shards or merging two cold shards into one shard.

Consumers Types:

Shared (Classic):
* Read throughput: 2mb/sec per shard accross all consumers
* Consumer **poll** data from Kinesis using `GetRecords` API call
* Latency: 200ms
* lower cost

Enhanced:
* Read throughput: 2mb/sec per consumer per shard
* Kinesis pushes data to consumer via HTTP/2 using `SubscribeToShard` API
* Latency: 70ms

Kinesis VS SQS ordering: Let's assume 100 trucks, 5 kinesis shards VS 1 SQS FIFO

Kinesis Data Streams:
* 20 trucks per shard
* maximum consumer number in parallel is 5

SQS FIFO:
* you will have 100 Message Group ID
* you can have up to 100 Consumers


Data Streams vs Firehose

Streams: 
* Kinesis data streams is highly customizable and best suited for developers building custom applications or streaming data for specialized needs.
* Going to write custom code
* Real time (200ms latency for classic, 70ms latency for enhanced fan-out)
* You must manage scaling (shard splitting/merging)
* Data storage for 1 to 7 days, replay capability, multi consumers
* Use with Lambda to insert data in real-time to ElasticSearch
Firehose:
* Firehose handles loading data streams directly into AWS products for processing.
* Fully managed, send to S3, Splunk, Redshift, ElasticSearch
* Serverless data transformations with Lambda
* Near real time (lowest buffer time is 1 minute)
* Automated Scaling
* No data storage



## Observability

**Metrics**: 
EC2 metrics are sent every 5 mins by default. Detailed EC2 monitoring sends every 1 min

Custom metrics:
* Standard resolution: data haveing a **1 min** granularity
* High resoulution: data at **1 sec** granularity

using `put-metric-data` api call

```bash
#!/bin/bash    <----------------you can use `stress-ng` to run this script every minute

# Create a token for IMDSv2 that expires after 60 seconds
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60" -s`

# Use the token to fetch the EC2 instance ID
INSTANCE_ID=`curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/instance-id`

# Get memory usage and put metric data to CloudWatch
MEMORY_USAGE=$(free | awk '/Mem/{printf("%d", ($2-$7)/$2*100)}')  # <-------------------use Linux's free command and pattern scanning
aws cloudwatch put-metric-data --region us-east-1 --namespace "Custom/Memory" --metric-name "MemUsage" --value "$MEMORY_USAGE" --unit "Percent" --dimensions "Name=InstanceId,Value=$INSTANCE_ID"
```


**Metric Filter** : let you turn log into metric and you can create a Alarm based on it


reservior and rate in aws xray? what does rate mean by "additional percentage of requests" how does it know how many requests it will be in advance
## <---------------------------------------------------need to review all concepts later after learning tracing and One Observability Workshop



## Lambda 

At the heart of AWS Lambda is Firecracker, a virtualization technology developed by Amazon and implemented in Rust. Firecracker powers the engine on which all Lambda functions run, providing a lightweight yet robust platform for executing code. Firecracker's unique design allows it to offer the security and workload isolation associated with virtual machines while maintaining the speed and efficiency that containers typically provide. Firecracker uses the Linux Kernel-based Virtual Machine (KVM) to create and manage microVMs. Firecracker has a minimalist design. It excludes unnecessary devices and guest functionality to reduce the memory footprint and attack surface area of each microVM.

Lambda will create its execution environments on a ﬂeet of Amazon EC2 instances (bare netak EC2 uses KVM) called **AWS Lambda Workers**. Workers are bare metal Amazon EC2 AWS Nitro instances which are launched and managed by Lambda in a separate isolated AWS account which is not visible to customers


**ALB with Lambda**:

Create a ALB with security group and a target group (where you need to choose target type as "Lambda function")

ALB transits http request to json to Lambda and then ALB transits Json response from Lambda to Http response. ALB support **multi-header** value via ALB settings, so a Http request passes from client suc as http://example.com/path?name=foo&name=bar   ---->  ALB (invoke Lambda with `{ "queryStringParameters": { "name" : ["foo", "bar"]}}` ) ----> Lambda

```bash
## sync
aws lambda invoke --function-name mytestfunction --payload BASE64-ENCODED-STRING response.json  # default invocation-type for sync is RequestResponse

## async, Lambda will retry 3 times on errors, you can setup a Dead-Letter queue for Lambda to publish over-error event to SQS
aws lambda invoke --function-name mytestfunction --invocation-type Event --payload BASE64-ENCODED-STRING response.json  ## --invocation-type Event flag makes it async
```


**Event Sourcing Mapping** 

Lambda -----poll SQS queue------->  SQS  <-------------message added by user
  |
  |--------> event written to CW


```bash
# create a role for Lambda with AWSLambdaSQSQueueExecutionRole policy and attached to the lambda function below
aws lambda create-function --function-name EventSourceSQS --zip-file fileb://function.zip --handler index.handler --runtime nodejs16.x --role arn:aws:iam::211125381467:role/my-sqs-role

## create a event source mapping, this is equivalent to "Add Trigger" in UI which connected to SQS logo 
aws lambda create-event-source-mapping --function-name EventSourceSQS --batch-size 10 --event-source-arn arn:aws:sqs:ap-southeast-2:211125381467:MyQueue

aws lambda list-event-source-mappings --function-name EventSourceSQS --event-source-arn arn:aws:sqs:ap-southeast-2:211125381467:MyQueue

aws lambda delete-event-source-mapping --uuid c3dbb673-e9d7-4328-9de1-a03ffb969c7d  ## uuid is generated by create-event-source-mapping
```

```json
// AWSLambdaSQSQueueExecutionRole policy
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:GetQueueAttributes",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

**Version and Aliases**

```bash
## Invoke Lambda versions
aws lambda invoke --function-name myversiontest response.json
aws lambda invoke --function-name myversiontest:\$LATEST response.json  ## Lambda automatically use $LATEST

## modify the code and click "Deploy", click "Publish new version", then you specify a version name e.g v1 or z1 (Lambda will only use integer e.g 1, 2, 3 later in the dropdown)
aws lambda invoke --function-name myversiontest:1 response.json

## modify the code again ....
aws lambda invoke --function-name myversiontest:2 response.json

## Create an alias (note that alias name cannot be a integer because number is for version, string is for alias) and Invoke Lambda alias
aws lambda invoke --function-name myversiontest:myapp response.json
## you can shift traffic between two version when createing an alias by specifying a weight to each version such as myversiontest:1 and myversiontest:2
```

  