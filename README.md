# Deploy a Dynamic WordPress Website on AWS (Host a WordPress Website on AWS)

![3-tier VPC Architecture](image.png)
`3-tier VPC Architecture`

Welcome Cloud advocates to another journey of cloud exploration using the provider AWS. In today’s project, we will use AWS management console to deploy a dynamic WordPress website. We are going to use various AWS services (3-tier vpc architecture from scratch with public and private subnets, security groups, NAT gateways, ec2 instances, rds, application load balancer, route53, auto scaling group, certificate manager, efs, and more)

## Step 1: Build a Virtual Private Cloud (VPC) with public and private subnet

We will be using the architecture diagram above to create a 3-tier VPC. In the first tier, we will have public subnet which will hold resources such as NAT Gateway, Application Load balancer. In the second tier, we will have private subnets containing resources such as our web server and in the third tier, we will have other private subnets which will have the RDS database. We will duplicate these subnets across multiple availability zones (AZ) for high availability and fault tolerance. Resources in our VPC will access the internet via internet and NAT gateways using route tables.

### I. Create a 3-tier VPC

  a. From the management console, first, select the region to create the VPC, in our case, it will be **“us-east-1”**
  b. In the Search box, type VPC and select VPC under services
  c. click on **Create VPC**
  d. Give a name (we will be using **"My VPC"**)
  e. In the IPv4 CIDR field, enter an IP address (this is a classless inter-domain routing IP address range), as per our architecture, it will be “**20.1.0.0/16**”
  f. We will leave IPv6 (**No IPv6 CIDR block**) and tenancy (**Default**) unchanged
  g. Click **create VPC**
  h. You can view your VPC by using the filter box and selecting your VPC name. ![View VPC](image.png)

### II. Enable DNS Hostname in your VPC (this will enable it to resolve any domain name specified in a private hosted zone in Route 53)

      a. Select the VPC created in (I) above
      b. Goto Actions -> Edit DNS hostnames
      c. Check “Enable DNS hostnames” box and save changes

### III. Create an Internet gateway for your VPC

      a. On the VPC page, on the left, select Internet Gateways -> Create internet gateway
      b. Give it a name (we will use “My Internet Gateway”) and click Create internet gateway


IV — Attach the internet gateway created in (III) above to your VPC (this allows instances in the VPC to communicate with the internet)

a) Click the Attach button shown on the notification (in green)

b) Under Available VPCs, select your VPC (it will only show VPCs that have no internet gateway attached to it).

c) Click Attach internet gateway. The status will now show Attached



V — Create Public subnets in the 1st and 2nd Availability Zones

a) On the VPC page, select Subnets -> Create subnet

b) Select the VPC where you want to create your subnet from the dropdown (in our project it will be “My VPC”)

c) Under Subnet name, enter a name (we will be using “Public Subnet AZ1”)

d) Under Availability Zone, select “us-east-1a” (according to our reference architecture)

e) Under IPv4 CIDR block, enter “20.1.0.0/24”

f) Click Create subnet

g) Repeat steps (a) to (e) and create the 2nd subnet with name as “Public Subnet AZ2”, availability zone as “us-east-1b” and IPv4 CIDR as “20.1.1.0/24”

h) Filter by your VPC to see the 2 subnets just created


VI — Enable the Auto-assign IP settings for the 2 public subnets (this enables any instance launch in these subnets to be assigned a public IPv4 address

a) Select the 1st subnet (Public Subnet AZ1)

b) Goto to Actions -> Edit subnet settings

c) Under Auto-assign IP settings, enable the check box

d) Scroll down and click save

e) Do the same for 2nd subnet (Public Subnet AZ2)


VII — Create a Public Route Table, add a route and associate the 2 public subnets (this is used by the VPC router to control how traffic data is directed)

a) With your VPC still filtered, you will see one route table (which is the default and is private, generally called the main)

b) On the left side, select Route tables -> Create route table

c) Under Name, give a name (we will use “Public Route Table”)

a) Next, select the VPC where we want to create the route table (we will select “My VPC” from the dropdown)

d) Click on create route table

e) To add a route, under the route tab, click Edit routes -> Add route

f) Under destination, enter “0.0.0.0/0”, under target select your internet gateway (We will select “My Internet gateway”), then save changes

g) To associate the public subnets to this route table, under the Subnet associations tab, click Edit subnet associations

h) At this moment, we will see the 2 non-associated subnets in the VPC, select both Public Subnet AZ1 and Public Subnet AZ2

i) Save associations



VIII — Create our 4 private subnets

Follow the same steps nin (V) above to create the 4 subnets under our VPC with the following details:

a) 1st Private subnet => Name: Private App Subnet AZ1, availability zone: us-east-1a, IPv4 CIDR: 20.1.2.0/24

b) 2nd Private subnet => Name: Private App Subnet AZ2, availability zone: us-east-1b, IPv4 CIDR: 20.1.3.0/24

c) 3rd Private subnet => Name: Private Data Subnet AZ1, availability zone: us-east-1a, IPv4 CIDR: 20.1.4.0/24

d) 4th Private subnet => Name: Private Data Subnet AZ2, availability zone: us-east-1b, IPv4 CIDR: 20.1.5.0/24

e) We can filter by our VPC and see the total 6 subnets.


FACTS:

// Any route table that directly routes traffic to the internet via an internet gateway is considered a public route table

// Any subnet associated with a public route table is considered a public subnet

// A default route table is created when you create a vpc and it’s private by default, it’s also called the main route table

// The default route table routes traffic within the VPC

// Any subnet that is not explicitly associated to a route table is automatically associated with the default route table and hence considered private subnets

// The 4 subnets created in (VIII) above are all private

Step 2: Create Nat Gateways, Private Route tables in the 1st and 2nd Availability Zones

We will create 2 Nat gateways in both Public subnets, create 2 private route table in AZ1 and AZ2 respectively with each having a route to internet via the Nat gateway, then we will associate both private app and private data subnets in AZ1 to route table in AZ1, likewise both private app and private data subnets in in AZ2 to route table in AZ2. We make sure to be in the right region.

I — Create Nat Gateways

a) On the VPC dashboard, on the left side, select Nat Gateways -> Create Nat gateway

b) Give it a name (we will use “Nat Gateway AZ1”) and click Create internet gateway

c) Select the subnet we want to create the Nat Gateway, (it will be Public Subnet AZ1)

d) Connection type, select Public

e) Under Elastic IP allocation ID, click on Allocate Elastic IP (this will allocate an elastic IP to this Nat gateway)

f) Click Create NAT gateway.

g) Repeat steps (a) to (f) to create 2nd Nat Gateway with name: “Nat Gateway AZ2” and subnet: “Public Subnet AZ2”


II — Create Route Tables and add routes

a) On the left side, select Route tables -> Create route table

b) Under Name, give a name (it will be “Private Route Table AZ1”)

c) Select the VPC from the dropdown list you want to create this route table in (in our project, it will be “My VPC”)

d) To add a route, under the route tab, click Edit routes -> Add route

e) Under destination, enter 0.0.0.0/0, under target select a Nat gateway (select “Nat Gateway AZ1” from dropdown), then save changes

f) While still in “Private Route Table AZ1”, to associate the private subnets to this route table, under the Subnet associations tab, click Edit subnet associations

g) At this moment we will see the 4 non-associated subnets in our VPC, select both Private App Subnet AZ1 and Private Data Subnet AZ1

h) Save associations

i) Repeat steps (a) to (h) to create 2nd route table with name: Private Route Table AZ2, route destination 0.0.0.0/0 and target Nat Gateway AZ2, and associate both Private App Subnet AZ2, Private Data Subnet AZ2.

FACTS:

// With this setting, the 4 private subnets can access the internet via the Nat gateways in the public subnets


Step 3: Create Security Groups

We will create 5 security groups; Application Load Balancer (ALB) Security Group, SSH Security Group, Webserver Security Group, Database Security Group and EFS Security Group, EIC Endpoint Security Group, EC2-Instance Security Group. This is to add a layer 3 and 4 (network/Transport layer of the OSI model) security to our infrastructure by controlling what is allowed in and out of a resource within our VPC.

- eic-endpoint-sg: Open outbound traffic on port 22 and use the VPC CIDR for the source destination.

- ec2-instance-sg: Open inbound traffic on port 22 and use the eic-endpoint-sg for the source destination.

I — ALB Security Group (Port: 80, 443 and source: 0.0.0.0/0 for anywhere on the internet)

Here we open port 80 (HTTP) and port 443 (HTTPS) with source coming from anywhere (0.0.0.0/0) on the internet

a) On the VPC dashboard, on the left side, select Security Groups -> Create security group

b) Give it a name (it will be ALB Security Group), use same name as description

c) Under VPC, remove the default selected vpc and select our VPC (“My VPC” from the dropdown list)

d) Under Inbound rules, click Add rule, under Type, select HTTP (port 80), under source, type “0.0.0.0/0”

e) Repeat (d) and add HTTPS (port 443) rule with source “0.0.0.0/0”

f) Scroll down and click create security group

II — SSH Security Group (Port: 22 and source: our private IP address)

Here we open port 22 (SSH) with source coming from our IP address

a) Follow the steps in (I) above with following details, name: SSH Security Group, Type: SSH (port 22), source: select “My IP”

b) Scroll down and click create security group

III — EIC Endpoint Security Group (Outbound Port: 22 and source: our VPC CIDR block, 20.0.0.0/16)

Here we open outbound port 22 (SSH) with source coming from VPC CIDR block (20.1.0.0/16) on the internet

a) On the VPC dashboard, on the left side, select Security Groups -> Create security group

b) Give it a name (it will be EIC Endpoint Security Group), use same name as description

c) Under VPC, remove the default selected vpc and select our VPC (“My VPC” from the dropdown list)

d) Under Outbound rules, click Add rule, under Type, select SSH (port 22), under source, type “20.0.0.0/16”

e) Scroll down and click create security group

IV — EC2-Instance Security Group (Port: 22 and source: EIC Endpoint Security Group, Port: 22 and source: our private IP address)

Here we open port 22 (SSH) with source coming from EIC Endpoint Security Group. This will enable us connect directly to any ec2 instance in any subnet (public and private) in our vpc without the of keypairs

a) Follow the steps in (I) and create a security group with name EC2-Instance Security Group having 2 inbound rules with following details;

Type: SSH (port 22), source: select “EIC Endpoint Security Group”

Type: SSH (port 22), source: select “Your IP address”

b) Scroll down and click create security group

V — Webserver Security Group (Port: 80, 443 with source: ALB security group, port 22 with source: ssh security group)

Here we open port 80 (HTTP) and port 443 (HTTPS) with source coming from ALB Security group, and port 22 (SSH) with source coming from SSH Security Group

a) Follow the steps in (I) above with following details, name: Webserver Security Group, Type: HTTP (port 80), source: select “ALB Security Group”

b) Repeat (a) and add another rule with Type: HTTPS (port 80), source: select “ALB Security Group”

c) Add another rule with Type: SSH (port 22), source “EC2-Instance Security Group”

d) Scroll down and click create security group

VI — Database Security Group (Port: 3306 with source: Webserver security group)

Here we open port 3306 (HTTP) with source coming from Webserver Security group

a) Follow the steps in (I) above with following details, name: Database Security Group, Type: MySQL/Aurora (port 3306), source: select “Webserver Security Group”

b) Scroll down and click create security group

VII — EFS Security Group (Port: 2049 with source: Webserver security group, port 22 with source: ssh security group)

Here we open port 2049 (HTTP) with source coming from Webserver Security group, port 22 (SSH) with source: SSH Security Group

a) Follow the steps in (I) above with following details, name: EFS Security Group, Type: NFS (port 2049), source: select “Webserver Security Group”

b) Repeat (a) and add another rule with Type: SSH (port 22), source: select “EC2-Instance Security Group”

c) Scroll down and click create security group

d) While the EFS security group page, click on Edith inbound rule -> Add another rule with Type: NFS (port 2049), source EFS Security Group”

e) Scroll down and click safe rule


Step 4: Create RDS Instance (database)

We will create the rds instance in the private data subnet.

I — Create the subnet group.

This allows us to specify which subnet(s) we want to create our database in

a) From management console, type rds in the search box and select on RDS under services.

b) On the left side, select Subnet group -> Create DB Subnet group

c) Give it a name (we will call it “Database subnets”. Use same name for the description

d) Under VPC, select our VPC (“My VPC”) from the dropdown

e) Under Availability Zones, select the 2 availability zones (it will be us-east-1a, us-east-1b)

f) Under subnets, select the IP CIDR block that corresponds to the 2 subnets (20.1.4.0/24 for Private data Subnet AZ1, 20.1.5.0/24 for Private data subnet AZ2)

g) Scroll down and click create


II — Create database.

a) On the left side, select database -> Create database

b) For database creation method, we will select “Standard create”

c) Under engine type, select “MySQL”

d) Under version, select the latest version from dropdown list

e) Under template, select “Dev/Test”

f) For Availability and durability, select “Single DB instance” (according to our reference architecture, we can select “Multi-AZ DB instance” to create a standby database, but we select the single so we are not billed)

g) For DB instance identifier, enter “test-rds-db”

h) Master username, enter a username (we will use “test24”)

i) Master password, enter a password (we will use “Test1234”)

j) DB instance class, select “Burstable classes (includes t classes)”, toggle “include previous generation classes” then select db.t2.micro (this is the free one from the dropdown

k) For storage, enter “20”

l) Under VPC, select our vpc name (“My VPC”)

m) Under subnet group, select the subnet we created in step 4(I) above

n) Under public access, select “No”

o) Under VPC security group, select “Choose existing”, then remove the default security group, select “Database Security group” from the dropdown

p) Under Availability Zone, select “us-east-1b”

q) Under database authentication, check “Password authentication”

r) Expand “Additional configuration”, give a name under “initial database name” (we will use “applicationdb”)

s) Leave rest as default and scroll down, then click create database

t) It will take some time to create the rds instance. Wait for the status to change to available


Step 5: Create Elastic File System (EFS)

We will create an EFS with mount Target in the private data subnets in each availability zones. They will allow the webservers in the private app subnets to pull application code and configuration files from the same location. The webservers will use the EFS Mount Targets to connect to the EFS

a) From management console, type EFS in the search box and select EFS under services.

b) Click on Create File system, then click Customize

c) Give it a name (we will call it “test-EFS”)

d) Scroll all the way to Encryption. Check the box if you want to encrypt the file system data. We will uncheck it so we don’t get charged.

e) Under tags, give a tag key and tag value, then click Next

f) Under VPC, select your VPC (for this project, it is “My VPC”)

g) Under Mount targets, select the subnet for each availability zone. For us-east-1a, select Private Data Subnet AZ1, and Private Data Subnet AZ2 for us-east-1b. For each subnet, under security groups, remove the default security group and select “EFS Security Group”

h) We won’t add any file system policy, leave as default.

i) Review, scroll down and click create.


Step 6: Launch the Setup Server (EC2 Instance)

We will launch an ec2 instance that will be used to install WordPress.

a) From management console, type EC2 in the search box and select EC2 under services.

b) On the left side, select Instances -> Launch instances

c) Give it a name (it will be “setup server”)

d) Under Quick Start tab -> Amazon Linux, select “Amazon Linux 2023 AMI” (it’s free within the Free Tier period)

e) Under instance type, select “t2.micro” (which is also free)

f) Under key pair name, select “Proceed without a key pair” (since we’ll use eic endpoint)

g) Click “Edit” under network settings

h) Under VPC, select our VPC (“My VPC”), and select “Public Subnet AZ1” under subnet

i) Auto-assign public IP, select “Enabled”

j) Check “Select existing security group” and choose “EC2-Instance Security Group, ALB Security Group and Webserver Security Group”

k) Scroll to the right, review the settings and click “launch instance”

l) Wait for some time for the instance state = “running” and status check = “2/2 checks Passed”

Step 7: SSH into the EC2 instance (launched in step 6)

Here we will use EC2 instance connect endpoint (eice) to ssh into our instance.

For the step-by-step process to create an EICE and ssh to the EC2 instance, click here to follow the guide.

Step 8: Install and configure WordPress

I — After we have ssh into the setup server, we will run the following commands:

#1. create the html directory and mount the efs to it 
sudo su 
yum update -y 
mkdir -p /var/www/html
# Before running the next command, update the mount information of our EFS, from EFS page -> select efs ID -> 
# click attach, under “using the NFS client:”, copy the information in red and replace the text "fs-0b37356f0b8d8f269.efs.us-east-1.amazonaws.com"  in the command below
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0b37356f0b8d8f269.efs.us-east-1.amazonaws.com:/ /var/www/html

#2. install apache  
sudo yum install -y httpd httpd-tools mod_ssl 
sudo systemctl enable httpd
sudo systemctl start httpd

#3. install php 7.4 
sudo yum clean metadata 
sudo yum install php php-common php-pear -y 
sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y 
 
#4. install mysql5.7 
sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm 
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022 
sudo yum install mysql-community-server -y 
sudo systemctl enable mysqld 
sudo systemctl start mysqld 
 
#5. set permissions 
sudo usermod -a -G apache ec2-user 
sudo chown -R ec2-user:apache /var/www 
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \; 
sudo find /var/www -type f -exec sudo chmod 0664 {} \; 
chown apache:apache -R /var/www/html  
 
#6. download wordpress files 
wget https://wordpress.org/latest.tar.gz 
tar -xzf latest.tar.gz 
cp -r wordpress/* /var/www/html/ 
 
#7. create the wp-config.php file 
cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php 
 
#8. edit the wp-config.php file 
nano /var/www/html/wp-config.php
 # Once in the editor, we will update the following information
 # DB_NAME (insert “applicationdb”), DB_USER (insert “test24”), DB_PASSWORD (insert “Test1234”)
 # DB_HOST (insert the RDS endpoint, from the RDS console -> Connectivity & security tab -> copy the endpoint)
 # to exit the editor, on your keyboard, press “CTRL+X”, then “Y”, then “Enter”
 
#9. restart the webserver 
service httpd restart
II — Configure WordPress

a) Once we have restarted the Apache Web server, from the EC2 (setup server) management console, copy the public IPv4 address and paste in a new browser page (this will enable us finish WordPress installation)

b) Enter a Site title (we will use “test”)

c) Enter a username (we will use “test24”) and password (we will use “Test1234”)

d) Enter an email (we will use “mpndevops84@gmail.com”) and click “Install WordPress”

e) On the next page, click login, enter your username and password created above and click login.


Step 9: Create Application Load Balancer (ALB)

We will next create an ALB that will route traffic to our webservers (ec2 instances) in the private subnet

I — Launch an EC2 instance in each of the Private App Subnet AZ1 and Private App Subnet AZ2

a) From management console, type EC2 in the search box and select EC2 under services.

b) On the left side, select Instances -> Launch instances

c) Give it a name (it will be “WebServer AZ1”)

d) Under Quick Start tab -> Amazon Linux, select “Amazon Linux 2023 AMI” (it’s free within the Free Tier period)

e) Under instance type, select “t2.micro” (which is also free)

f) Under key pair name, select “Proceed without a key pair” (since we’ll use eic endpoint)

g) Click “Edit” under network settings

h) Under VPC, select our VPC (“My VPC”), and select “Private App Subnet AZ1” under subnet

i) Auto-assign public IP, select “Disabled”

j) Check “Select existing security group” and choose “EC2-Instance Security Group, and Webserver Security Group”

k) Scroll down to “Advanced details” and expand,

l) Scroll down to “User data” and enter the bash script below:

#!/bin/bash
yum update -y
sudo yum install -y httpd httpd-tools mod_ssl
sudo systemctl enable httpd 
sudo systemctl start httpd
sudo yum clean metadata
sudo yum install php php-common php-pear -y
sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y
sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum install mysql-community-server -y
sudo systemctl enable mysqld
sudo systemctl start mysqld
echo "fs-0b37356f0b8d8f269.efs.us-east-1.amazonaws.com:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a
chown apache:apache -R /var/www/html
sudo service httpd restart
a) Replace the text “fs-0b37356f0b8d8f269.efs.us-east-1.amazonaws.com” (in the echo command) with what we copied in step 8 (I)

b) Scroll to the right, review the settings and click “launch instance”

c) Wait for some time for the instance state = “running” and status check = “2/2 checks Passed”

d) After launching the first instance, repeat the steps (a) to (o) to launch the second instance with name “WebServer AZ2” in Private App Subnet AZ2


II — Create a Target Group

We will create a target group and put these instances in the target group to allow the ALB route traffic to them

a) In the ec2 management console, on the left side, under load balancing, select Target groups -> Create target group

b) Under target type, select “instances”

c) Under target group name, give a name (it will be “Test-TG”)

d) Protocol is “HTTP”

e) Under VPC, select your vpc (for our project, it is “My VPC”)

f) Protocol version is “HTTP1”

g) Under Health checks, select advanced health checks settings

h) Under success codes, enter “200,301,302”

i) Click Next

j) On the next page, select the instances “WebServer AZ1” and “WebServer AZ2” and click on “include as pending below”

k) Click create target group


II — Create an Application Load Balancer

a) In the ec2 management console, on the left side, under load balancing, select Load Balancers -> Create Load Balancer

b) Click create under the Application Load Balancer

c) Give it a name (for our project, we will give “Text-ALB”)

d) It will be “Internet-facing”

e) IP address type is “IPv4”

f) Select our VPC (for our project, it will be “My VPC”)

g) Under mappings, we will select our 2 public subnets, so select us-east-1a and check “Public Subnet AZ1”, similarly on us-east-1b, check “Public Subnet AZ2”

h) Under Security groups, remove the default and select “ALB Security Group” from the dropdown

i) Setup HTTP listener on port 80, under Default action, select your target group (“Test-TG”) from the dropdown list

j) Scroll down and click Create load balancer

k) Wait for the state to change to active.

l) Once state is Active, copy the DNS name from the Description tab and paste in a new browser page to see our Website displayed.


III — Change the domain settings in the WordPress configuration

a) While still on the website page, at the end of the url link, add “/wp-admin” and press enter

b) Login to your wordpress website with username and password created in step 8(II)

c) On the left side, select settings -> General

d) Under general settings, update both the WordPress Address (URL) and Site address (URL) and replace it with the DNS url copied in step II(l) above and remove the forward slash at the end

e) Scroll down and save changes.

f) It will log you out, just login again and check the website url should have changed.


IV — Terminate the Setup Server

Now we have 2 webservers up and running and can access them using the DNS url of the ALB, so we don’t need the setup server anymore, hence we will terminate it

a) From the EC2 management console, click on instance (running)

b) Select the setup server, goto to instance state -> Terminate instance

c) Then click terminate.

Step 10: Register a Domain name in Route 53

We will register a domain name to allow users to access our website instead of using the ALB DNS url.

Click here to follow the step-by-step guide to register a domain in Route 53


Step 11: Create a Record Set in Route 53 and update WordPress admin portal

A record set in route 53 will enable us access our website using our domain name registered in step 10 above.

a) From AWS management console, search Route 53 in the search box and select Route 53 under services

b) On Route 53 Dashboard, click on Hosted zone

c) Click your domain name registered in step 10 (In our case, it will “mpndevops.com”)

d) Under Records, click Create record

e) Record name will be “www” (for the subdomain)

f) Record type will be “A-Routes traffic to an IPv4 address and some AWS resources”

g) Toggle Alias button “ON”

h) Under Route traffic to, select “Alias to Application and Classic Load Balancer” for endpoint

“US East (N. Virginia) [us-east-1]” for the region

For Load balancer, select the ALB we created from the dropdown

i) Click Create records

j) Click view status

k) Wait for the status to change from Pending to INSYNC

l) Select the record just created, copy the record name under record details, paste it in a browser page and press Enter

m) Copy the Url from the browser (in our case, it will http://www.mtndevops.com)

n) At the end of the url, add “wp-admin”, then login with your username and password

o) On the left side, select settings -> General

p) Under general settings, update both the WordPress Address (URL) and Site address (URL) and replace it with the url copied in step (m) above and remove the forward slash at the end

q) Scroll down and save changes.

r) It will log you out, just login again and check the website url should have changed.


Step 12: Register for an SSL Certificate in AWS Certificate Manager (ACM)

We will register a free SSL certificate in ACM. If you already have a domain name with an SSL certificate, you can skip this step. This enables us to encrypt all communications between the browser and our webservers. This is referred to as Encryption in Transit

a) From AWS management console, search for Certificate manager in the search box and select Certificate manager under services

b) In the Certificate manager console, click Request certificate

c) Select Request a Public certificate, and click Next

d) Under domain name, we will enter our domain we registered in step 10 above (we will use mpndevops.com)

e) Click Add another name to this certificate and enter “*.<your domain name>” (in our case, it will be “*.mpndevops.com”)

f) Under validation method, select DNS validation

g) Scroll down and click Request

h) Under view certificate, the status will be “Pending validation”

i) To validate, we need to create a record set in Route 53 to show this domain name belongs to us

j) Click Create record in Route 53

k) Then select both domain name and the wild card (“mpndevops.com”, “*.mpndevops.com”)

l) Click Create records

m) Click refresh and status will be “issued”



Step 13: Create an HTTPS Listener for the Application Load Balancer (ALB)

We will create an HTTPS listener for our ALB to secure our website

I — Create an HTTPS listener

a) From AWS management console, search for EC2 in the search box and select EC2 under services

b) On the left, under Load balancing, select Load Balancers, goto the Listeners tab and click Add Listener

c) Under Protocol, select HTTPS (443), then default actions will be Forward to

d) Under target group, select your target group from the dropdown (in our case will “Test-TG”)

e) Under default certificate from ACM, select your certificate from the dropdown

f) Click Add

II — Edit HTTP listener to redirect traffic to HTTPS

a) Select the HTTP listener and click on Edit

b) Under default actions, select redirect

c) We will redirect it to HTTPS, type 443

d) Scroll down and save changes

III — Edit the wp-config file

To make sure our server is secure, we will edit the wp-config file in our webserver

a) Using EICE, ssh into one of the webservers in the private subnet (Webserver AZ1 or Webserver AZ2)

b) Change to root with the command “sudo su”

c) Type the command “nano /var/www/html/wp-config.php”

d) Move down to the free space and paste the following code below

/* SSL Settings */
define('FORCE_SSL_ADMIN', true);

// Get true SSL status from AWS load balancer
if(isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';
}
a) Press “CTRL+X” then “Y”, then “Enter” to save and exit the editor

b) Go ahead and test your website with https to confirm it’s secured, also test the website with http to confirm it redirects to https


Step 14: Create an Auto Scaling Group (ASG)

We are going to create an ASG. This will dynamically scale our webservers in the private subnets.

I — Terminate the 2 Webservers

First, we will terminate the 2 ec2 instances (“Webserver AZ1” and “Webserver AZ2”)

a) From the EC2 dashboard, select instance (running)

b) Then select Webserver AZ1 and Webserver AZ2, goto instance state -> Terminate instance

c) Click Terminate

II — Create a launch Template

A launch template has all the configurations on the ec2 that our ASG will use to create instances

a) From the EC2 Management console, on the left, under instances select Launch Templates -> Create launch template

b) Under Launch template name, give it a name (it will be “Test-launch-Template”), enter same name for description

c) Check the box under Auto Scaling Guidance

d) Under Quick Start tab -> Amazon Linux, select “Amazon Linux 2023 AMI” (it’s free within the Free Tier period)

e) Under instance type, select “t2.micro” (which is also free)

f) Under key pair name, select “Proceed without a key pair” (since we’ll use eic endpoint)

g) Under security groups, check “Select existing security group” and choose “Webserver Security Group”

h) Scroll down and click Advance

i) Click on Advance and scroll down to the User Data section

j) In this section, paste the following code

#!/bin/bash
yum update -y
sudo yum install -y httpd httpd-tools mod_ssl
sudo systemctl enable httpd 
sudo systemctl start httpd
sudo yum clean metadata
sudo yum install php php-common php-pear -y
sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y
sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
sudo yum install mysql-community-server -y
sudo systemctl enable mysqld
sudo systemctl start mysqld
echo "fs-0b37356f0b8d8f269.efs.us-east-1.amazonaws.com:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a
chown apache:apache -R /var/www/html
sudo service httpd restart
a) After pasting the commands, click Create launch template


III — Create the Auto Scaling Group (ASG)

a) From the EC2 Management console, on the left, under Auto scaling, select Auto Scaling Groups -> Create Auto scaling group

b) Give it a name (for our project, it will “Test-ASG”)

c) Under launch template, select your launch template from the dropdown (“Test-launch-Template”), and click on Next

d) Under VPC, select your VPC from the dropdown (for our project “My VPC”)

e) Under availability zones and subnets, select “Private App AZ1” and “Private App AZ2”, and click Next

f) Select Attach to an existing Load balancer

g) Next select Choose from your load balancer target group

h) Select your target group from dropdown list (“Test-TG”)

i) Select “ELB” under health checks

j) We can enable “Enable group metrics collection within CloudWatch” and click Next

k) Under group size, let’s specify our Desired capacity = 2, Minimum capacity = 1, Maximum capacity = 4

l) Scroll down and click Next

m) Click Add notification

n) Click Create a topic, once completed, click Next

o) Under Add tags, click Add tags

p) Give it a name and value (name = “Name”, Value = “Test-ASG”), click Nex, review settings and click Create auto scaling group

q) At this moment, our ASG will be launching ec2 instance with our desired capacity (2)

r) We can check the instances in EC2, wait for them to pass the “Status checks”, Next check if they are healthly in target group.

s) Once this is all ok, we can check our website “www.devops.com”




Step 15: Install WordPress Theme and Template

In this last step, we will give our WordPress website a theme
