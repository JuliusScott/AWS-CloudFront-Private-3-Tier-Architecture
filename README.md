# AWS-CloudFront-Private-3-Tier-Architecture
Secure AWS architecture using CloudFront as the public entry point with an internal ALB, private EC2 instances, and private Amazon RDS.

## Project Overview

This project demonstrates a secure AWS architecture where Amazon CloudFront serves as the only public entry point.

CloudFront delivers static content from Amazon S3 and routes dynamic requests through a VPC origin to an internal Application Load Balancer. The ALB, EC2 instances, and Amazon RDS database remain private.

## Business Problem

A company needs to host a web application containing both static and dynamic content while reducing direct public exposure of its application infrastructure.

The company needs to:

- Deliver static content efficiently.
- Provide access to dynamic application content.
- Keep the Application Load Balancer and EC2 instances private.
- Keep the database private.
- Reduce the number of publicly exposed infrastructure components.

## Proposed Solution

Use Amazon CloudFront as the only public entry point for the application.

CloudFront uses two origins:

- Amazon S3 for static content.
- A CloudFront VPC origin connected to an internal Application Load Balancer for dynamic content.

Dynamic traffic flows through the private infrastructure:

CloudFront → Internal ALB → Private EC2 → Private RDS

## Benefits of the Solution

- CloudFront is the only public entry point.
- The Application Load Balancer and EC2 instances remain private.
- Amazon RDS remains isolated in private database subnets.
- Static and dynamic content are handled separately.
- Security groups control communication between each tier.
- AWS Systems Manager provides access to private EC2 instances without opening SSH to the internet.

## Architecture Flow

### Static Content
User → CloudFront → Amazon S3

### Dynamic Content
User → CloudFront → VPC Origin → Internal ALB → Private EC2 → Private RDS

## Project Goal

To design, deploy, troubleshoot, and validate an AWS architecture where Amazon CloudFront is the only public entry point while the Application Load Balancer, EC2 instances, and Amazon RDS database remain private.

## AWS Services Used

- Amazon CloudFront
- Amazon S3
- CloudFront VPC Origins
- Application Load Balancer (ALB)
- Amazon EC2
- Amazon RDS for MySQL
- Amazon VPC
- AWS Systems Manager
- AWS Identity and Access Management (IAM)

## Validation

The completed architecture was validated by confirming:

- Static content is successfully delivered from Amazon S3 through CloudFront.
- Dynamic requests are routed through CloudFront to the internal ALB.
- The internal ALB successfully reaches the private EC2 instances.
- Private EC2 instances successfully connect to Amazon RDS.
- Amazon RDS is not publicly accessible.
- EC2 instances can be managed through AWS Systems Manager.

## Troubleshooting & Lessons Learned

### Private EC2 Instances Unable to Connect to Systems Manager

**Problem:**  
The EC2 instances were running in private subnets and initially did not appear as managed nodes in AWS Systems Manager.

**Cause:**  
The instances did not have a private network path to the required Systems Manager services.

**Solution:**  
Created interface VPC endpoints for:

- `ssm`
- `ssmmessages`
- `ec2messages`

The endpoints were deployed across the private application subnets, allowing the EC2 instances to communicate with Systems Manager without requiring public IP addresses.

### Private EC2 Instances Unable to Connect to Amazon RDS

**Problem:**  
The private EC2 instances were initially unable to connect to the Amazon RDS MySQL database.

**Investigation:**  
Tested the connection from the EC2 instances using the RDS endpoint and MySQL client.

**Cause:**  
The required security group relationship between the application tier and database tier was not correctly configured.

**Solution:**  
Configured the RDS security group to allow inbound MySQL traffic on port `3306` from the EC2 security group.

Both private EC2 instances were then able to successfully connect to the RDS database.

### Dynamic Application Returned HTTP 500 Error

**Problem:**  
Static content from Amazon S3 worked through CloudFront, but requests to the dynamic `/api/` path returned an HTTP 500 error.

**Investigation:**  
Connected to the private EC2 instance using Systems Manager and tested the application locally.

Commands used included:

`curl -i http://localhost/api/time.php`

`sudo tail -50 /var/log/httpd/error_log`

`php -d display_errors=1 /var/www/html/api/time.php`

**Cause:**  
The RDS MySQL database required secure transport, while the PHP application was attempting an insecure database connection.

**Solution:**  
Updated the database connection configuration to account for the RDS secure transport requirement.

After correcting the database connection, the dynamic `/api/time.php` request successfully returned live database data through CloudFront.

### End-to-End Dynamic Request Validation

After resolving the database connectivity issue, the dynamic application path was successfully validated through CloudFront.

The request flow was:

`User → CloudFront → VPC Origin → Internal ALB → Private EC2 → Private RDS`

The final test used:

`/api/time.php`

The page successfully returned live data from Amazon RDS, confirming that the complete private application path was working as designed.

## Troubleshooting Commands

### Apache / Application

`httpd -v`

`ls -la /var/www/html/api/`

`curl -i http://localhost/api/time.php`

`sudo tail -50 /var/log/httpd/error_log`

`php -d display_errors=1 /var/www/html/api/time.php`

### SELinux

`getsebool httpd_can_network_connect_db`

`sudo setsebool -P httpd_can_network_connect_db 1`

### MySQL / RDS

`mysql -h <RDS-ENDPOINT> -u admin -p`

`SHOW DATABASES;`

`USE myappdb;`

`SELECT NOW();`

## Architecture Diagram

The architecture uses Amazon CloudFront as the only public entry point while the Application Load Balancer, EC2 instances, and Amazon RDS remain within private subnets.

![AWS CloudFront Private 3-Tier Architecture](architecture-diagram.png)

## Key Takeaways

- CloudFront can serve as the only public entry point for both static and dynamic content.
- CloudFront VPC Origins allow an internal ALB to remain private.
- EC2 instances can operate without public IP addresses.
- Security groups control communication between the ALB, EC2, and RDS tiers.
- VPC endpoints allow private EC2 instances to communicate with AWS Systems Manager.
- Troubleshooting each layer independently helped isolate application, networking, and database connectivity issues.

  
