### creating a 3 tier networking architecture for a wordpress website ###
# STEP 1, CREATE A VPC WITH.

  Cidr_block            = 10.0.0.0/16
  Enable_dns_hostnames  = true
  Enable_dns_support    = true
  Tags                  = practice-vpc

# STEP 2, CREATE 6 SUBNETS, 2 PUBLIC AND 4 PRIVATE.

 - Public subnets for web server
   VPC_ID                  = practice-vpc
   Cidr_blocks             = [10.0.0.0/24, 10.0.1.0/24]
   Availability_zones      = [us-east-1a ,us-east-1b]
   Map_public_ip_on_launch = true
   Tags                    = [public-web-subnet-AZ1, public-web-subnet-AZ2]

 - Private subnets for APP server
   VPC_id                  = practice-vpc
   Cidr_blocks             = [10.0.3.0/24, 10.0.4.0/28]
   Availability_zones      = [us-east-1a ,us-east-1b]
   Tags                    = [private-app-subnet-AZ1, private-app-subnet-AZ2]

 - Private subnets for database server
   VPC_id                  = practice-vpc
   Cidr_blocks             = [10.0.2.0/24, 10.0.5.0/28]
   Availability_zones      = [us-east-1a ,us-east-1b]
   Tags                    = [private-db-subnet-AZ1, private-db-subnet-AZ2]

# STEP 3, CREATE AN INTERNET GATEWAY AND ATTACH TO THE ABOVE VPC.

   Vpc_id  = practice-vpc
   Tags    = practice-igw

# STEP 4, CREATE 2 NAT GATEWAY IN THE PUBLIC SUBNETS [AZI & AZ2].
   - Select allocate elastic ip address and add it to the private 
     route tables

# STEP 5, CREATE 3 ROUTE TABLES, 1 PUBLIC AND 2 PRIVATE ROUTE TABLES.
  1. Public route table for web subnets.

   - Vpc_id        = practice-vpc
   - Route         =
      Destination = 0.0.0.0/0
      Target      = practice-igw
   - Tags          = public-RT

  NOTE: Associate the 2 public-web-subnets to the public route table 

 2. Private route table  AZ1
   - Vpc_id        = practice-vpc
   - Route         =
      Destination = 0.0.0.0/0
      Target      = NAT Gateway-AZ1
   - Tags          = pri-RT-AZ1

  NOTE:Associate the  pri-DB-sn1 and the pri-app-sn1 to the private route
  table AZ1

  3. Private route table AZ2
   - Vpc_id        = practice-vpc
   - Route         =
      Destination = 0.0.0.0/0
      Target      = NAT Gateway-AZ2
   - Tags          = pri-RT-AZ2

  NOTE:Associate the  pri-DB-sn2 and the pri-app-sn2 to the private route table AZ2

# STEP 6, CREATE THE SECURITY GROUPS.
    1. ALB security group
        - port: 80 & 443 / source: internet gateway

    2. Webserver security group
        - port: 80 & 443 / source: ALB security group
        - port: 22 / source: myIP

    3. Database security group
        - port: 3306 / source: Webserver secrity group
      
    4. Elastic file system security group
        - port: 2049 / Websrever security group / Elastic file system security group
        - port: 22 / Webserver security group

# STEP 7, CREATE A KEYPAIR.
    1. Navigate to the EC2 console an keypair
     - select create key pair
     - Name - wordpress-key
     - check RSA AND .pem
     - click create key pair and make sure you save the private key.

# STEP 8, CREATE AN RDS INSTANCE.
    1. Navigate to RDS in your console and select subnet group
        - click on create DB subnet group
        - Name: practice-DB-SG 
        - Description: practice-DB-SG
        - VPC: practice-vpc
        - Availability Zones: select [a & b]
        - Subnets: select the 2 database subnets.

    2. Navigate to RDS service in the console and select database
       - click on create database
       - select: standard create
       - Engine: MYSQL
       - Version: lastest 
       - Template: Dev/test
       - Deployment option: multiAZ
       - DB instance identify: practice-rds-db
       - master username: Anthony
       - Password: myfirstname
       - DB instance type: Burstable classes
       - Storage type: General purpose
       - Allocated storage: 30gb
       - Storage Autoscaling: Enable
       - VPC: practice-vpc
       - DB subnet group: practice-DB-SG
       - Security group: database-SG
       - Availability zone: select 1b
       - Database authentication: password authentication
       - click on additional configuration
           initial database name:Application-db
       - Click create.

# STEP 9, CREATE AN ELSTIC FILE SYSTEM.
   1. Navigate to EFS service in th console
      - click create file system and click customize
      - Name: practice-EFS
      - Tag: Dev-EFS and click next
      - Network: practice-vpc
      - Mount-target-1:
         Availability Zone: 1a
         Subnet: DB-SN-1
         Security group: EFS-SG
      - Mount-target-2
          Availability Zone: 1b
         Subnet: DB-SN-2
         Security group: EFS-SG 
      - click next and then create.

# STEP 10, CREATE A SETUP SERVER (EC2 INSTANCE)  
   1. Navigate to the EC2 service in the console
       - click on launch an instance
       - Name:jump-server
       - AMI: Amazon linux 2
       - instance type: t2.micro
       - select a keypair
       - VPC: practice-vpc
       - subnet: pub-web-sn1
       - security group
          FrondEnd ALB-SG
          Webserver-SG
       - click create

# STEP 11, SSH INTO THE SETUP SERVER.
   1. Run all the commands in the setup-server script


# STEP 12, CREATE AN APPLICATION LOAD BALANCER
   1. Navigate to the EC2 console and select target group
      - click create target group
      - type = instance
      - target group name = wordpressdb-tg
      - VPC = practice-vpc
      - protocol / port = HTTP/80
      - health check = HTTP
      - health check path = /
      - click next and create target group

  2. Navigate to the EC2 console and select load balancers
     - click create load balancer
     - select apllication load balancer
     - name = wordpress-ALB
     - internet-facing
     - IPv4
     - VPC = practice-vpc
     - select AZ1(public-web-subnetAZ1) & AZ2(public-web-subnetAZ2) 
     - select ALB security group
     - listener protocol/port = HTTP/80
     - default action = target group(wordpressdb-tg)
     - then click create load balancer,



# STEP 13, CREATE A LAUNCH TEMPLATE
   1. Navigate to the EC2 console and select launch templates
      - click create a launch template
      - name = wordpress-LT
      - version = wordpress-LT-v1
      - auto scaling guidance, check
      - TAG = key(Name), value(wordpress-LT)
      - AMI = Amazon Linux
      - instance type = t2.micro
      - key pair = wordpress-keypair
      - security group = webserver security group
      - resource tag = ASG-Webserver
      - click the advance drop down and paste the
        # Auto scaling user-data in the user data section 
      - click create launch template

# STEP 14, CREATE AN AUTO SCALING GROUPS
   1. Navigate to the EC2 console and select auto scaling
      - click create auto scaling 
      - name = wordpress-ASG
      - select a launch template = wordpress-LT
      - VPC = practice-vpc
      - select the 2 pravite app subnets
      -  then click next
      - select attach to an exiting load balancer
       - choose from your load balancer target group, check
       - click the drop down, select taget groups and select(wordpressdb-tg)
       - health check type = ELB
       - monitoring = Enable, then click next
       - group size 
         - Desire capacity = 2
         - Minimum capacity = 1
         - Maximum capacity = 4
      - scaling policy = None
      - clcik add notification
        - click create a Topic 
          - name = wordpress-topic
          - input your email and check all the box
          - click next
        - tag = key(name), value(webserver)
        - review and launch
        

# STEP 15, REGISTER THE TARGETS
   1. Navigate to the EC2 conlose and select target group
      - click on the wordpress-db-tg
      - select target and clcik on register targets
      - select the servers that auto scaling group created 
      - click include as pending below
      - then click register pending targets

# STEP 16, REGISTER A DOMAIN NAME AND CREATE A RECORD SET IN ROUTE 53
   1. Navigate to the Amazon Route 53 console
      - under register domain, type your propose domain name and click check
      - click continue and fill out the form and then click register
  2. Create a record.
     - Navigate to the Amazom route 53 console and select hosted zone
     - click on the domain name we just registered  
     - select create record
     - subdomain = www
     - record type = A record and check the Alias box
     - route traffic = Appliocation and classic load balancer
     - region = same region we created our ALB
     - routing policy = simple routing
     - click create records
