## Demystifying AWS

AWS operates 24 **Regions** around the world, each region is composed of at least two, usually three **Availability Zones** (AZ). Each AZ is composed of one or more physically separated data centers. 

A `VPC` is a logically isolated portion within a region. A VPC can contain multiple AZs and each AZ is physically isolated. 


#### IAM


IAM role contains two kinds of polilcies: 

**Trust Policy** (control who can assume the role), below is a trust policy, only trust policy has "Principal" section and the "Action" has to be "sts:AssumeRole"

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
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

**AWS Security Token Service** (STS) involves when a service such as EC2 instance **assumes** a IAM role (`sts:AssumeRole` API call), STS create temporary security credentials for EC2 instance to access resource

So the steps are:

1. Create user Paul and copy the arn
2. Create a role:
  a. (click custom policy) configure trust policy by copying Paul's arn
  b. add  permission policy
  c. Click Create the role and name it e.g EC2-Full-Access and copy the link of this role
3. Login as Paul (Paul doesn't have permission at the moment to use EC2 service), and click Switch role, now Paul can have full EC2 operations.

When you check upon on a Role, you can click "Trust relationships" tab to see the Trust Policy attached to it.

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

A **Service** choose a Task definition and defines long running tasks-can control task count with Auto Scaling and attach an ELB. You multi-select  *Fargate*, *EC2 Instances* or *External*.

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


## CloudFormation

**Service Role**:
Image current user Paul doesn't have permission to create EC2 instance but the senior developer still assigne a CloudFormation template for him to execute. That's how "Service Role" comes into play when Paul is createing a stack he can specify a service role for CloudFormation to perform tasks. Below is the steps:

1. Create a service role e.g  **CloudFormationServiceRole** that specify CloudFormation has permission to create EC2 instance. Its trust entity is like:

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


2. Create a user role e.g **IAMPassRole** where you specify
  *  a trust policy that Paul can assume rule
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
 * a permission that is attached to the role

 ```json
 {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "iam:PassRole"  // <----------------PassRole is needed, what it means is "Paul can attach 'CloudFormationServiceRole' as service role to CF
        ],
        "Resource": "arn:aws:iam::<account-id>:role/CloudFormationServiceRole"
    }]
}
 ```