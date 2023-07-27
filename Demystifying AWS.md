## Demystifying AWS

AWS operates 24 **Regions** around the world, each region is composed of at least two, usually three **Availability Zones** (AZ). Each AZ is composed of one or more physically separated data centers. 

A `VPC` is a logically isolated portion within a region. A VPC can contain multiple AZs and each AZ is physically isolated. 


#### IAM


IAM role contains two kinds of polilcies: 

* Trust Policy (control who can assume the role), below is a trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::794303841582:user/Paul"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

* Permissions Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "ec2:*",
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Action": "elasticloadbalancing:*",
      "Effect": "Allow",
      "Resource": "*"
    },
  ]
}
```

**AWS Security Token Service** (STS) involves when a service such as EC2 instance **assumes** a IAM role (`sts:AssumeRole` API call), STS create temporary security credentials for EC2 instance to access resource


## CLI

We first create a trusted policy and attach it to a role (`EC2-Full-Access`)we create in the console management so only user Paul can assume the role:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::794303841582:user/Paul"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

`C:\Users\Dong\.aws` folder contains two files to save content when you run `aws configure`:

* config file

contains profiles:
```yml
 [default]
region = us-east-1
output = json

[profile ec2-full-access]    # <----------manully copy and paste, there is no [ec2-full-access] counterpart in credentials file
region = us-east-1
role_arn = arn:aws:iam::794303841582:role/EC2-Full-Access
source_profile = default

[profile Paul]          # <----------create by `aws configure --profile Paul`
region = us-east-1
```

* credentials file

access key and secret key are created under user
```yml
[default]
aws_access_key_id = AKIA3R4BSYUXLG6WTTMR
aws_secret_access_key = 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   

[Paul]        # <----------create by `aws configure --profile Paul`
aws_access_key_id = xxxx
aws_secret_access_key = /xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 
```

note that when you do `aws ec2 describe-instances --profile ec2-full-access`, it uses the `[default]` credentials since we copy and paster the profile in the config file and therefore there is no counterpart in credentials file. But when you do `aws ec2 describe-instances --profile Paul`, it uses both `[profile Paul]` in the config file and `[Paul]` in the credentials file.

So if your default users is not Paul, when you try assue the role by `aws ec2 describe-instances --profile ec2-full-access` you will get an error


## VPC

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

1. When a new VPC is created, a route table is automatically created in the Route tables for the VPC

`Route Table`
```bash
| Route table ID        | VPC                   | Main | Name |
| rtb-0dd7233d23fcd8e6f | vpc-0253ab4d25d6c6b3e | Yes  |      |
```

```bash
#-----------------------------------------V extra info for the automatically created public route table rtb-0dd7233d23fcd8e6f
Routes for rtb-0dd7233d23fcd8e6f
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

2. Create four subnets under `MyVPC` and enable auto-assign public IPV4 address for the two public subnets

3. Create the private route table, now you have:

```bash
| Route table ID        | VPC                   | Main | Name       |
| rtb-0dd7233d23fcd8e6f | vpc-0253ab4d25d6c6b3e | Yes  |            |
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
#-----------------------------------------V new extra info for the route table rtb-0dd7233d23fcd8e6f
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
#-----------------------------------------V extra info updated for route table rtb-08015053a2faf19c2
Routes for rtb-0dd7233d23fcd8e6f            
| Destination | Target | 
| 10.0.0.0/16 | local  |

Explicit subnet associations (0)

Subnets without explicit associations (4)
| Name       | Subnet ID                 |  # Private-1A and Private-1B are no longer associated with the public route
| Public-1A  | subnet-08f53ac471b6fa9a3  |
| Public-1B  | subnet-01d55e6b1ca805701  |
#-----------------------------------------Ʌ
```

5. Create a new Internet gateways `MyIGW` and attach it to MyVPC

6. Since we create the Internet gatewat, we need to add a new route in the **public route table** `rtb-0dd7233d23fcd8e6f` as below so it can access the outside internet

```bash
Routes for rtb-0dd7233d23fcd8e6f
| Destination | Target                        | 
| 10.0.0.0/16 | local                         |
| 0.0.0.0/0   | igw-0ea30a8ccd7c91ed8 (MyIGW) |
```

7. Create a NAT Gateway (`MyNATGW`) under `Public-1A` subnet

8. Add a route with `MyNATGW` in the private route table `Private-RT`

```bash
Routes for rtb-08015053a2faf19c2
| Destination | Target                          | 
| 10.0.0.0/16 | local                           |
| 0.0.0.0/0   | nat-0e4589425f512051a (MyNATGW) |
```

this step is important because our private subnets will have access to the internet as when our instance launches, we're going to run some user data. And that's going to connect to the internet to download some binaries for HTTPD.
