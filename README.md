# DigitalBoost

> ## Project Scenario

A small to medium-sized digital marketing agency, "DigitalBoost", wants to enhance its online presence by creating a high-performance WordPress-based website for their clients. The agency needs a scalable, secure, and cost-effective solution that can handle increasing traffic and seamlessly integrate with their existing infrastructure. 

My task as an AWS Solutions Architect is to design and implement a WordPress solution using various AWS services, such as Networking, Compute, Object Storage, and Databases.

># Step 1: Create a VPC (Virtual Private Cloud), Subnets, Route table, Nat gateway

## VPC configuration

![vpc](./img/1.%20vpc.png)

- The VPC is named as **"wordpressVPC-vpc"**
- The standard CIDR Block of `10.0.0.0/16` is used to allow a large range of IP addresses for flexibility.
- DNS Settings: Enable DNS hostnames and DNS resolution for easier communication between resources.

## Subnet Configuration 

- Four Subnets are created.
- Public Subnets: `wordpressVPC-subnet-public1-eu-west-2a` and `wordpressVPC-subnet-public2-eu-west-2b`.
- Private Subnets: `wordpressVPC-subnet-private1-eu-west-2a` and `wordpressVPC-subnet-private2-eu-west-2b`.
- Availability zone: The subnets are distributed across two Availability Zones `(AZs: eu-west-2a and eu-west-2b)`. This ensures high availability and fault tolerance, which is perfect for this project.
- Placement: Public subnets are placed for resources like NAT Gateway and Application Load Balancer.
- Private subnets are placed for sensitive resources like EC2 instances and RDS.

The private subnet does not have direct internet access in this configuration. Instead, it uses a NAT Gateway in the public subnet to access the internet. This setup ensures that resources in the private subnet, such as our database or application servers, remain protected from direct inbound internet traffic while still being able to make outbound requests.

## Route Table configuration

- Public Route Table (`wordpressVPC-rtb-public`): is to the public subnets, it Contains a route to the internet gateway (`0.0.0.0/0`)` for outbound traffic.

- Private Route Tables (`wordpressVPC-rtb-private1-eu-west-2a` and `wordpressVPC-rtb-private2-eu-west-2b`): routes external traffic through the NAT Gateway to ensure resources in private subnets can reach the internet securely.

## Network connections

- Internet Gateway: Ensures public subnets have internet access – this is necessary for the ALB and NAT Gateway.

- NAT Gateway: Located in a public subnet, providing secure internet access for private resources like RDS and EC2 for updates or installations.

- VPC Endpoint (S3): This allows private subnet resources to securely access AWS S3 without going through the public internet. This is excellent for security.

> # Step 2: Set up an EC2 Instance.

The main purpose for setting up an EC2 instance is because this instance will host our WordPress application.

## EC2 configuration

- The EC2 instance for this project is named *Digital-Boost*.
- Amazon Linux 2 AMI is used to launch this instance because it is lightweight and compatible with WordPress.

![EC2](./img/2.%20ec2a.png)

- I will be using my keypair named *VPC-keypair* for SSH.

- For network configuration, the instance will be assigned to the public subnet of our just created VPC.

- Auto-assign public IP is enabled.

![EC2](./img/3.%20ec2b.png)

- Security groups that allow SSH (port 22) only from IP, HTTP (Port 80) and HTTPS (Port 443) for public access will be assigned to our inbound security group rule.

![EC2](./img/4.%20Inbound%20rule.png)

![EC2](./img/5.%20launch%20instance.png)

After launching the EC2 instance, I will now create and configure RDS for the database.

># Step 3: Configure RDS for Database.

The purpose of creating RDS is to store WordPress data like posts, settings, and user information.

## RDS configuration for Database.

- First, I will be creating a new security group for the RDS. This security group allows traffic from EC2 to RDS on port 3306.

- Under the inbound rule section, MySQL/Aurora with default port 3306 will be added.

- The security group ID for the EC2 instance we just created will be selected as the source, and the same VPC will also be attached.

![RDS SG](./img/rds%20security%20group.png)

- For the RDS configuration, we will be using the standard database creation method with a *MySQL* engine option.

![RDS config](./img/6.%20rds%20config1.png)

- The instance configuration will be using `db.t3.micro`, which is free tier-eligible and cost-effective for development or testing environments.

- Storage type will be kept as *General Purpose SSD(GP2) and storage allocated will be kept at 20 GiB.

![RDS config](./img/7.%20Rds%20config%202.png)

- We will be connecting our EC2 instance to the RDS. This is to ensure the two components work together seamlessly for the project. It reduces debugging time and gives it a more streamlined setup.

![RDS config](./img/8.%20RDS%20CONFIG%203.png)

- For the DB subnet group, we are going to attach a private subnet group spanning multiple availability zones for redundancy.

![RDS config](./img/7a.%20rds%20config.png)

- We will also be attaching the *RDS-DB-SG* security group to the configuration.

![RDS config](./img/9.%20Rds%20config%204.png)

> ## Step 4: Connect WordPress.

Connecting WordPress to your RDS database is an important step to ensure WordPress can save and retrieve data from your database.

- We will first use our terminal to connect to the instance using ssh client.

![EC2 CONNECT](./img/10.%20connect%20to%20EC2.png)

- We will now create the HTML Directory and mount EFS. To do this, we will be switching to the root user using command `sudo su`, then we will update the system and create the directory using command `yum update -y`
`mkdir -p /var/www/html`.

![EC2 CONNECT](./img/11.%20html.png)

## Create and configure an EFS File System.

EFS stands for Elastic File System, it is a serverless, fully elastic file storage system. 

Creating an Elastic File System (EFS) for your WordPress project is essential when scalability and shared storage are required.

- To create EFS, we will navigate to the EFS section of our AWS console.

- We will set up our file system by naming it *WordpressEFS*, enable encryption for security and attach the same VPC as our EC2 instance.

- We will also ensure that there are mount targets created for each availability zone where EC2 instance resides.

- And then create.

![EFS](./img/12.%20efs.png)

- Next, we will update the security group attached to the EFS. EFS must allow NFS traffic from the EC2 instance.

![EFS](./img/13.%20EFS%20Security%20group.png)

## Mounting EFS on the EC2 Instance

- Next, we will now mount our EFS onto the HTML directory created by using the command `sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-005532cb83eeff189.efs.eu-north-1.amazonaws.com:/ /var/www/html`. This command contains the DNS name for our EFS.

![EFS](./img/14.%20mount.png)

![EFS](./img/14.%20MOUNT%202.png)

- We will be installing *Apache* because it acts as the web server that will host the WordPress site. To install Apache, we will use command `sudo yum install -y httpd httpd-tools mod_ssl`.

![Apache](./img/15.%20aPACHE.png)

- To enable and start Apache service, we will be using command `sudo systemctl enable httpd`
`sudo systemctl start httpd`.

![Apache](./img/16.%20aPACHE%202.png)

- Next, we will be installing PHP and necessary extensions. PHP works with Apache to process WordPress scripts.

![PHP](./img/17.%20PHP.png)

- We will now add MySQL 5.7 locally to our EC2 instance using `sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm`
`sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022`

![MySQL](./img/18.%20mYSQL.png)

- Install and start MySQL service using command `sudo yum install mysql-community-server -y`
`sudo systemctl enable mysqld`
`sudo systemctl start mysqld`.

![MySQL](./img/19.%20mysql%202.png)

![MySQL](./img/20.%20mysql%203.png)

## Setting Permissions for WordPress

- We will be adding the EC2 user to Apache group using command `sudo usermod -a -G apache ec2-user`. This is so we are able to manage file permissions and ensure smooth operation of the WordPress application.

![WP Permission](./img/21.%20USER.png)

- To set directory and file permissions, we will be using command `sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
sudo chown apache:apache -R /var/www/html`

![WP Permission](./img/22.%20permission.png)

## Download and configure WordPress Files.

- We will now be downloading and extracting WordPress file using command `wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
cp -r wordpress/* /var/www/html/`.

![WP Permission](./img/23.%20Download.png)

- Configure worldpress using command `cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php`

- Locate wp-config-sample.php in the WordPress directory, because this is just a template, we will use command `cp wp-config-sample.php wp-config.php` to create an actual wp-config.php file.

- We will now use Nano text editor to edit the wp-config.php file and update the database settings.

![WP Permission](./img/24.%20nano.png)

## Testing the connection

- To test the connection, we will copy our EC2 instance's public IP and run on a browser. The result below shows that the connection is successful.

![WP Permission](./img/25.%20result.png)

># Step 5: Enable SSL for WordPress.

To enable HTTPS, we need an SSL certificate. For simplicity, we can proceed using the public IP address with a self-signed SSL certificate.

- First, we will be creating a required directory structure for our private key using command `sudo mkdir -p /etc/ssl/private`

- Once the directory is created, run the OpenSSL command `sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt`

- After generating the SSL certificate successfully, the next step is to configure the Apache web server to use it and enable HTTPS for your site.

- With command `sudo nano /etc/httpd/conf.d/ssl.conf` we will be opening the Apache's SSL configuration file, locating the default SSL certificate file path and the default SSL Certificate key file path and replacing it with our own path. 

![Apache Permission](./img/27.%20CONFIG%20APACHE.png)

- Then we will restart the Apache to apply changes.

## Testing the connection.

After restarting the Apache, we will now test by visiting the website putting our public IP in a browser.

![Apache result](./img/28.%20it%20works.png)

From the above image, we can see the default Apache welcome page. This shows that our web server is running properly, but it's not showing the WordPress site.

To fix this issue, we will first locate our downloaded WordPress using command `find / -name 'index.php' | grep wordpress`

![Apache result](./img/29.%20move.png)

From the image above, we can see the WordPress files are located in `/home/ec2-user/wordpress`. To make the WordPress site accessible, I will be moving the files from `/home/ec2-user/wordpress` to `/var/www/html` using command `sudo mv /home/ec2-user/wordpress/* /var/www/html/`

Now I'll be granting Apache with the necessary permissions to access the moved files. After that, I'll restart the Apache service.

![Apache result](./img/30.%20result.png)

Selecting my preferred language to English (United States). Next, I'll be setting up the WordPress site and filling in the image details below.

![WordPress](./img/31.%20WP%20homepage.png)

Once all the fields are filled in, I will Install WordPress by clicking the install button at the bottom of the page. 

![WordPress](./img/32.%20success.png)

After installation, we can see the success message above, then I'll login.

![WordPress](./img/33.%20login.png)

After putting in my login info, we can see the WordPress Dashboard.

![WordPress](./img/34.%20Homepage.png)

> # Step 6: Application Load Balancer (ALB)

The Application Load Balancer manages and distributes incoming traffic to the WordPress EC2 Instance. 
In this project, the ALB serves as the front-facing component of our WordPress infrastructure, balancing traffic efficiently and ensuring your site is resilient, scalable, and secure.

## Setting up Application Load Balancer (ALB).

- Navigate to the EC2 page, and click on Load balancer, then create Load balancer.

![ALB](./img/35.%20ALB.png)

- Select Application Load Balancer.

![ALB](./img/36.%20ALB%201.png)

- I am going to create a new security group for my ALB that allows HTTP (Port 80), HTTPS (port 443) and SSH (port 22).

![ALB](./img/37.%20Security%20group%20wp.png)

- I am also going to create a target group for the Listeners tab. 

![ALB](./img/38.%20Target%20group.png)

- The target group will be Instance since we are targeting our WordPress EC2 instance.

![ALB](./img/39.%20Target%20group%202.png)

- For the ALB, I named it *wordpress-alb*, chose internet-facing and IPv4 as IP address type.

- Same VPC as my instance is attached, I used a subnet that covers multiple availability zones. My new security group and target groups are attached.

![ALB](./img/40.%20ALB.png)

- After Creating the ALB, I will test the ALB's HTTP connection by visiting the DNS name in my browsers.

![ALB](./img/41.%20result.png)

It works, from this result we can see the WordPress site is live and accessible through DNS.

> # Step 7: Auto-scaling Group

The ASG gives our WordPress site the resilience and flexibility it needs to handle real-world scenarios while keeping costs manageable.

## Setting up Auto-scaling Group

- Navigate to Auto Scaling on the EC2 Dashboard in AWS.

- On the left-hand menu, click Auto Scaling Groups under the Auto Scaling section.

![ALB](./img/42.%20auto%20scaling.png)

- We are going to attach our VPC and multiple availability zones.

![ALB](./img/43.%20ASG.png)

## Load Balancer Integration

- We will be attaching an existing load balancer from the list, this will be the ALB we previously created.

- Under the target group, we will attach our WordPress target group we created previously.

- Desired capacity will be 1, minimum capacity will be 1 and maximum capacity will be 3.

![ALB](./img/44.%20ASG.png)

> # Step 8: Simulating increased traffic to showcase Auto-scaling.

In this test, we will be using Apache's built-in tool Apache Benchmark (ab).

We will be simulating increased traffic using command `ab -n 1000 -c 50 http://wordpress-alb-1926726016.eu-north-1.elb.amazonaws.com/`. 

-n 1000: Sends 1000 total requests to the URL.

-c 50: Sends 50 requests concurrently, simulating multiple users.

This will generate load on our ALB and test how the Auto Scaling Group handles increased traffic.

![ALB](./img/44a.%20check%201.png)

## Results

From our EC2, we can see additional instances were created to support the high traffic flow. We have 3 instances running because we set the maximum number of instances to be 3.

![ALB](./img/46.%20CHECK%20INSTANCE.png)

From our Auto-scaling group, we can see unhealthy instances are automatically replaced.

![ALB](./img/45.%20CHECK.png)

From the Load balancer, we can see the spike as a result of the 1000 requests sent to the URL at that particular time.

![ALB](./img/48.%20CHECK.png)
![ALB](./img/47.%20CHECK.png)

> # Conclusion 

In this project, we successfully built and deployed a high-performing WordPress site on AWS, implementing critical security measures to ensure a robust and secure infrastructure. Below are the key security measures integrated into the architecture:

> VPC Isolation:

Created a Virtual Private Cloud (VPC) to isolate resources and secure the WordPress infrastructure.

Segmented the network into public and private subnets, ensuring that sensitive resources remain protected from direct internet exposure.

> NAT Gateway for Private Subnets:

Configured a NAT Gateway to provide internet access to resources in private subnets while blocking inbound internet traffic.

> Security Group Rules:

Defined security groups to restrict traffic, allowing only specific ports for services (e.g., HTTP/HTTPS on 80/443 and SSH on 22 for restricted IPs).

Created separate security groups for the