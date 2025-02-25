## Demystifying AWS

# Animals4Life

BNE HQ:   `192.168.10.0/24`, `10.0.0.0/16` (AWS Pilot),
                             `172.31.0.0/16` (Azure Pilot)
London:   `192.168.15.0/24`
New York: `192.168.20.0/24`
Seattle:  `192.168.24.0/24`
Google:   `10.128.0.0/9`  (previous vendor's usage)

Design:

~~Single Tier~~:
* Reserve at least **2** subnets/networks per region per account for buffering
* **40 ranges/subnets**: 3 × US regions, 1 × Europe region, 1 × Australia region, 4 × AWS accounts: (3 + 1 + 1) × 2 × 4 = 40
* Use the `10.16.0.0/16` -> `10.127.0.0/16` to avoid common such as `10.8.0.0`, and Google's reserved above
* <--------------------------why no overlapping ranges????

Multi-Tiers:

* **4 AZs**: AZ-A, AZ-B, AZ-C, AZ-Future
* **4 Tiers**: **Web**, **App**, **Db**, **Spare**

```bash 
# You custom VPC's CIDR: 10.16.0.0/16
##-------------------------------------------------------------------------V us-east-1
#AZ       Reserved            Db               App             Web (public)
AZ-A    10.16.0.0/20     10.16.16.0/20     10.16.32.0/20     10.16.48.0/20
AZ-B    10.16.64.0/20    10.16.80.0/20     10.16.96.0/20     10.16.112.0/20
AZ-C    10.16.128.0/20   10.16.144.0/20    10.16.160.0/20    10.16.176.0/20
#AZ-X   10.16.192.0/20   10.16.208.0/20    10.16.224.0/20    10.16.240.0/20   # AZ-X is for Reserved
##-------------------------------------------------------------------------Ʌ us-east-1
```

```bash
# Route Table for Public Subnets (Web)--------------------------------V

# Destination        Target
10.16.0.0/16         local  # VPC's CIDR, not subnet's CIDR, as it is easier to associate a route table with multiple subnets that falls into criteria as you see in NAT section
0.0.0.0/0            igw-xxx
#---------------------------------------------------------------------Ʌ

# Route Table for Private Subnets (Db, App)---------------------------V

# Destination              Target
10.16.0.0/16               local      # VPC's CIDR, not subnet's CIDR
#---------------------------------------------------------------------Ʌ
```

The VPC Router knows about all the subnet CIDR ranges, they are subsets of the VPC CIDR range. In a route table, the `local` entry is usually **configured with the CIDR range of the entire VPC**. When an instance sends a packet to VPC Router, it looks up the route table, and sees that the destination is "local", i.e. another host in the same VPC, it would then look through the subnet CIDR ranges, sees that private is in another subnet's range, and so routes the packets there.


# VPC (Virtual Private Cloud)

```bash
# private address ranges
10.0.0.0     -   10.255.255.255  (CIDR block: 10.0.0.0/8)
172.16.0.0   -   172.31.255.255  (CIDR block: 172.16.0.0/12)
192.168.0.0  -   192.168.255.255 (CIDR block: 192.168.0.0/16)

169.254.0.0  -   169.254.255.255 (CIDR block: 169.254.0.0/16)  link-local addresses fall within the range of 
```

```bash
# default VPC
172.31.0.0/16  # <-------------------------------------------------------------------------default VPC is not recommended to use, seel later in the course and come back edit the reason

# you can create a custom VPC with CIDR block 10.16.0.0/16

# default ap-southeast-2 subnets    last two octets         each network/subnet's host range        each subnet maps to an AZ
172.31.0.0/20                     0000|0000,00000000       start 172.31.0.0  end 172.31.15.255          ap-southeast-2a
172.31.16.0/20                    0001|0000,00000000       start 172.31.16.0 end 172.31.31.255          ap-southeast-2b
172.31.32.0/20                    0010|0000,00000000       start 172.31.32.0 end 172.31.47.255          ap-southeast-2c
#... extend and map your custom subnet into one AZ

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

* A VPC has DHCP Options Set that applied to itself, and this DHCP setting flows into its subnets

* Every VPC has a **VPC Router**, see `10.16.0.1`- VPC Router below. The primary function of this VPC router is to take all of the route tables defined within that VPC, and then direct the traffic flow within that VPC, as well as to subnets outside of the VPC, based on the rules defined within those tables. VPC Router manages the traffic between subnets (in same VPC of course) and between a subnet with AWS Public Zone such as Internet Gateway, note that the traffic can be controlled by **Route Table** of subnets. Every subnet comes with a default Route Table (e.g the one in next section where the first two entries are the default ones).

* An **Internet Gateway** (`IGW`)is region resilient (works for all AZs) and can be attached to a VPC

* For ipv4, no public ip address is assigned to e.g EC2 instance, **public IP addresses are assigned to an Elastic Network Interface (ENI)**. An instance can have multiple ENIs. and  `IGW` assoicates the instance's private address with a public ip address via translation (this process is called **static NAT**), so OS won't be able to see the IPv4 address. However for IPv6, public ip addresses are directly assigned to instances (underlying OS can see ipv6 public address). 
When you lauch a new EC2 instance, you get one public DNS name someting like `ec2-3-89-7-136.compute-1.amazonaws.com`, one **primary** private IPV4 address and one public IPV4 address (will change when you stop and start the EC2 instance), inside the VPC, the public DNS name always resolves to the private ip address and outside in public internet, the public DNS name resolve to the public ip address. Note that EC2 instance can be attached mutiple ENI (Elastic Network Interface) and each ENI has its private IP address, and you can have one elastic ip per pirvate IPV4 address

Note that for most of online course that instructs to create an EC2 instance then change security group to enable SSH on port 22 then remote connect to the instance, this work because you create the EC2 instance in default VPC's subnets which are all public subnets by default and those public subnets are connected to default IGW.


## Subnets

A subnetwork of a VPC, **1 subnet can only be placed in 1 AZ, a subnet can never span over multiple AZs**, while an AZ can contain multiple subnets

There are 5 reserved IP address in every subnet which cannot be used, for example  `10.16.0.0/20`
* `10.16.0.0`- Network Address that represents the subnet itself
* `10.16.0.1`- VPC Router
* `10.16.0.2`- Reserved DNS
* `10.16.0.3`- Reserved for future use
* `10.16.31.255`- Broadcast Address

## Create Public Subnets

1. Create the VPC and its subnets
2. Create an Internet Gateway (says it is `igw-xxx`) to assoicate it with the VPC you create
3. Create a new Route Table per public subnet and then click "Edit Subnet Associations" to associate the public subnet (e.g `Web`) you created. By doing this, those subnets are not longer using the main route table of the VPC
4. Add two routes in the newly created Route Table:

```bash 
# VPC CIDR Range 10.16.0.0/16
##-------------------------------------------------------------------------V
#AZ     Db (private)       App (private)      Web (public)
AZ-A    10.16.16.0/20     10.16.32.0/20      10.16.48.0/20
                                                 |-----------------------------> VPC Router----------> igw-xxx # all 3 public subnets connect to same igw since
                                                                                  Ʌ  Ʌ                         # internet gateway is region resilient
AZ-B    10.16.80.0/20     10.16.96.0/20      10.16.112.0/20                       |  |
                                                 |--------------------------------|  |
                                                                                     |
AZ-C    10.16.144.0/20    10.16.160.0/20     10.16.176.0/20                          |
                                                 |-----------------------------------|
##-------------------------------------------------------------------------Ʌ

#---------------------------------------------------------------------V
# default route table for public subnet "Web" when this subnet is created 
#and this route table is auto assocaited with this subnet by default

# Destination              Target
10.16.0.0/16               local
#---------------------------------------------------------------------Ʌ

#---------------------------------------------------------------------------------V
# New route table created and associated by you for public subnet "Web"

# Destination              Target
10.16.0.0/16               local
0.0.0.0/0                  igw-xxx
#---------------------------------------------------------------------------------Ʌ
```
You also need to enable "Enable auto assign public IPv4/IPv6 address" in each subnet's setting


## NAT Gateway

* NAT Gateway is **not regionally resilient** (`IGW` is), NATGW is AZ level resilient which means you needs to add a NATGW per AZ (when you create a NATGW, you need to specify the subnet you want it to work at)
* EC2 instances (no public ip) that route to NATG can only initial outbound connections, outsite internet cannot initial inbound connections to those instances
* Exam Question on NAT Instance (An EC2 Instance), you need to **disable "Source and Destination Checks"** (source or destination needs to be the EC2 instance itself and this setting is on by default) to make an EC2 instance eligible to be an NAT instance first, then SSH to the instance and run the following command:
```bash
# Turning on IP Forwarding
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Making a catchall rule for routing and masking the private IP
sudo iptables -t nat -A POSTROUTING -o ens5 -s 0.0.0.0/0 -j MASQUERADE
```
Since NAT instance is an EC2 instance so it can use security group while NAT gateway can only use NACLs
* NAT Gateway doesn't work with IPv6


## Create Private Subnet with NAT Gateway

To Create a NAT Gateway in Console UI, you select the public "Web" subnet (sn-web-a) to put NAT Gateway inside the public subnet, NAT Gateway has to be put inside public subnet
and then assign an Elastic IP to NAT Gateway. Note that NAT Gateway gets an public elastic IP and a private address which is controlled by the route tables which in turn will be looked upon by VPC router
For the A4L architecture, you need to create 3 NAT Gateway instances along with 3 route tables

```bash
# VPC CIDR Range 10.16.0.0/16
##--------------------------------------------------------------------------------------------------V
#AZ     Db (private)          App (private)      Web (public)
AZ-A    10.16.16.0/20         10.16.32.0/20      10.16.48.0/20
             |                     |                    |-----------------------------> VPC Router----------> igw-xxx
             |------> nat-a <------|                                                    Ʌ  Ʌ  Ʌ
                       |                                                                |  |  |
                       |----------------------------------------------------------------|  |  |                                              
                                                                                           |  |
AZ-B    10.16.80.0/20     10.16.96.0/20      10.16.112.0/20                                |  |
             |                     |            |------------------------------------------|  |     
             |------> nat-b <------|                                                       |  |
                       |                                                                   |  |
                       |-------------------------------------------------------------------|  |                                                       |
AZ-C    10.16.144.0/20    10.16.160.0/20     10.16.176.0/20                                   |
             |                     |            |---------------------------------------------|
             |------> nat-c <------|                                                          |
                       |                                                                      |
                       |----------------------------------------------------------------------|
##-------------------------------------------------------------------------------------------------Ʌ


#---------------------------------------------------------------------------------V
# one of three new route table says rt-a created and associated with private subnet e.g sn-web-a:

# Destination              Target
10.16.0.0/16               local
0.0.0.0/0                  nat-a
#---------------------------------------------------------------------------------Ʌ

# then you need to assocate subnets with route table:
# sn-db-a, sn-app-a with rt-a 
# sn-db-b, sn-app-b with rt-b
# sn-db-c, sn-app-c with rt-c
```


## Network Access Control Lists (NACL)

* Stateless and created on **VPC** level and  apply on **subnet** level 
* traffic between same subnet won't be impacted by NACLs (compared to SG, traffic between same subnet will be impacted)
* Allow explicit deny (compared to SG which supports explicit deny)

Default NACLs (automatically created when a subnet is created) for inbound and outbound as below shows:

```bash
# Inbound Rules (2)
# Rule Number                Type           Protocol      Port range     Source        Allow/Deny
# 100                      All trafic         All            All        0.0.0.0/0         Allow   <----------default one which means a subnet is implicitly allow any requests
# *                        All trafic         All            All        0.0.0.0/0         Deny    <----------always in there and cannot be removed

# Outbound Rules (2)
# Rule Number                Type           Protocol      Port range     Source        Allow/Deny
# 100                      All trafic         All            All        0.0.0.0/0         Allow   <----------default one which means a subnet is implicitly allow any requests
# *                        All trafic         All            All        0.0.0.0/0         Deny    <----------always in there and cannot be removed
```

Rules are processed **in order**, lowest rule number first. Once a match occurs, processing stops. "*" is an implicit deny if nothing else mathes.
Each network ACL also includes **a rule whose rule number is an asterisk (*). This rule ensures that if a packet doesn't match any of the other numbered rules, it's denied. You can't modify or remove this rule**. Note that default NACLs for a VPC has a default "implict allow" rule (which has rule number 100) which reduces admin overhead while custom NACLs is different.

Custom NACLs is created for a specific VPC and are **initially assoicated with no subnets**, and there is no "implicit allow"  any more:
```bash
# Inbound Rules (1)
# Rule Number                Type           Protocol      Port range     Source        Allow/Deny
# *                        All trafic         All            All        0.0.0.0/0         Deny 

# Outbound Rules (1)
# Rule Number                Type           Protocol      Port range     Source        Allow/Deny
# *                        All trafic         All            All        0.0.0.0/0         Deny 
```
so Custom NACLs are **implicit deny** by default


## Security Group

* Apply on instance level (technically ENI level) compared to the NACLs which applys on subnet level
* Statefull, so allow request = allow response
* No explicit deny (compared to NACLs which supports explicit deny), only **allow** or **implicit deny**
* Cannot block specific bad IP
* Support IP/CIDR, logical resource, other security group and itself. When sg is used in Source, it means "any instace in the sg"
* Attched to Elastic network interfaces (ENI, which is associated with an instance) not instances itself (even if the UI shows it this way). Remember an EC2 instance can have one primary private IP (used in the internet gateway) and multiple ENIs (each ENI has one private IP).

```bash
# Inbound Rules (1)

#   Type           Protocol      Port range     Source       Description-optional
# Custom TCP         TCP            443         sg-xx               xxx

```

## EC2

* AZ Reliant: an EC2 is placed in an AZ, if that AZ fails, all EC2 instance in that AZ fails
* Since EC2 is AZ Reliant, all related network, persistent storage (EBS etc) are per AZ. So if an AZ in AWS experience major issues, it impacts all of those things
* Restart an EC2 instance, the instance will be on the same host; Stop an EC2 instance and start it (different than restarting) will result the instance in a different host (still in same AZ)


# EC2 Placement Group

Background knowledge: Each rach has its own network and power source

1. `Cluster`: instances are placed into **same rack** (same AZ of course) in datacentre, often same host, so lowest latency and 10Gbps speed communication between instances. However, it offer little resilient as if the physical fails all instances are down. When you create a cluster group, you don't specify an AZ, it is the first instance being placed into a AZ then the AZ is locked for further placements. Only certain supported instance type can be used with this cluter placement group

2. `Spread`: maximize resilient by placing each instance into **distict rack** across mutiple AZs (each instance runs from a different rack. Inside a same AZ, each difference still run on different rack). For example, you want to lauch 6 instances, then 3 (a, b, c) instances are in AZA, 3 (d, e, f) instances are in AZB, for instance in same AZ, each instance are placed into distict rack physical hardware, so that if one fails e.g `a`'s rack's physical machine got power down, the rest of 2 instances (b, c) are still working. Spread placement has a limit of **maximum 7 instance per AZ**. Not supported for Dedicated Instances or Hosts

3. `Partition`: similar to Spread, but it is **maximum 7 partition group per AZ**, inside a artition group, you can lauch as many as instance you want. Each partition group has its  own rack. You can choose the partition to lauch an instance or auto placed


# EC2 Categories

example instance type: `R5dn.8xlarge`:
`R` is the *Instance Family*, `5` is *Instance Generation*, `8xlarge` is *Instance Size*. `dn` is *Additional Capabilities* (`d` means dense for NVMe, `n` means network optimized )


* **General Purpose**: default, diverse workloads, equal resource ratio:

  `A1`, `M6g`: 'A1' (Graviton), 'M6g' (Graviton 2), Graviton is a family of 64-bit ARM-based CPUs designed by AWS
  `T3`, `T3a`: 'T' means Turbo (Burstable), cheaper assuming low levels of usage, with occasional peaks.
  `M5`, `M5a`, `M5n`: 'M' Stands for "Main"/ "General Purpose", steady state workload alternative to T3

* **Compute Optimized**: Machine Learning, gaming, scientific modelling:

  `C5`, `C5n`: general machine learning, gaming server

* **Memory Optimized**: large in-memory dataset process and database workloads

  `R5`, `R5a`: real time analytics, in-memory caches, certain DB application (in-memory operation)
  `X1`, `X1e`: large scale in-memory applications, lowest $ per GB memory in AWS

* **Storage Optimized**: massive IO operations per second, data warehousing, analytics workloads

   `I3`, `I3en`: high performance NVMe
   `D2`: dense storage (HDD), lowest price disk throughput
   `H1`: high throughout balance CPU/Memory

* **Accelerated Computing**: Harware GPU, field programmable gate arrays (FPGAs)


```bash
ssh -i "A4L.pem" ec2-user@ec2-18-234-163-215.compute-1.amazonaws.com
```

## Instance Metadata

```bash
curl http://169.254.169.254/latest/meta-data/
# ami-id
# ami-launch-index
# ami-manifest-path
# block-device-mapping/
# events/
# hostname
# iam/
# identity-credentials/
# instance-action
# instance-id
# instance-life-cycle
# instance-type
# local-hostname
# local-ipv4
# mac
# managed-ssh-keys/
# metrics/
# network/
# placement/
# profile
# public-hostname
# public-ipv4
# reservation-id
# security-groups
# services/
```

```bash
curl http://169.254.169.254/latest/meta-data/public-ipv4
curl http://169.254.169.254/latest/meta-data/public-hostname

wget http://s3.amazonaws.com/ec2metadata/ec2-metadata  # a tool let you interact with Metadata easier
chmod u+x ec2-metadata

ec2-metadata --help
ec2-metadata -v  # same as curl http://169.254.169.254/latest/meta-data/public-ipv4
ec2-metadata -p  # query public host name
```

## EC2 Bootstrapping

Only run once on Launch, and you can access user data at `http://169.254.169.254/latest/user-data`


## EC2 Instance Role

Create an Instance Role (a normal role) and attach it to an EC2 instance via: -> Security -> Modify IAM Role. Note that when you create a role use Console UI, an **Instance Profile** is created, and when you attach the role to the EC2 instance, you actually attach the instance profile with the EC2 instance, UI makes you think you attach "role" to it but behind the scene it is the instance profile attached to it.

An application on the instance retrieves the security credentials (automatically renewed) provided by the role from the instance metadata:

```js
// Inside an EC2 instance (AWS's own AMI) that you attach an instance role to it

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/YourInstanceRole

/* this credentials will be automatically renewed when it is about to expiry 
{
  "Code" : "Success",
  "LastUpdated" : "2025-02-16T13:58:09Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "xxx...",
  "SecretAccessKey" : "yyy...",
  "Token" : "zzz...=",
  "Expiration" : "2025-02-16T20:32:20Z"
}
*/
```


## EC2 Purchase Options

* On-Demand: most common way, bill per sec
  You can purchase on-demand capacity reservations (minimum commitment is 14 days) to avoid the situation when AWS doesn't have enough readily available compute power to fulfill your request for the chosen instance type, when you reserve capacity for on demand, you are charged by bill er sec and you are still charge when the instance is stopped

* Spot Instance: Bid like Auction, once the price falls to the minimum you set, it starts to run and it can be stopped by aws when the price goes up

* Reserved: get discount for committing to run an instance for 1 up to 3 years, you will be charged even you don't use/start these instances.

* Dedicated Instance: no other AWS Account will run an instance on the same Host, but other instances (both dedicated and non-dedicated) from the same AWS Account might run on the same Host. Just like you're hiring a small laptop. if you stop/restart that laptop, next time, you're hiring another laptop and only you can use this laptop.

* Dedicated Host: No Per Instance Charge, since you already purchase the whole physical server; used for requirements of licensing based on sockets/cores. Similar to dedicated instance, but guess what, that laptop is yours, forever, as long as you pay the bill. You stop/restart that laptop, next time you still boot on the same laptop. But usually this laptop is huge, and the price is a lot more expensive


## Elastic Block Store (EBS) and Instance Store

Ephermeral Storage: `Instance Store`
Persistent Storage: `EBS`

https://stackoverflow.com/questions/74678898/what-does-ec2-store-and-why-does-it-even-need-a-storage-solution-like-ebs-or-ins
Think of an Amazon EC2 instance as a normal computer. Inside, there is CPU, RAM and (perhaps) a hard disk.

When an EC2 instance has a hard disk, it is called `Instance Store` and it behaves just like a normal hard disk in a computer. However, when you turn off the instance and stop paying for it, the EC2 instance can give that computer to somebody else. Rather than giving your data to somebody else, **the disk is erased**. So, anything you stored on Instance Store is gone.

If you want to **keep your data** so that it is still there when you turn on the instance in future, it needs to be stored on a network disk and that is what Amazon EBS provides. Think of it a bit like **an USB drive that you can plug into one computer, then disconnect it and plug it into another computer**. However, rather than being a physical device, it uses a storage service that keeps multiple copies of the data (in case a disk fails) and lets you modify the size of the disk. You are charged based on the amount of storage space assigned and how long the data is kept ("GB-Month").


**Instance Store** volumns are physically attached to EC2 host (offer highest-performance compare to network based gp2 etc), and they are non-persistent so if you stopped, hibernated, or terminated your EC2 instance, the instance store data is gone (reboot is fine). **Instance Store can only be attached at launch time (exam question)** while other EBS can be attached after EC2 instance is created

**EBS** volume is attached over network, not within EC2 host. EBS volumn still persist when the EC2 instance get terminated. An EBS volumn can only be attached to **only one EC2 instance at a time**.

Every EC2 instance has a root EBS volumn (to store operating systmes) created when the EC2 instance is created. Note that root volumn will be deleted when you delete the corresponding EC2 instance but any addtional volumns you attach to the instance will still be available

**Note that An EBS volumn can only be assigned to EC2 instances which are in the same AZ**, you have to use snapshot feature to copy an EBS volume to another EC2 instance in a different AZ or region

There are 3 main category of EBS:

* **General Purpose SSD volumes**: `gp2` (default for boot volumn), `gp3`, which is "credit allocated" storage, all volumes get an inital 5.4 million IO credit (a credit can do IO operation 16kb/s), which can run 30 mins at 3000 IOPS (one IOP is 16kb in one second which is the block size). Note that buckets get refilled with minimum 100 IO Credits per second regardless of volume size, and every volumn get refills with 3 IO credits per second, so a 100Gb volumn 300 credits per sec (which is called baselne rate, which is 3 times the GB size of the volumn), so you can see that if you have a very large e.g 1TB, the volumn won't run out with credits, so it always achieve baseline performance (credits comsumed is roughly the same as credits refilled), but volumn is not limited to run below baseline rate, **gp2** can take up to maximum **16,000 IO credits per second**

`gp3` (which will be default soon) is cheaper than `gp2` and 4x fastern max throughput vs `gp2`- 1,000MB/s VS 250 MB/s, gp3 removes the credit architecture, gp3 is standardise to 3000 IOPS regarding of volumn size 

* **Provisioned IOPS SSD Volumes** (highest performance EBS and support multi-attach): `io1`, `io2`, IOPS can be adjusted independently of size, up to 64,000 IOPS per volume (4 x GP2/3), up to 1000MB/s throughput, great for database workload. And `BlockExpress` takes it to next level by 4x. There is still maximum size to performance ratio: `io1`: 50 OPS/GB, `io2`: 500 OPS/GB, `BlockExpress`: 1000 OPS/GB, note that these maximum value mutiple by GB size is capped by the 64,000 IOPS per volume. If you want a small volume but have high performance, io1 and io2 is the way to go.

note that multi-attach can attach a EBS volume to multiple EC2 instances (maximum 16 EC2 instances) in the **same AZ**

* **Hard disk drive (HDD) volumes**: `st1`, `sc1` (Cold HDD volumes, cheaper), HDD cannot be used as boot volumes, they are not good for random access but good for large sequential data. st1 supports maximum 500 IOPS (500MB/s, one IOP is 1MB in HDD not 16KB like SSD), and it has similar credit architecture like gp2. sc1 (lowest price of all EBS option) supports maximum 250 IOPS


## EBS Snapshots

We can copy an EC2's EBS volume and restore it into another EC2 instance (in different AZ or region)'s new volume. Snapshot restore lazily, which means you have to use some admin tool to force a read of all data before using it in Production otherwise, IO operation will be "async await" and takes some time to read data for users. 
**Fast Snapshot Restore (FSR)** allows you to immediate restore like force read in every block of the new volumn, which can save you overhead of using admin tool
 

## EBS Encryption

* Each volumnuses 1 unique DEK, snapshots and future restored volume uses the same DEK but if you create another volume, you will be using a different DEK


## Create Custmized AMI

AMI itself doesn't store EBS volume, it reference Snapshots which are stored in Amazon S3, in S3 buckets that you can't access directly.  

## Attach EBS to EC2 Practice

```bash
#1
$ lsblk
NAME      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
xvda      202:0    0   8G  0 disk 
├─xvda1   202:1    0   8G  0 part /
├─xvda127 259:0    0   1M  0 part 
└─xvda128 259:1    0  10M  0 part /boot/efi
xvdf      202:80   0  10G  0 disk    ## <-----------------------the new volumn attached to an EC2 instance

#2
$ sudo file -s /dev/xvdf
/dev/xvdf: data   ## <------------"data" mean there isn't any file system on this block device, you can only mount file systems on Linux so we need to create a FS on top of it

#3
$ sudo mkfs -t xfs /dev/xvdf
meta-data=/dev/xvdf              isize=512    agcount=4, agsize=655360 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

#4
$ sudo file -s /dev/xvdf
/dev/xvdf: SGI XFS filesystem data (blksz 4096, inosz 512, v2 dirs)

$ sudo mkdir /ebstest
$ sudo mount /dev/xvdf /ebstest  ## <------------------------------

#5
$ df -k
Filesystem     1K-blocks    Used Available Use% Mounted on
devtmpfs            4096       0      4096   0% /dev
tmpfs             486128       0    486128   0% /dev/shm
tmpfs             194452     444    194008   1% /run
/dev/xvda1       8310764 1627904   6682860  20% /
tmpfs             486132       0    486132   0% /tmp
/dev/xvda128       10202    1310      8892  13% /boot/efi
tmpfs              97224       0     97224   0% /run/user/1000
/dev/xvdf       10420224  105704  10314520   2% /ebstest  ## <-------------------------------

#6
$ sudo reboot  # restart the instance and if we connect to the same EC2 instance again we cann't see the file system associated with EBS volume when run `df -k`
# because by default EBS volume are not automatically mounted (EBS volume is still automatically attached to the EC2 instance)

#7
$ sudo blkid  # select the unique volumn id 9657018b-98fd-438d-ae98-e1fe0c01544e, /dev/xvda1 is the boot volume
/dev/xvda128: SEC_TYPE="msdos" UUID="1883-E9E2" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="f39eccca-429b-4ad0-8537-7e3b00bace36"
/dev/xvda127: PARTLABEL="BIOS Boot Partition" PARTUUID="c8dfb03e-f4d6-4010-9e9a-141a2e5596ed"
/dev/xvda1: LABEL="/" UUID="9d0c119b-7697-47bd-99a8-48ecf6e02f0d" BLOCK_SIZE="4096" TYPE="xfs" PARTLABEL="Linux" PARTUUID="55732cf4-e5b5-4052-9572-ad256392a887"
/dev/xvdf: UUID="9657018b-98fd-438d-ae98-e1fe0c01544e" BLOCK_SIZE="512" TYPE="xfs"

#8
$ sudo nano /etc/fstab  # an a new entry that contains the unique UUID above, fstab means "file system table"

#9
$ sudo mount -a   # mount all File Systems in the fs table
```


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


## Elastic Load Balancers (ELB)

* ~~Class Load Balance (CLB)~~: 1 SSL per CLB, lacking features etc, do not use
* Application Load Balancer (`ALB`): HTTP, HTTPS, WebSocket
* Network Load Balacner (`NLB`): TCP, UDP, TLS


learn dns first
```
                SubnetA1                                            SunbetA2             
                                                                                                
        ┌──────────────────────────────────┐                  ┌────────────────────────────────┐
        │                                  │                  │                                │
        │                                  │                  │                                │
        │                                  │                  │                                │
        │                                  │                  │                                │
AZ-A    │                                  │                  │                                │
        │                                  │                  │                                │
        │                                  │                  │                                │
        │                                  │                  │                                │
        │                                  │                  │                                │
        │                                  │                  │                                │
        │                                  │                  │                                │
        └───────────────▲──────────────────┘                  └────────────────────────────────┘
                        │                                                                       
                        │                                                                       
                        │                                                                       
                        │                                                                       
                        │                                                                       
                        │                                                                       
                        │                                                                       
                        │                                                                       
                        │                                                                       
                   ┌────┼────┐                                                                  
                   │         │                                                                  
           ELB1    │         │                                                                  
                   │         │                                                                  
                   └─────────┘                                                                     
```


## Route53

`DNS Zone`     : a database, for example  *.netflix.com (that's the zone) containing records
`ZoneFile`     : the "file" storing the zone on disk
`Name Server`  : a DNS server which hosts 1 or more Zones by storing 1 or more ZoneFiles
`Authoritative`: contains real/genuine record (boss)
`Non-Authoritative`/ `Cached`: copies of records/zones that saved as cached e.g. you internet provider or router can be this category as you have vistied some sites before


Q: Why not let TLD such .com NS providing IP address directly, why it delegate the job to Authoritative servers?

A: Separating out the Authoritative servers from the TLD servers allows the owner of an individual domain to make changes to their domain's records without having to involve the TLD maintainers in any way. Say that we own `example.com`, which has an authoritative server at `ns.example.com`, and we want to set up a new subdomain at `blog.example.com`. Since we maintain our own authoritative server, all we have to do to set up the blog subdomain is add a new A record to our authoritative server.


DNS records:

┌─────────┬─────────────────┬─────────────────┬────────┐                  ┌─────────┬─────────────────┬─────────────────┬────────┐
│  Type   │      Name       │   IP Address    │   TTL  │                  │  Type   │      Name       │   IP Address    │   TTL  │
├─────────┼─────────────────┼─────────────────┼────────┤                  ├─────────┼─────────────────┼─────────────────┼────────┤
│   A     │   example.com   │   12.34.56.78   │  7200  │                  │  AAAA   │   example.com   │    ipv6 ip      │  7200  │
└─────────┴─────────────────┴─────────────────┴────────┘                  └─────────┴─────────────────┴─────────────────┴────────┘


┌─────────┬─────────────────┬─────────────────┬────────┐                  ┌─────────┬─────────────────┬─────────────────┬────────┐
│  Type   │      Name       │    Alias To     │   TTL  │                  │  Type   │      Name       │    Alias To     │   TTL  │
├─────────┼─────────────────┼─────────────────┼────────┤                  ├─────────┼─────────────────┼─────────────────┼────────┤
│  CNAME  │ www.example.com │   example.com   │  7200  │                  │  ALIAS  │    example.com  │    example.net  │  7200  │
├─────────┼─────────────────┼─────────────────┼────────┤                  └─────────┴─────────────────┴─────────────────┴────────┘
│  CNAME  │ ftp.example.com │   example.com   │  7200  │
├─────────┼─────────────────┼─────────────────┼────────┤                  both CNAME and ALIAS records are stored in Authoritative NS, not TLD
│  CNAME  │ www.example.com │ lb.example.net  │  7200  │
└─────────┴─────────────────┴─────────────────┴────────┘

┌─────────┬─────────────────┬──────────────────┬────────┐                  
│  Type   │      Name       │      Value       │   TTL  │                
├─────────┼─────────────────┼──────────────────┼────────┤                 
│   NS    │   example.com   │ dns1.example.com │  7200  │                
├─────────┼─────────────────┼──────────────────┼────────┤   
│   NS    │   example.com   │ dns2.example.com │  7200  │                
└─────────┴─────────────────┴──────────────────┴────────┘              



                        Root NS                                                  
                        ┌─────┐                                                  
                        │     │                                                  
                        │     │                                                  
                        │     │──────────────┐                                   
                        │     │              │                                   
                        │     ◄───────────┐  │    TLD DNS Server                 
                        │     │         ┌─┼──▼──────────────────────────────────┐
  Local DNS Server      └▲────┘         │  (NS,example.com, dns1.example.com)   │   both NS and A records are returned to the local DNS Server
      ┌──────┐           │              │  (A,dns1.example.com, 212.212.212.1)  │
      │      │           │              └───▲───────────────────────────────────┘
      │      │◄──────────┘                  │
      │      │                              │                                    
      │      │◄─────────────────────────────┘                                    
      │      │                                                                   
      │      │◄────────────┐                                                     
      └───▲──┘             │                                                     
          │                │ Authoritative DNS Server (IP Addr: 212.212.212.1)   
          │               ┌▼────────────────────────────────────────────────────┐
    Host  │               │ (CNAME, Name: www.example.com, Alias: example.com)  │
┌────────────┐            │ (CNAME, Name: ftp.example.com, Alias: example.com)  │
│            │            │ (A, example.com, 12.34.56.78)                       │
│            │            └─────────────────────────────────────────────────────┘
└────────────┘            


`CNAME` is __invalid__ for naked/apex domains.  An **apex** domain is a custom domain that does not contain a subdomain, such as `example.com`, `blogs.example.com` is not apex domain
`ALIAS` can be used for both apex and non-apex domains and it is implemented by AWS and outside of the usual DNS standard.
check https://www.ibm.com/think/topics/alias-vs-cname for more details such as how ALias records can delegte the second query to Authoritative NS




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


IAM role contains two kinds of polilcies: **Trust policy**: Who can assume the IAM role ; **Permission policy**: What does the role allow once you assume it

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

When you create an new aws account in using AWS Organization, `OrganizationAccountAccessRole` (OAAR) that has admin rights in that account is automatically created, if you invite an existing aws account (note that you will be using management account to send invitation to the account) to join the organization, then you have to manually create the role so that management account can assume it. Management account doesn't have any role such as OAAR being created.

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


## CloudWatch (for metrics) and CloudWatch Logs (for logging)


`Namespace`  :  container for metric e.g `AWS/EC2` & `AWS/Lambda`, `AWS` prefix is for namespace created by AWS
`Datapoint`  :  Every datapoint has a Timestamp, Value and (optional) an unit of measure, e.g EBS IOPS, a datapoint is : { TimeStamp: "20240110T09:27:33", Value: 1000, unit: "count"}
`Metric`     :  time ordered set of data points, e.g EC2's `CPUUtilization`, `DiskWriteBytes`
`Dimension`  :  name/value pair e.g `Name=InstanceId, Value=xxxx` to differentiates between instances, e.g instanceA's `CPUUtilization` VS instanceB's `CPUUtilization`
                AWS adds extra dimension e.g `ImageId`, `InstaceType`, `AutoScalingGroupName` for built-in metric `CPUUtilization`
`Resolution` :  granularity, standard is 60s granularity, high is 1s
`Retension`  :  for a given Resolution, AWS stores them differently. Less than 1min granularity, data is retained for 3hours; 1min for 15 days; 5mins for 63 days, 1hour for 455 days

A **Log Stream** is a series log events from the **same source/instance**, a **Log Group** consists multiple Log Stream

```bash
# download agent locally
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard # accept all defaults, until default metrics .. pick advanced

# then when asking for log files to monitor
/var/log/httpd/access_log   # enter this file name twice as file and log group are the same
## ... configure log group, log stream with instance id and so on

# Config can also be stored in /opt/aws/amazon-cloudwatch-agent/bin/config.json and stored in SSM but you need to attach an instance role with sufficient permission

# Load Config and start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-linux -s
```

## CloudTrail

* Logs AWS's own Product's event
* **90** days stored by default in **Event History**, enabled by default, which has no cost for 90 days
* Log **Management Event** (Terminate an EC2 instance etc) and **Data** Events (an obejct is uploaded to S3), by default only logs Management Events


























## end of learn cantrill

===================================================================================================================================================== 

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

1. Creates a **service role** e.g **CloudFormationServiceRole** that specify CloudFormation has permission to create EC2 instance. Its trust entity is like:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
        "Sid": "",
        "Effect": "Allow",
        "Principal": {
            "Service": "cloudformation.amazonaws.com"   // <-----------"Service" under "Principal" means it is a service role
        },
        "Action": "sts:AssumeRole"
    }
  ]
}
```

then attach e.g `AmazonEC2FullAccess` permission


2. The senior developer creates a **user role** where you specify a trust policy that Paul can assume rule

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
below is a permission that is attached to the user role above

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

  