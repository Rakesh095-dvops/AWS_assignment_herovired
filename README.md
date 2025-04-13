# MERN application SetUp using AWS services

### Introduction:

The Travel Memory application has been developed using the MERN stack. Your challenge is to deploy this application on an Amazon EC2 instance. This will provide you with hands-on experience in deploying full-stack applications, working with cloud platforms, and ensuring scalable architecture.

### Project Repository:

Access the complete codebase of the TravelMemory application from the provided GitHub link: https://github.com/UnpredictablePrashant/TravelMemory

### Objective:

- Set up the backend running on Node.js.
- Configure the front end designed with React.
- Ensure efficient communication between the front end and back end.
- Deploy the full application on an EC2 instance.
- Facilitate load balancing by creating multiple instances of the application.
- Connect a custom domain through Cloudflare.

_____
## Table Of Content 

1. [Architecture Diagram](#1-architecture-diagram)
2. [EC2 Instance Setup](#2ec2-instance-setup)
3. [Backend Configuration](#3-backend-configuration)
4. [Frontend Configuration](#4-frontend-configuration)
5. [Nginx Configuration](#5-nginx-configuration)
6. [Scaling the Application](#6-scaling-the-application)
7. [Load Balancing Setup](#7-load-balancing-setup)
8. [Cloudflare Integration](#8-cloudflare-integration)
9. [Loadbalancer and DNS-verification](#9-loadbalancer-and-dns-verification)
10. [Appendix]()

### 1. Architecture Diagram
![alt text](image/AWSBASICS_HRV.jpg)

### 2.EC2 Instance Setup
1.  **Log in to the AWS Management Console.**
2.  **Navigate to EC2:** Services -> EC2
3.  **Launch Instance:** Click "Launch Instance".
4.  **Choose an AMI:** Select "Ubuntu Server 20.04 LTS (HVM), SSD Volume Type" (or a later version).  
5.  **Choose Instance Type:** Select `t2.micro` (free tier eligible).
6.  **Configure Instance Details:**
    *   **Network:** Choose your VPC. If you don't have one, accept the default.
    *   **Subnet:** Choose a public subnet in your VPC to allow internet access.
    *   **Auto-assign Public IP:**  Enable "Auto-assign Public IP" to ensure instances get a public IP address (for initial setup and testing; remove in production and configure a NAT gateway if you want to be fully private).
    *   Leave the rest as default.
7.  **Add Storage:**  The default 8 GB is usually sufficient for this example.  Adjust as needed.
8.  **Add Tags:** Add a tags for better identification
9.  **Configure Security Group:**
    *   Create a new security group (or choose an existing one if applicable).
    *   **Allow SSH (port 22):**  Source: Your IP address (for secure access).
    *   **Allow HTTP (port 80):** Source: 0.0.0.0/0 
    *   **Allow HTTPS (port 443):** Source: 0.0.0.0/0 
10. **Review and Launch:** Review your configuration and click "Launch".
11. **Key Pair:** Choose an existing key pair or create a new one.  **Important:** Store the private key file (`.pem`) securely. You will need it to SSH into the instances.
12. Instance will be initiated and once ec2 health status is okay . connect via earlier shared key to login. 
![alt text](image/EC2_instance.png)

## 3. Backend Configuration

### Install Required Packages
```bash
# Update package lists
sudo apt update -y && sudo apt upgrade -y

# Install essential tools
sudo apt install -y git nginx certbot python3-certbot-nginx

# Install Node.js 18.x (LTS)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install PM2 process manager globally
sudo npm install -g pm2

# Install build tools (for node-gyp)
sudo apt install -y build-essential
```
**Verification**

![alt text](image/installation_verification.png)

### Clone and Setup Backend

```bash
# Clone the repository
git clone https://github.com/UnpredictablePrashant/TravelMemory
cd TravelMemory/Backend

# Install dependencies
npm install

# Create .env file with database credentials
cat <<EOT > .env
PORT=3000
MONGODB_URI=mongodb+srv://username:password@cluster0.example.mongodb.net/travelmemory?retryWrites=true&w=majority
EOT

# Start the backend with PM2
pm2 start index.js --name "travelmemory-backend"
pm2 save
pm2 startup
```
**Verification**

![alt text](image/backendfile_list.png)

```nodejs
pm2 list
```
![alt text](image/pm2_verification.png)

## 4. Frontend Configuration
updated Urls.js with Frontend load balancer created and configured in cloudflare DNS 

```bash
cd ../Frontend
npm install

# Update API endpoints (urls.js)
# Change the base URL to your domain or ALB endpoint
nano ~/TravelMemory/frontend/src/url.js
nano ~/TravelMemory/frontend/.env
# Build production version
npm run build
```

**Verification**

![alt text](image/frontendfile_list.png)

## 5. Nginx Configuration

1. Take backup of existing default OOTB file and create default file to configure MERN  Application
```bash 
cd etc/nginx/sites-available/
sudo mv default default.old

sudo nano /etc/nginx/sites-available/default
```
2. Update the nginx config ([default](default))

3. Nginx configuration verification 

```bash
sudo nginx -t # configuration verification
sudo systemctl reload nginx # reload the nginx 
sudo systemctl restart nginx # restart the nginx
sudo systemctl status nginx # status of nginx 
```
**Verification**

![alt text](image/Nginx_config.png)

## 6. Scaling the Application

1. Navigate to EC2 Instance -> Actions ->Image and templates -> Create Image
![alt text](image/image_AMI_Creation.png)
2. Once Image is created it is available under AMIs
![alt text](image/image-AMI.png)

3. Follow same steps as [ec2-instance-setup](#2ec2-instance-setup) , however instead of ubuntu, select earlier created AMI,security group and key-pair.
![alt text](image/EC2-2_1.png)
![alt text](image/EC2-2-2.png)

## 7. Load Balancing Setup
1. Go to EC2 Dashboard → Load Balancers → Create Load Balancer
2. Select "Application Load Balancer"
    - Configure:
        - Name: rik-mern-TravelMemory-lb
        - Scheme: Internet-facing
        - Listeners: HTTP (80)
    - Target Groups:
        - Name: 
rikhrv-mern-TravelMemory-TG
        - Protocol: HTTP
        - Port: 80
3. Health Check Path: / (or /api/health)
4. Register Targets: Add both EC2 instances
5. Create & Verify ALB DNS by checking DNS Name 
![alt text](image/ALB-1.png)
![alt text](image/ALB-2.png)
## 8. Cloudflare Integration
- Log in to Cloudflare dashboard
- Go to DNS > Records
- Add records and provide appropriate target from earlier created ALB
    ```
    Type: CNAME
    Name: travelmemory
    Target: your-alb-dns-name.region.elb.amazonaws.com 
    TTL: Auto
    ```
- Add another Record:
    ```
    Type: CNAME
    Name: apitravelmemory
    Target: travelmemory.your-site-dns.site
    TTL: Auto
    ```

## 9. LoadBalancer and DNS Verification 

1. Navigate up to earlier created Target group verify that ***healthcheck***, if DNS has been configured correctly and nginx has been correctly configured ,Status should be ***green***

    ![alt text](image/Loadbalancer_healthcheck.png)
2. Frontend URL - ```travelmemory.rakeshchoudhury.site```

    - Website is now available and MongoDB data is being accessible
        ![alt text](image/frontend_image-0.png)
    - Push entry to MONGO DB Using Add Experience form 
        ![alt text](image/frontend_image-1.png)
    - Once Entry has been pushed , data is reflected in home page,more details are available under "more details"
        ![alt text](image/frontend_image-2.png)
       
3. Backend URL ```apitravelmemory.rakeshchoudhury.site```
- As per Nginx configs only hello and trip end point are accessible.
    - apitravelmemory.rakeshchoudhury.site/hello
       ![alt text](image/Backend_image_hello.png)
    - apitravelmemory.rakeshchoudhury.site/trip
      ![alt text](image/Backend_image_trip.png)

4. Post Request Backend 
 - POST request is also working,tested the same using postman. Received response "200". 
        ![alt text](image/Backend_postman_POST_REQ.png)
 - Frontend shows same data 
        ![alt text](image/Frontend_postman_ver_1.png)
        ![alt text](image/Frontend_postman_ver_2.png)
 - get specific API details 
        ![alt text](image/Backend_postman_POST_REQ_specific.png)

## Appendix





