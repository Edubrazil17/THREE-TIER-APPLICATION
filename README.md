Deploying Three Tier Web Application On AWS
In this project, a public-facing Application Load Balancer forwards client traffic to web tier EC2 instances. The web tier is running Nginx webservers that are configured to serve a React.js website and redirects API calls to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ database and returns it to web tier. Load balancing, health checks and autoscaling groups are created at each layer to maintain the availability of this architecture.



Part 1: Setup
For this project, I will be downloading my code from Github and upload it to S3 so my instances can access it. I will also create an AWS Identity and Access Management EC2 role so that i can use AWS Systems Manager Session Manager to connect to the instances securely and without needing to create SSH key pairs.


https://github.com/Edubrazil17/THREE-TIER-APPLICATION.git


Next Step: Creating S3 Bucket 



I navigate to the S3 service in the AWS console and create a new S3 bucket.


 

Give it a unique name, and select the region that i intend to run this project in. This bucket house the file for this project 




Next Step: Creating Iam role and neccessary permission for the EC2 instance


 



Select EC2 as the trusted entity.




When adding permissions, i include the following AWS managed policies.  

AmazonSSMManagedInstanceCore

AmazonS3ReadOnlyAccess
These policies will allow the instances to download the code from S3 and use Systems Manager Session Manager to securely connect to the instances without SSH keys through the AWS console.





Part 2: Networking and Security
In this part i build out the VPC networking components

VPC

Subnets

Route Tables

Internet Gateway

NAT gateway

Security Group

Nacl

Security groups and Nacl add layer of protection around the EC2 instances, Aurora databases and Elastic Load Balancers.


VPC Creation






Subnet Creation

For this project i create six subnets across two availability zones. That means that three subnets will be in one availability zone, and three subnets will be in another zone. Each subnet in one availability zone will correspond to one layer of the three tier architecture. 


I Create each of the 6 subnets by specifying the VPC then choose a name, availability zone and appropriate CIDR range for each of the subnets.


NOTE:

The CIDR range for the subnets will be subsets of the VPC CIDR range.





Next Step: Internet Gateway

In order to give the public subnets in the VPC internet access i create and attach an Internet Gateway. 



I Create the internet gateway by simply giving it a name and clicking Create internet gateway.



After creating the internet gateway, i attach it to the VPC






Next Step: NAT Gateway

In order for the instances in the app layer private subnet to be able to access the internet they will need to go through a NAT Gateway. For high availability, l deploy one NAT gateway in each of the public subnets. 






Next Step: Routing Configuration

In this step, I set up a route table for each layer of the architecture specifing how reasources in the vpc communicate with each other and also with reasources outside the vpc.

For web layer public subnets, I Navigate to Route Tables on the left side of the VPC dashboard and click Create route table 





After creating the route table, and automatically taken to the details page. I click on the Routes tab and Edit routes.



I Add a route that directs traffic from the VPC to the internet gateway. In other words, for all traffic destined for IPs outside the VPC CDIR range, I add an entry that directs it to the internet gateway as a target.



I Edit the Explicit Subnet Associations of the route table by navigating to the route table details again. Select Subnet Associations and click Edit subnet associations.



I identify and associate the two web layer public subnet to the public route table



Now i create 2 more route tables, one for each app layer private subnet in each availability zone. These route tables route app layer traffic destined for outside the VPC to the NAT gateway in the respective availability zone.






Once the route tables are created and routes added, I add the appropriate subnet associations for each of the app layer private subnets.



Next Step: Security Groups

Security groups will tighten the rules around which traffic will be allowed to the Elastic Load Balancers and EC2 instances.



The first security group l create is for the public, internet facing load balancer and i add an inbound rule to allow HTTP and HTTPS type traffic.



The second security group l create is for the public instances in the web tier. After typing a name and description, I add an inbound rule that allows HTTP  and HTTPS type traffic from the internet facing load balancer security group. This will allow traffic from public facing load balancer to hit the instances. Then, i add an additional rule that will allow HTTP type traffic for my IP. This will allow me to access the instance to test.



The third security group will be for internal load balancer.  I Create this security group and add an inbound rule that allows HTTP type traffic from public instance security group. This will allow traffic from the web tier instances to hit the internal load balancer.



The fourth security group l configure is for the private instances. After typing a name and description,  I add an inbound rule that will allow TCP type traffic on port 4000 from the internal load balancer security group. This is the port the app tier application is running on and allows the internal load balancer to forward traffic on this port to the private instances. and also add another route for port 4000 that allows my IP for testing.



The fifth security group protects the private database instances. For this security group, I add an inbound rule that will allow traffic from the private instance security group to the MYSQL/Aurora port (3306).



Part 3: Database Deployment
This section of the project focus on deploying the database layer of the three tier architecture.

First Step: Subnet Groups


I Navigate to the RDS dashboard in the AWS console and click on Subnet groups on the left hand side. Click Create DB subnet group.




I give the subnet group a name, description, and choose the appropriate VPC.




Next I add the subnets in each availability zone specificaly for the database layer by selecting the correct db subnet IDs.



Database Deployment
Navigate to Databases on the left hand side of the RDS dashboard and click Create database.




Next Step i go through several configuration steps.  I start with a Standard create and for engine type, I select Amazon Aurora database




Under the Templates section I choose Dev/Test since this isn't being used for production at the moment. Under Settings  I set a username and password. I will use this password for authentication to access the database.




Next, under Availability and durability I change the option to create an Aurora Replica or reader node in a different availability zone. Under Connectivity, I set the VPC, choose the subnet group I created for this database, and select no for public access. 



I Set the security group I created for the database layer and under Database authentication I select password as my authentication choice and create the database.




After the database is provisioned, A reader and writer instance is deploy to different subnets of each availability zone. 



Part 4: App Tier Instance Deployment

In this section of the project I create an EC2 instance for the app layer and make all necessary software configurations so that the app can run. The app layer consists of a Node.js application that will run on port 4000. I will also configure the database with some data and tables.


First Step I navigate to the EC2 service dashboard and click on Instances on the left hand side. Then, click Launch Instances.




Next Step I Select ubuntu as my AMI and T.2 micro as my instance type and when configuring the instance details,  I make sure to select the correct Network, subnet, and IAM role I created. Since this is the app layer, I use one of the private subnets I created for this layer.


Next Step: Connect to Instance

I connect to the instance through Session Manager by navigating to the list of running EC2 Instances and clicking on Instances on the left hand side of the EC2 dashboard. When the instance state is running






When I first connect to the instance I was logged in as ssm-user which is the default user. I  switch to ec2-user by executing the following command in the browser terminal


sudo -su ubuntu


At this moment to make sure that I can reach the internet via the NAT gateways. I ping the google DNS servers to test the network configuration by running the following command in the browser terminal 


ping 8.8.8.8


And i received a transmission packet back which means i can reach the internet through my nat gateway 


NEXT: Configure Database
I start by downloading the MySQL CLI by executing the following command 


sudo apt-get install mysql-server -y 


Next i connect to the database through the Aurora writer endpoint with the following command 


mysql -h "RDS-ENDPOINT -u USER-NAME -p


When prompt for password i input the password i created when provision the database to be able to authenticate into the database.


I Create a database called webappdb with the following command using the MySQL CLI:


CREATE DATABASE webappdb;   





I Create a data table by first navigating to the database i just created by the following command 


USE webappdb;


Then, create the following transactions table by executing this create table command


CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL

AUTO_INCREMENT, amount DECIMAL(10,2), description

VARCHAR(100), PRIMARY KEY(id));    




I Verify the table was created by 


SHOW TABLES; 



I Insert data into table for use/testing later by 


INSERT INTO transactions (amount,description) VALUES ('400','groceries');   


I Verify that the data was added by executing the following command


SELECT * FROM transactions;




Next Step: Configure App Instance


The first thing I do is update the database credentials for the app tier. To do this, I open the application-code/app-tier/DbConfig.js file from the github repo and update the empty strings for the hostname, user, password and database and fill this in with the credentials i configured for the database, the writer endpoint of the database as the hostname, and webappdb for the database. 


NOTE: This is NOT considered a best practice, and is done for the simplicity of the project. For best practice and more secure, configure the databses credentials to a suitable place like Secrets Manager 






Upload the app-tier folder to the S3 bucket created in part 1.

On the SSM session. I install all of the necessary components to run the backend application. By installing NVM (node version manager).


curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash

source ~/.bashrc


Next, I install a compatible version of Node.js and make sure it's being used


nvm install 16

nvm use 16


PM2 is a daemon process manager that keep node.js app running when I exit the instance or if it is rebooted.


npm install -g pm2   


Now I need to download the code from the s3 buckets onto the instance. In the command below, 


cd ~/

aws s3 cp s3://app-tier/ app-tier --recursive




Next I navigate to the app directory, install dependencies, and start the app with pm2.


cd ~/app-tier

npm install

pm2 start index.js






Right now, pm2 is just making sure the app stays running when i leave the SSM session. However, if the server is interrupted for some reason, I want the app to start and keep running. 


pm2 startup




The pm2 startup command output a  the following command that i copy and run on the terminal and save 




 





Part 5: Internal Load Balancing and Auto Scaling

In this section of the project i will create an Amazon Machine Image (AMI) of the app tier instance i just created, and use that to set up autoscaling with a load balancer in order to make this tier highly available.


I navigate to Instances on the left hand side of the EC2 dashboard. Select the app tier instance and under Actions select Image and templates then Create Image.





Next Step i create Target Group










Next i create Internal Load Balancer












Next I create Launch Template












Next i create Auto Scaling











Part 6: Web Tier Instance Deployment
In this section I will deploy an EC2 instance for the web tier and make all necessary software configurations for the NGINX web server and React.js website.


Before i create and configure the web instances, I open up the application-code/nginx.conf file from the repo. Scroll down to line 58 and replace [INTERNAL-LOADBALANCER-DNS] with the internal load balancer’s DNS entry. 




Then, upload this file and the application-code/web-tier folder to the s3 bucket created for this project.


Next Step: Web Instance Deployment


I follow the same instance creation instructions i used for the App Tier instance in Part 4 App Tier Instance Deployment, with the exception of the subnet. i will be provisioning this instance in one of the public subnets. I make sure to select the correct network components, security group, and IAM role. This time, auto-assign a public ip on the Configure Instance Details page. 


Next Step: I Connect to the Instance through the system manager and change the default user to ubuntu and test the network connectivity through ping.


sudo -su ubuntu 

ping 8.8.8.8






 Next i Configure Web Instance


I install all of the necessary components needed to run the front-end application. Again, start by installing NVM and node


curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash

source ~/.bashrc

nvm install 16

nvm use 16


Nowi need to download the web tier code from the s3 bucket


cd ~/

aws s3 cp s3://app-tier/web-tier/ web-tier  --recursive





I Navigate to the web-layer folder and create the build folder for the react app i can serve the code


cd ~/web-tier

sudo apt install npm 

npm run build


NGINX can be used for different use cases like load balancing, content caching etc, but I will be using it as a web server that i will configure to serve the application on port 80, as well as help direct API calls to the internal load balancer


sudo apt-get install nginx -y


I configure NGINX. Navigate to the Nginx configuration file with the following commands and list the files in the directory


cd /etc/nginx

ls


I delete this file and use the one I uploaded to s3. 


sudo rm nginx.conf

sudo aws s3 cp s3://app-tier/application-code/nginx.conf 


Then, restart Nginx with the following command


sudo service nginx restart


To make sure Nginx has permission to access our files execute this command


chmod -R 755 /home/ec2-user


And then to make sure the service starts on boot, run this command


sudo chkconfig nginx on


Now when I plug in the public IP of the web tier instance, the website is serve through the internet load balancer that distribute traffic to the instance, the database is also working and i'm able to add data. 







Part 7: External Load Balancer and Auto Scaling
In this section of the project i will create an Amazon Machine Image (AMI) of the web tier instance i created and use that to set up autoscaling with an external facing load balancer in order to make this tier highly available.


Next Step: Creating Web Tier AMI


I Navigate to Instances on the left hand side of the EC2 dashboard. Select the web tier instance I created and under Actions select Image and templates and click Create Image.






Next Step: Creating Target Group


While the AMI is being created, i go ahead and create target group to use with the load balancer. On the EC2 dashboard i navigate to Target Groups under Load Balancing on the left hand side and click on Create Target Group.




The purpose of forming this target group is to use with theload blancer so it may balance traffic across the public web tier instances. I Select Instances as the target type and give it a name.







Next Step: Creating Internet Facing Load Balancer









I select the security groupi created for this internal ALB. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to the target group that i created, so i select it from the dropdown, and create the load balancer.




Next Step: Creating Launch Template


Before i configure Auto Scaling, i need to create a Launch template with the AMI i created 

earlier. On the left side of the EC2 dashboard i navigate to Launch Template under Instances and click Create Launch Template.







Under Instance Type i select t2.micro. For Key pair and Network Settings i don't include it in the template. i don't need a key pair to access the instances and i will be setting the network information in the autoscaling group




I set the correct security group for the web tier, and then under Advanced details i use the same IAM instance profile i have been using for the EC2 instances.






Next Step: Creating Auto Scaling


I will now create the Auto Scaling Group for the web instances. On the left side of the EC2 dashboard i navigate to Auto Scaling Groups under Auto Scaling and click Create Auto Scaling group.












To test if the entire architecture is working, i navigate to the external facing loadbalancer, and plug in the DNS name into my browser.



