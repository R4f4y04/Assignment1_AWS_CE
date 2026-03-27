# Assignment 1: AWS Cloud Engineering - UniEvent Deployment
**Submitted by:** Rafay Akram  BS CS  2023491

## Project Overview
This repository contains the source code and deployment documentation for **UniEvent**, a Node.js web application integrating the Ticketmaster API. The system is architected and deployed on Amazon Web Services (AWS) strictly adhering to cloud best practices, focusing on high availability, secure private networking, and fault tolerance.

## Architecture & Infrastructure Design
The application infrastructure was built from scratch to ensure a secure and scalable environment:
* **Virtual Private Cloud (VPC):** Configured with 2 Public Subnets and 2 Private Subnets spanning 2 Availability Zones (`ap-southeast-2a` and `ap-southeast-2b`) for strict network isolation.
* **Compute (EC2):** The Node.js application runs on `t3.micro` EC2 instances deployed exclusively within the Private Subnets. 
* **Load Balancing (ALB):** An Internet-facing Application Load Balancer sits in the Public Subnets, securely proxying HTTP traffic to the private backend instances.
* **Outbound Connectivity:** A NAT Gateway is provisioned in the public subnet, allowing the private EC2 instances to securely fetch external event data from the public Ticketmaster API.
* **Storage (S3):** Event posters are uploaded and stored in a private S3 bucket. Access is governed via IAM Roles attached to the EC2 instances.


## Deployment Methodology

### 1. Network & Security Provisioning
* Created the custom VPC and subnets.
* Configured an ALB Security Group (allowing inbound HTTP via port 80) and a Private EC2 Security Group (allowing inbound traffic on port 3000 strictly from the ALB).
* Created an IAM Role (`AmazonS3FullAccess` and `AmazonSSMManagedInstanceCore`) to allow seamless, secure access to the S3 bucket and to enable AWS Systems Manager (Session Manager) connectivity without relying on legacy SSH keys.

### 2. Application Deployment (Primary Server)
* Launched the first EC2 instance (`UniEvent-Server-1`) into Private Subnet A.
* Used AWS Session Manager to securely tunnel into the instance.
* Installed Node.js, pulled the application zip file directly from S3, installed dependencies, and started the application background process using `pm2`.


### 3. Load Balancer Configuration
* Created an EC2 Target Group pointing to the Node.js application on port 3000.
* Provisioned the Application Load Balancer across the two Public Subnets and routed default traffic to the Target Group.


### 4. Fault Tolerance & High Availability Implementation
* To eliminate single points of failure, an Amazon Machine Image (AMI) was created from the configured primary server.
* Launched a secondary instance (`UniEvent-Server-2`) from this AMI into Private Subnet B (the second Availability Zone).
* Registered the secondary instance with the ALB Target Group to distribute traffic and ensure continuous uptime even if one Availability Zone experiences an outage.


## Challenges Faced & Technical Resolutions

During the deployment phase, a few architectural challenges were encountered and resolved:

**1. Load Balancer Health Check Failures (HTTP 302):**
* **Issue:** The ALB initially marked the backend targets as "Unhealthy."
* **Resolution:** Diagnosed that the Node.js application was issuing a standard HTTP 302 (Redirect) for specific internal routing, which the ALB interpreted as a failure. Updated the Target Group's advanced health check settings to accept `200,302` as success codes, stabilizing the routing.

**2. Application Process Halting on AMI Creation:**
* **Issue:** After creating the blueprint AMI to clone the server for fault tolerance, both the original and cloned EC2 instances passed AWS status checks, but the application went offline. 
* **Resolution:** Identified that the AMI creation process triggers an automated safe-reboot of the EC2 instance. Because `pm2` was not configured to run on startup, the Node app remained offline. Tunnelled back into both instances via Session Manager and manually restarted the `pm2` daemon to restore service.

**3. Strict HTTPS Browser Forcing:**
* **Issue:** Initial attempts to access the ALB DNS resulted in a timeout.
* **Resolution:** Determined that modern browsers were defaulting to HTTPS (Port 443), while the ALB Security Group was strictly configured for HTTP (Port 80) per assignment scope. Explicitly appending `http://` to the request bypassed the local browser override and successfully reached the secure backend.




Link to test deployed website : http://unievent-alb-1550279616.ap-southeast-2.elb.amazonaws.com/events
