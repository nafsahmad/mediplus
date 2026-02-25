# Deploying Mediplus: A Production-Ready CI/CD Pipeline


## The Challenge


Most web applications require manual deployment—a developer builds the code locally, copies files to a server, restarts services, and hopes nothing breaks. This is slow, error-prone, and doesn't scale.


## The Solution

I built a fully automated DevOps pipeline for Mediplus, a medical tech web application. Now, any developer can push code to GitHub, and within minutes, it's automatically:
- Containerized with Docker
- Built and pushed to AWS ECR
- Deployed to production with HTTPS


Zero manual intervention. Zero downtime.


## Architecture


![Architecture Diagram](https://github.com/nafsahmad/mediplus/blob/main/img/architecture-diagram.png)

The system uses two EC2 instances:

1. **App Server** - Runs the containerized application (private, not exposed to internet)
2. **Proxy Server** - Handles HTTPS, SSL termination, and forwards traffic to the app

This separation improves security (app isn't directly accessible) and makes scaling easier (can add more app servers behind the same proxy later if traffic grows).

## Tech Stack

- **Containerization:** Docker
- **CI/CD:** GitHub Actions
- **Container Registry:** AWS ECR
- **Infrastructure:** AWS EC2, VPC, Security Groups
- **Reverse Proxy:** Nginx
- **SSL/TLS:** Let's Encrypt (Certbot)
- **Monitoring:** Portainer

**Live Demo:** https://narzcloudteam.online

---

## Prerequisites

To reproduce this project, you'll need:
- AWS account with ECR and EC2 access
- GitHub account
- Domain name (I used narzcloudteam.online from Namecheap)
- Basic understanding of Docker and Linux

## Estimated Costs

- EC2 instances: ~$15/month (can use free tier)
- Domain: ~$10/year
- ECR storage: Negligible for small images

---

## Implementation

### Step 1: Clone the Application Repository

I started by cloning the GitHub repository containing the Mediplus code:

```bash
git clone https://github.com/digitalwitchdemo/mediplus.git
cd mediplus
```

### Step 2: Containerize the Application

The repository already included a Dockerfile:

```dockerfile
# Get a base image from Docker Hub
FROM nginx:latest

RUN rm -rf /usr/share/nginx/html/*

# Define working directory
WORKDIR /usr/share/nginx/html

# Copy from host to container
COPY . /usr/share/nginx/html

# Expose port 
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

I used GitHub Actions instead of building locally because it ensures consistent builds regardless of the developer's machine, and it automatically triggers on every commit to main, making the deployment truly continuous.

**Workflow file (`.github/workflows/deploy.yml`):**

```yaml
name: CI/CD Pipeline - Mediplus App

on:
  push:
    branches: [ main ]

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v3

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Log in to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build, Tag, and Push Docker Image
      run: |
        IMAGE_URI=${{ secrets.REGISTRY_NAME }}:$GITHUB_RUN_NUMBER
        docker build -t $IMAGE_URI .
        docker push $IMAGE_URI
```

This builds the Docker image, tags it with the GitHub run number, and pushes to ECR automatically.

**How it works:**
- `IMAGE_URI` uses the GitHub run number to create a unique image tag each time the workflow runs
- `docker build -t $IMAGE_URI .` builds the Docker image directly from the repo
- `docker push $IMAGE_URI` pushes the image to ECR, making it available for deployment

**Required GitHub Secrets:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `REGISTRY_NAME` (ECR registry URL)

---

### Step 3: Preparing the AWS Infrastructure

I separated the application server from the proxy server for security and scalability. This way, the app instance doesn't need to be exposed directly to the internet, and I can add more app instances behind the same proxy later if traffic grows.

#### Application Instance

Runs the Mediplus Docker container.

**Specifications:**
- AMI: Amazon Linux 2
- Instance type: t3.micro
- Open ports: 22 (SSH), 8080 (app), 9000 (Portainer)

**Setup:**

```bash
# Install Docker
sudo amazon-linux-extras install docker -y
sudo systemctl start docker
sudo systemctl enable docker

# Install Portainer for container management
sudo docker run -d -p 9000:9000 --name portainer portainer/portainer-ce
```

#### Proxy Instance

Runs Nginx as a reverse proxy and handles SSL termination with Let's Encrypt.

**Specifications:**
- AMI: Amazon Linux
- Instance type: t3.micro
- Open ports: 22 (SSH), 80 (HTTP), 443 (HTTPS)

**Setup:**

```bash
# Install Nginx
sudo amazon-linux-extras install nginx1 -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

### Step 4: Deploying the Application Container

On the app instance, authenticate with ECR and pull the image:

```bash
# Authenticate with ECR
aws ecr get-login-password --region eu-north-1 | docker login --username AWS --password-stdin 032474760548.dkr.ecr.eu-north-1.amazonaws.com

# Pull and run the container
docker pull 032474760548.dkr.ecr.eu-north-1.amazonaws.com/mediplus:$GITHUB_RUN_NUMBER
docker run -d -p 8080:80 --name mediplus 032474760548.dkr.ecr.eu-north-1.amazonaws.com/mediplus:$GITHUB_RUN_NUMBER
```

---

### Step 5: Setting Up the Reverse Proxy

On the proxy instance, I configured Nginx to forward traffic to the app instance.

**Nginx configuration (`/etc/nginx/conf.d/mediplus.conf`):**

```nginx
server {
    listen 80;
    server_name narzcloudteam.online;

    # Redirect HTTP to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }

    # Allow Let's Encrypt validation
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}

server {
    listen 443 ssl;
    server_name narzcloudteam.online;

    ssl_certificate /etc/letsencrypt/live/narzcloudteam.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/narzcloudteam.online/privkey.pem;

    location / {
        proxy_pass http://172.31.X.X:8080;  # Replace with app instance private IP
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Important:** Replace `172.31.X.X` with your app instance's **private IP** address.

Create the directory for Let's Encrypt validation:

```bash
sudo mkdir -p /var/www/certbot
```

---

### Step 6: Securing with SSL

Install Certbot and obtain an SSL certificate:

```bash
# Install Certbot
sudo amazon-linux-extras install epel -y
sudo yum install certbot python3-certbot-nginx -y

# Obtain certificate
sudo certbot --nginx -d narzcloudteam.online
```

Certbot automatically configures Nginx and creates the certificate files:
- `/etc/letsencrypt/live/narzcloudteam.online/fullchain.pem`
- `/etc/letsencrypt/live/narzcloudteam.online/privkey.pem`

---

### Step 7: Testing the Setup

**Application:**
- URL: https://narzcloudteam.online
- Status: ✅ Loads successfully with HTTPS

**Portainer Dashboard:**
- URL: http://APP_PUBLIC_IP:9000
- Status: ✅ Shows container management interface

---

## Challenges and Solutions

### Challenge 1: 502 Bad Gateway Errors

**Problem:** After setting up Nginx, I kept getting 502 errors when accessing the site.

**Root Cause:** I was using the app instance's public IP in the Nginx config instead of the private IP. Since both instances were in the same VPC, they should communicate privately.

**Solution:** Changed `proxy_pass` to use the private IP (172.31.X.X:8080) instead of the public IP. Added proper security group rules to allow port 8080 traffic between instances.

**Lesson:** Always use private networking within a VPC for internal communication. It's more secure and doesn't incur data transfer costs.

---

### Challenge 2: Let's Encrypt Certificate Validation Failing

**Problem:** Certbot couldn't validate my domain because the `.well-known/acme-challenge` path wasn't accessible.

**Root Cause:** My Nginx config was redirecting ALL HTTP traffic to HTTPS before the certificate existed.

**Solution:** Added a specific location block for `/.well-known/acme-challenge/` that serves files from `/var/www/certbot` before the redirect happens. This allows Let's Encrypt to validate the domain via HTTP while redirecting all other traffic to HTTPS.

**Lesson:** SSL certificate validation requires temporary HTTP access. Always create an exception for the ACME challenge path before forcing HTTPS redirects.

---

### Challenge 3: Docker Image Not Pulling from ECR

**Problem:** `docker pull` command was failing with authentication errors.

**Root Cause:** I hadn't logged into ECR on the EC2 instance.

**Solution:** Ran `aws ecr get-login-password` and piped it to `docker login` to authenticate properly:

```bash
aws ecr get-login-password --region eu-north-1 | docker login --username AWS --password-stdin 032474760548.dkr.ecr.eu-north-1.amazonaws.com
```

**Lesson:** ECR requires explicit authentication. AWS credentials alone aren't enough—you must perform the login step before pulling images.

---

## Key Takeaways

### Technical Skills Gained

- Dockerfile best practices (base images, layer optimization, proper CMD usage)
- AWS ECR authentication and image versioning strategies
- GitHub Actions secrets management and workflow automation
- Nginx reverse proxy configuration for production environments
- Let's Encrypt certificate automation and renewal workflows
- Container management with Portainer
- AWS VPC private networking and security group configuration

### DevOps Principles Learned

- **Separation of Concerns:** App server vs proxy server improves security and enables independent scaling
- **Automation First:** CI/CD reduces human error and makes deployments predictable
- **Private Networking:** Internal communication should never traverse the public internet
- **SSL Termination:** Handling certificates at the proxy layer simplifies application configuration
- **Infrastructure Documentation:** Every decision should be documented for team knowledge sharing

### What I'd Do Differently Next Time

- **Use Terraform** to provision the infrastructure instead of manual EC2 setup
- **Add automated health checks** in the deployment pipeline to catch failures before they reach production
- **Implement blue-green deployment** for true zero-downtime updates
- **Add monitoring** with CloudWatch or Prometheus/Grafana for observability
- **Create a staging environment** to test changes before production deployment
- **Automate certificate renewal** with a cron job or Lambda function

---

## What's Next

This project demonstrates a working CI/CD pipeline, but there's room to grow:

### Immediate Improvements

- Migrate infrastructure provisioning to Terraform
- Add CloudWatch monitoring and alerting for container health
- Implement automated rollback on failed deployments
- Add integration tests to the CI/CD pipeline

### Long-Term Enhancements

- Move from single EC2 instances to Kubernetes (EKS) for better scalability
- Implement a proper staging → production promotion workflow
- Add blue-green or canary deployments for zero-downtime updates
- Implement secrets management with AWS Secrets Manager
- Add automated backups and disaster recovery procedures

These improvements will be tackled in future projects as I continue building production-grade DevOps workflows.

---

## Technologies Used

| Category | Technology |
|----------|-----------|
| Containerization | Docker |
| CI/CD | GitHub Actions |
| Container Registry | AWS ECR |
| Compute | AWS EC2 |
| Networking | AWS VPC, Security Groups |
| Reverse Proxy | Nginx |
| SSL/TLS | Let's Encrypt (Certbot) |
| Container Management | Portainer |
| Version Control | Git, GitHub |

---

## Repository Structure

```
mediplus/
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions CI/CD pipeline
├── Dockerfile                   # Container definition
├── README.md                    # This file
├── architecture-diagram.png     # System architecture diagram
└── [application files]          # Mediplus web application code
```

---

## Author

**Nafs Ahmad**

- LinkedIn: [linkedin.com/in/nafs-ahmad](https://linkedin.com/in/nafs-ahmad)
- Medium: [@nafsahmad](https://medium.com/@nafsahmad)
- GitHub: [@nafsahmad](https://github.com/nafsahmad)

---

## License

This project is for educational and portfolio purposes.

---

## Acknowledgments

- Original Mediplus application: [digitalwitchdemo/mediplus](https://github.com/digitalwitchdemo/mediplus)
- AWS documentation for ECR and EC2 best practices
- Let's Encrypt for free SSL certificates
- The DevOps community for continuous learning resources