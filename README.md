# Dynamic Website Hosting on AWS – DevOps Project

## Overview

This project demonstrates the deployment of a dynamic, scalable, and secure web application on AWS using a variety of core AWS services. The infrastructure is defined and managed using scripts, and all resources are provisioned to ensure high availability, security, and automation.  
A [GitHub repository](#) contains the reference architecture diagram and all deployment scripts.

---

## AWS Resources and Architecture

1. **Virtual Private Cloud (VPC):**  
   Configured with both public and private subnets spanning two availability zones for high availability.

2. **Internet Gateway:**  
   Enables connectivity between VPC instances and the internet.

3. **Security Groups:**  
   Serve as network firewalls to control inbound and outbound traffic.

4. **Multi-AZ Deployment:**  
   Resources are distributed across two Availability Zones for fault tolerance.

5. **Public Subnets:**  
   Used for the NAT Gateway and Application Load Balancer.

6. **EC2 Instance Connect Endpoint:**  
   Provides secure SSH access to instances in both public and private subnets.

7. **Private Subnets:**  
   Web servers (EC2 instances) are placed here for enhanced security.

8. **NAT Gateway:**  
   Allows instances in private subnets to access the internet.

9. **EC2 Instances:**  
   Host the dynamic website.

10. **Application Load Balancer (ALB) & Target Group:**  
    Distributes web traffic evenly to an Auto Scaling Group of EC2 instances across multiple AZs.

11. **Auto Scaling Group:**  
    Automatically manages EC2 instances for availability, scalability, and elasticity.

12. **AWS Certificate Manager:**  
    Secures application communications with SSL/TLS certificates.

13. **Simple Notification Service (SNS):**  
    Sends alerts about activities within the Auto Scaling Group.

14. **Route 53:**  
    Domain registration and DNS record management.

15. **S3 Buckets:**  
    Store application code and migration scripts.

---

## Data Migration Script (RDS)

The following script migrates SQL data from S3 to an AWS RDS MySQL instance using Flyway:

```bash
#!/bin/bash

# Define variables
S3_URI=s3://oliver-sql-files/V1__shopwise.sql
RDS_ENDPOINT=dev-rds-db.coem6gby0vqn.us-east-1.rds.amazonaws.com
RDS_DB_NAME=applicationdb
RDS_DB_USERNAME=admin
RDS_DB_PASSWORD=cherches

# Clean up previous installations to avoid permission issues
sudo rm -rf flyway-*
sudo rm -f /usr/local/bin/flyway
rm -rf sql

# Update all packages
sudo yum update -y

# Install Java if not already installed
if ! command -v java &> /dev/null; then
    sudo yum install java-17-amazon-corretto -y
fi

# Download and extract Flyway
wget -qO- https://download.red-gate.com/maven/release/com/redgate/flyway/flyway-commandline/11.8.3/flyway-commandline-11.8.3-linux-x64.tar.gz | tar -xvz
sudo ln -s $(pwd)/flyway-11.8.3/flyway /usr/local/bin

# Create the SQL directory for migrations
mkdir -p sql

# Download the migration SQL script from AWS S3
aws s3 cp "$S3_URI" sql/V1__shopwise.sql

# Set permissions for Flyway to read the SQL file
chmod 644 sql/V1__shopwise.sql

# Run Flyway migration with SSL certificate validation disabled for testing
flyway -url="jdbc:mysql://$RDS_ENDPOINT:3306/$RDS_DB_NAME?useSSL=true&trustServerCertificate=true" \
  -user="$RDS_DB_USERNAME" \
  -password="$RDS_DB_PASSWORD" \
  -locations=filesystem:sql \
  migrate

echo "Migration complete!"

---

## Web Application Deployment Script (EC2)
This script provisions the web server, installs dependencies, and deploys the application code from S3:

bash
Copy Code
#!/bin/bash

# Update all packages
sudo yum update -y

# Install Apache web server
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP and required extensions
sudo dnf install -y \
php \
php-pdo \
php-openssl \
php-mbstring \
php-exif \
php-fileinfo \
php-xml \
php-ctype \
php-json \
php-tokenizer \
php-curl \
php-cli \
php-fpm \
php-mysqlnd \
php-bcmath \
php-gd \
php-cgi \
php-gettext \
php-intl \
php-zip

# Install MySQL 8
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Enable mod_rewrite in Apache
sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

# Download application code from S3
S3_BUCKET_NAME=oliver-project-web-files
sudo aws s3 sync s3://"$S3_BUCKET_NAME" /var/www/html

cd /var/www/html
sudo unzip shopwise.zip
sudo cp -R shopwise/. /var/www/html/
sudo rm -rf shopwise shopwise.zip

# Set permissions
sudo chmod -R 777 /var/www/html
sudo chmod -R 777 storage/

# Edit .env for database credentials
sudo vi .env

# Restart Apache
sudo service httpd restart
Issue Encountered and Solution
Issue
When running the Flyway migration script, the following error occurred:

PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
This error means Java (used by Flyway) did not trust the SSL certificate presented by the AWS RDS MySQL instance.
Additionally, permission errors occurred due to files and directories created by sudo in previous runs.

Solution
Cleaned Up Old Files:
The script was updated to remove any existing Flyway directories, symlinks, and the sql directory before running, ensuring no permission conflicts.
Fixed Permissions:
All new files and directories are created by the current user, not root, to avoid permission issues.
Bypassed SSL Validation:
The JDBC URL in the Flyway command was updated to include trustServerCertificate=true, which tells the MySQL driver to accept the server’s certificate without validation.
Note: This is suitable for development and testing. For production, import the AWS RDS root CA certificate into the Java keystore.

Ensured Java Installation:
The script checks for Java and installs it if missing, ensuring Flyway can run.
References
AWS RDS SSL Documentation
Flyway Documentation
AWS EC2 User Guide
License
This project is for educational and demonstration purposes.
