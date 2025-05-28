![image2f](https://github.com/user-attachments/assets/b9b3ba33-54e9-44f2-9cbb-c97680d20c5c)# AWS Three-Tier Architecture Configuration Guide

This document details the steps followed to set up a three-tier web architecture on Amazon Web Services (AWS).

## 1. VPC (Virtual Private Cloud) Configuration

The first step involved establishing the isolated network environment. A VPC named "project" was created in the `us-east-1` region.

![image](https://github.com/user-attachments/assets/41fc5131-b664-46fe-8128-a383b3d6a9f7)


To ensure high availability, this VPC spans two Availability Zones (AZs): `us-east-1a` (AZ1) and `us-east-1b` (AZ2). Within each of these zones, a subnet infrastructure was configured, comprising one public subnet and three private subnets. This segmentation allows for the separation of publicly accessible resources from those that must remain private.

![image](https://github.com/user-attachments/assets/2381d0c3-e0fd-4bb9-acb6-5c9e4aecd5a9)


To enable communication between the VPC and the internet, an Internet Gateway was created and attached to the "project" VPC.

![image2](https://github.com/user-attachments/assets/611ba407-3388-449d-bdf4-22229eaa40cd)


To allow instances located in the private subnets to access the internet for updates or external API calls without being directly exposed, two NAT (Network Address Translation) Gateways were set up, one in each Availability Zone.

![image3](https://github.com/user-attachments/assets/a1c64456-bc86-423f-86c5-16334beeb59f)


Network traffic management within the VPC was configured using distinct route tables.

![image4](https://github.com/user-attachments/assets/348cb45d-cd16-4f6b-af0f-29d3adf290f8)


A public route table was associated with the public subnets, directing traffic destined outside the VPC (0.0.0.0/0) to the Internet Gateway.

![image5](https://github.com/user-attachments/assets/6e22fb34-1534-435a-ba1b-110b2628d98f)

![image6](https://github.com/user-attachments/assets/fa4ff336-1a8e-4cfb-bda2-4a6757f8f6c8)

For the private subnets, two separate route tables were created: "Private route table AZ1" for the private subnets in `us-east-1a` and "Private route table AZ2" for those in `us-east-1b`. The AZ1 private route table routes outbound traffic to the NAT Gateway located in `us-east-1a`. The configuration of the AZ2 private route table is similar, directing traffic to its own NAT Gateway in `us-east-1b` and being associated with the corresponding private subnets.

![image7](https://github.com/user-attachments/assets/3a1b6f2b-0fb4-4791-88a8-3b139e77f6c2)

![image8](https://github.com/user-attachments/assets/1e599731-2831-43bf-81b8-13f9fe4e0622)


## 2. Security Group Setup

Following the network configuration, the next step was defining firewall rules using Security Groups. Several groups were created to finely control inbound and outbound traffic for each component of the architecture:


*   A security group for the bastion host, allowing SSH access from specific IP addresses for secure administration.
*   ![imageb](https://github.com/user-attachments/assets/2bccd1a4-b095-4ee0-b150-fd18c6ff0f2a)
*   A security group for the internet-facing load balancer, permitting HTTP and HTTPS traffic from the internet.
*   
*   A security group for the web servers, allowing traffic from the public load balancer and potentially the bastion host.
*   ![imagef](https://github.com/user-attachments/assets/808982dc-49a3-4631-a9e5-d3d5c610956b)
*   A security group for the internal load balancer, allowing traffic from the web servers.
*   ![image11](https://github.com/user-attachments/assets/557ba70d-b685-41be-8d89-506886e5f388)
*   A security group for the application servers, allowing traffic from the internal load balancer and potentially the bastion host.
*   ![image13](https://github.com/user-attachments/assets/15615100-4e3a-45d4-a891-2d66224cddd4)
*   A security group for the database (RDS), restricting access to the database port solely from the application servers and the bastion host.
*   ![image9](https://github.com/user-attachments/assets/e442da84-0c46-49f3-bf62-fa90392b3c04)


## 3. Database Deployment (RDS)

For the data persistence layer, a relational database instance was deployed using Amazon RDS (Relational Database Service). A MySQL instance of type `db.t3.micro` with 20 GB of storage was created and named `projectdb`. To ensure resilience, the primary instance was placed in Availability Zone AZ1 (`us-east-1a`), and a standby instance was configured in AZ2 (`us-east-1b`). RDS automatically handles replication and failover in case the primary instance fails.
![image15](https://github.com/user-attachments/assets/e10835b0-e744-497c-9bfe-6850db9f0401)
![image16](https://github.com/user-attachments/assets/8d817662-2d49-4b60-a389-813f795e69d9)
![image17](https://github.com/user-attachments/assets/fbe04010-bccc-40f1-84e1-49bccabe2c91)
![image18](https://github.com/user-attachments/assets/8b8dead0-0ce8-4afb-b924-eba08ce3eedb)

## 4. Instances Creation

1.  A Bastion Host EC2 instance to provide a secure access point to private instances.
   
An SSH key pair (`Host.pem`) was generated during the creation of the Bastion Host and reused for the other instances to simplify access management.

![image19](https://github.com/user-attachments/assets/a1cdc605-8656-4f65-a3b2-b8f8b331f783)

An Elastic IP address was associated with the Bastion Host to provide a static public IP address and facilitate SSH connections.

![image1c](https://github.com/user-attachments/assets/deeb5076-3610-4bc0-9f73-5889a8f50a90)


Connection to the private servers is made via the Bastion Host. After connecting to the Bastion, the private key (`Host.pem`) was copied onto it, subsequently allowing an SSH connection to the private server.


2.  A Web Server EC2 instance.
   
![image1b](https://github.com/user-attachments/assets/99bf7d35-78a4-409f-8cb3-f2391a22cef5)

4.  An Application Server EC2 instance.

![image1a](https://github.com/user-attachments/assets/e95527b7-0754-4949-8f27-628889a76a60)

## 5. Application Server Configuration

The first step is to connect to the App Server instance using the Bastion Host.

![image1d](https://github.com/user-attachments/assets/bbed4cf7-0ed8-48b7-81a1-c1acc7f92578)

On the application server, the necessary environment was installed. First, the MySQL client was installed to enable connection to the RDS database. The following commands were used (note: specific commands may vary slightly depending on the Linux AMI used; the example shows installation for an EL9-based distribution):

```bash
install mysql
sudo yum install https://dev.mysql.com/get/mysql80-community-release-el9-5.noarch.rpm
sudo yum install mysql-community-server
sudo systemctl enable --now mysqld 
```

The connection to the RDS database was established using the RDS instance endpoint:

```bash
mysql -h <Amazon RDS Endpoint> (e.g projectdb.cryby2rgpj6y.us-east-1.rds.amazonaws.com) -P 3306 -u <user> (e.g ProjectDB) -p
# Enter the password (e.g 'ProjectDB') when prompted
Note: It is possible to include the password in the previous command to connect without the prompt like this -p <password>, but it's not recommended because the password may be visible in the shell history
```

![image1e](https://github.com/user-attachments/assets/96371aa0-8a7b-4f41-ac0c-5a09c51a2bd0)
![image1f](https://github.com/user-attachments/assets/5570a892-2c4a-4cb4-ba14-78cb706dc211)
![image20](https://github.com/user-attachments/assets/5be5bb5f-dc86-421b-b494-7b63c6a88b6b)

Next, the Node.js environment was configured using NVM (Node Version Manager), and the PM2 process manager was installed to run the Node.js application in the background and ensure its availability:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
npm install -g pm2
```

The application source code was retrieved from a public GitHub repository:

```bash
sudo yum install git
git clone https://github.com/Adam-Zouari/DrOrlow.git app-tier
cd app-tier
rm -r public # Remove front-end files, not needed on the application server
cd server
npm install mysql express body-parser # Install back-end dependencies
pm2 start app.js # Start the application with PM2
pm2 save # Save PM2 configuration for automatic restarts
```

![image21](https://github.com/user-attachments/assets/d31539ce-c1c9-45eb-96c2-80668019f397)
![image22](https://github.com/user-attachments/assets/a8f2cfde-c1fc-452d-af4b-4bd48aaa8114)

Once the application server was configured and the application was functional, an Amazon Machine Image (AMI) was created from this instance. This AMI serves as a template for launching new, identical application server instances.

![image23](https://github.com/user-attachments/assets/df08c5aa-cea7-4d4f-9bc3-b068eeb663f4)
![image24](https://github.com/user-attachments/assets/7d431cd8-ed32-4141-a140-4a336a27bf0d)

To distribute incoming traffic to the application instances and improve availability, a Target Group named `AppTargetGroup` was created.

![image25](https://github.com/user-attachments/assets/88ea1735-f10e-438f-bf23-d4159f0fe03c)

Followed by an Internal Load Balancer. This load balancer is only accessible from within the VPC.

![image26](https://github.com/user-attachments/assets/58bff8f5-bddf-4f47-abfe-2cf2a272288b)

A Launch Template named `AppTemplate` was created using the previously generated AMI. 

![image27](https://github.com/user-attachments/assets/d8016b8d-c803-4bc6-a059-c81e9a2ef5ee)

Finally, an Auto Scaling group (`App Auto Scaling Group`) was configured using this launch template and target group. This Auto Scaling group automatically adjusts the number of application server instances based on load, distributing them across the configured Availability Zones.

![image28](https://github.com/user-attachments/assets/32b3da63-99f1-47b0-aeab-bf03a4af03ae)

## 6. Web Server Configuration

The presentation layer (web servers) was configured similarly. The initial connection was made via the Bastion Host to the previously created web server EC2 instance.

The Node.js environment (for potential front-end build tools) and Git were installed:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
sudo yum install git
```

The source code was cloned, and this time, the back-end server directory was removed:

```bash
git clone https://github.com/Adam-Zouari/DrOrlow.git web-tier
cd web-tier
rm -r server # Remove back-end files
```

The Nginx web server was installed and configured to serve static files (HTML, CSS, JavaScript) and to proxy dynamic requests (like form submissions) to the application tier's internal load balancer.

```bash
sudo yum install nginx
sudo truncate -s 0 /etc/nginx/nginx.conf # Clear default configuration
chmod -R 755 /home/ec2-user # Ensure permissions for Nginx
sudo chkconfig nginx on # Enable Nginx on startup
```

The configuration file `/etc/nginx/nginx.conf` was edited (e.g., using `sudo nano /etc/nginx/nginx.conf`) with the following content, adapted to serve files from `/home/ec2-user/web-tier/public` and proxy `/submit` requests to the internal load balancer (`internal-App-LB-*.elb.amazonaws.com`):

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format main 	'$remote_addr - $remote_user [$time_local] "$request" '
                      '	$status $body_bytes_sent "$http_referer" '
                      '	"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 4096;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen 80;
        listen [::]:80;
        server_name _;

        # Health check
        location /health {
            default_type text/html;
            return 200 "<!DOCTYPE html><p>Web Tier Health Check</p>\n";
        }

        # Front end files
        location / {
            root /home/ec2-user/web-tier/public;
            index Home.html "About me.html";
            try_files $uri /Home.html;
        }

        # Proxy requests to the internal application load balancer
        location /submit {
            proxy_pass http://<Internal Load Balancer DNS name>:80; (e.g http://internal-App-LB-612751870.us-east-1.elb.amazonaws.com:80;)
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

![image2a](https://github.com/user-attachments/assets/8fc13cdf-f040-49eb-af2e-2b4af324ca11)

After saving the configuration, the Nginx service was started and its status checked:

```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

![image2b](https://github.com/user-attachments/assets/428963eb-1503-4275-9a40-cfb50fed8948)

As with the application tier, an AMI was created from this configured web server instance.

![image2c](https://github.com/user-attachments/assets/bb3355fd-86d3-4be3-800f-70d033abbebc)
![image2d](https://github.com/user-attachments/assets/a5e24192-0190-4090-b54c-16dc92d3a8e9)

A Target Group were created.

![image2e](https://github.com/user-attachments/assets/1b2a6bd4-790c-456f-a5ab-9865aed2b66e)

Then, an internet-facing load balancer was set up to receive user traffic from the internet and distribute it to the web server instances.

![image2f](https://github.com/user-attachments/assets/9e578188-7e39-4446-a630-d670e23a5d1f)

Then, a launch Template (`WebTemplate`) was created.

![image30](https://github.com/user-attachments/assets/dd2553fd-76a2-4a10-98a2-639e78efe6c0)

Finally, an Auto Scaling group (`Web AutoScalingGroup`) was configured to automatically manage the web server instances using the `WebTemplate` and register them with the public load balancer.

![image31](https://github.com/user-attachments/assets/01a50741-754c-42f4-8647-fe3baba3e9fc)

## Conclusion

This configuration results in a resilient and scalable three-tier architecture on AWS. User traffic arrives at the public load balancer, is directed to the Auto Scaling-managed web servers, which serve static content and forward dynamic requests to the internal load balancer. The internal load balancer distributes requests to the application servers, also managed by Auto Scaling, which interact with the highly available RDS database. The bastion host provides secure access for maintenance.

### The hosted website

![image33](https://github.com/user-attachments/assets/0cd9d99d-fc47-45c8-8e68-3141de5d1b78)
