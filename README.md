# AWS Three-Tier Architecture Configuration Guide

This document details the steps followed to set up a three-tier web architecture on Amazon Web Services (AWS), based on the initial report by Zouari Adem.

## 1. VPC (Virtual Private Cloud) Configuration

The first step involved establishing the isolated network environment. A VPC named "project" was created in the `us-east-1` region. To ensure high availability, this VPC spans two Availability Zones (AZs): `us-east-1a` (AZ1) and `us-east-1b` (AZ2). Within each of these zones, a subnet infrastructure was configured, comprising one public subnet and three private subnets. This segmentation allows for the separation of publicly accessible resources from those that must remain private.

To enable communication between the VPC and the internet, an Internet Gateway was created and attached to the "project" VPC. To allow instances located in the private subnets to access the internet for updates or external API calls without being directly exposed, two NAT (Network Address Translation) Gateways were set up, one in each Availability Zone.

Network traffic management within the VPC was configured using distinct route tables. A public route table was associated with the public subnets, directing traffic destined outside the VPC (0.0.0.0/0) to the Internet Gateway. For the private subnets, two separate route tables were created: "Private route table AZ1" for the private subnets in `us-east-1a` and "Private route table AZ2" for those in `us-east-1b`. The AZ1 private route table routes outbound traffic to the NAT Gateway located in `us-east-1a`. The configuration of the AZ2 private route table is similar, directing traffic to its own NAT Gateway in `us-east-1b` and being associated with the corresponding private subnets.

## 2. Security Group Setup

Following the network configuration, the next step was defining firewall rules using Security Groups. Several groups were created to finely control inbound and outbound traffic for each component of the architecture:

*   A security group for the database (RDS), restricting access to the database port solely from the application servers.
*   A security group for the bastion host, allowing SSH access from specific IP addresses for secure administration.
*   A security group for the internet-facing load balancer, permitting HTTP and HTTPS traffic from the internet.
*   A security group for the web servers, allowing traffic from the public load balancer and potentially the bastion host.
*   A security group for the internal load balancer, allowing traffic from the web servers.
*   A security group for the application servers, allowing traffic from the internal load balancer and potentially the bastion host, as well as outbound access to the database.

## 3. Database Deployment (RDS)

For the data persistence layer, a relational database instance was deployed using Amazon RDS (Relational Database Service). A MySQL instance of type `db.t3.micro` with 20 GB of storage was created and named `projectdb`. To ensure resilience, the primary instance was placed in Availability Zone AZ1 (`us-east-1a`), and a standby instance was configured in AZ2 (`us-east-1b`). RDS automatically handles replication and failover in case the primary instance fails.

## 4. Application Server Configuration

The application tier was set up starting with the creation of three initial EC2 instances:

1.  A Bastion Host EC2 instance to provide a secure access point to private instances.
2.  A Web Server EC2 instance (initially used for configuration).
3.  An Application Server EC2 instance (used as the base for the AMI).

An SSH key pair (`Host.pem`) was generated during the creation of the Bastion Host and reused for the other instances to simplify access management. An Elastic IP address was associated with the Bastion Host to provide a static public IP address and facilitate SSH connections.

Connection to the private servers is made via the Bastion Host. After connecting to the Bastion, the private key (`Host.pem`) was copied onto it, subsequently allowing an SSH connection to the application server.

On the application server, the necessary environment was installed. First, the MySQL client was installed to enable connection to the RDS database. The following commands were used (note: specific commands may vary slightly depending on the Linux AMI used; the example shows installation for an EL9-based distribution):

```bash
sudo yum install https://dev.mysql.com/get/mysql80-community-release-el9-5.noarch.rpm
sudo yum install mysql-community-server # Or mysql-community-client if only the client is needed
sudo systemctl enable --now mysqld # If the server is installed
```

The connection to the RDS database was established using the RDS instance endpoint:

```bash
mysql -h projectdb.cryby2rgpj6y.us-east-1.rds.amazonaws.com -P 3306 -u ProjectDB -p
# Enter the password 'ProjectDB' when prompted
```

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
git clone https://github.com/Adem-Zouari/DrOrlow.git app-tier
cd app-tier
rm -r public # Remove front-end files, not needed on the application server
cd server
npm install mysql express body-parser # Install back-end dependencies
pm2 start app.js # Start the application with PM2
pm2 save # Save PM2 configuration for automatic restarts
```

Once the application server was configured and the application was functional, an Amazon Machine Image (AMI) was created from this instance. This AMI serves as a template for launching new, identical application server instances.

To distribute incoming traffic to the application instances and improve availability, a Target Group named `AppTargetGroup` was created, followed by an Internal Load Balancer. This load balancer is only accessible from within the VPC.

A Launch Template named `AppTemplate` was created using the previously generated AMI. Finally, an Auto Scaling group (`App Auto Scaling Group`) was configured using this launch template and target group. This Auto Scaling group automatically adjusts the number of application server instances based on load, distributing them across the configured Availability Zones.

## 5. Web Server Configuration

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
git clone https://github.com/Adem-Zouari/DrOrlow.git web-tier
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
            proxy_pass http://internal-App-LB-612751870.us-east-1.elb.amazonaws.com:80; # Replace with your internal LB URL
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

After saving the configuration, the Nginx service was started and its status checked:

```bash
sudo systemctl start nginx
sudo systemctl status nginx
```

As with the application tier, an AMI was created from this configured web server instance. A Launch Template (`WebTemplate`) and a Target Group were created. Then, an internet-facing load balancer was set up to receive user traffic from the internet and distribute it to the web server instances. Finally, an Auto Scaling group (`Web AutoScalingGroup`) was configured to automatically manage the web server instances using the `WebTemplate` and register them with the public load balancer.

## Conclusion

This configuration results in a resilient and scalable three-tier architecture on AWS. User traffic arrives at the public load balancer, is directed to the Auto Scaling-managed web servers, which serve static content and forward dynamic requests to the internal load balancer. The internal load balancer distributes requests to the application servers, also managed by Auto Scaling, which interact with the highly available RDS database. The bastion host provides secure access for maintenance.
