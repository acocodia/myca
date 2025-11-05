# Arsenal Website - Automated AWS Deployment

This project automates the deployment of a static website to AWS EC2 using Infrastructure as Code (IaC) principles. It combines Terraform for infrastructure provisioning, Ansible for configuration management, and Docker for containerization.

##  Architecture

The deployment pipeline consists of:

1. **Terraform** - Provisions AWS infrastructure
   - EC2 instance (Amazon Linux 2, t3.micro)
   - Security Group with HTTP, HTTPS, and SSH access
   - SSH Key Pair for secure access

2. **Ansible** - Configures the server
   - Installs Docker and Docker Compose
   - Installs and configures Nginx as a reverse proxy
   - Deploys the web application

3. **Docker** - Containerizes the application
   - Nginx-based container serving static HTML
   - Managed via Docker Compose

4. **Web Application** - A modern single-page website about Dublin
   - Built with TailwindCSS
   - Responsive design
   - Served via Nginx

##  Prerequisites

Before you begin, ensure you have the following installed:

- [Terraform](https://www.terraform.io/downloads.html) (>= 1.0)
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) (>= 2.9)
- [AWS CLI](https://aws.amazon.com/cli/) configured with appropriate credentials
- SSH key pair (`alexander` and `alexander.pub`)

### AWS Requirements

- AWS account with appropriate permissions
- AWS credentials configured (`~/.aws/credentials` or environment variables)
- Permissions to create:
  - EC2 instances
  - Security Groups
  - Key Pairs

##  Quick Start

### 1. Configure AWS Credentials

```bash
aws configure
```

### 2. Generate SSH Key Pair (if not already present)

```bash
ssh-keygen -t rsa -b 4096 -f alexander -N ""
```

This will create two files:
- `alexander` (private key)
- `alexander.pub` (public key)

### 3. Initialize Terraform

```bash
terraform init
```

### 4. Review the Deployment Plan

```bash
terraform plan
```

### 5. Deploy Infrastructure

```bash
terraform apply
```

Type `yes` when prompted to confirm the deployment.

### 6. Access Your Website

After deployment completes (approximately 2-3 minutes), Terraform will output the public IP address:

```
instance_public_ip = "xx.xx.xx.xx"
```

Open your browser and navigate to:
```
http://xx.xx.xx.xx
```

##  Project Structure

```
CA1/
├── main.tf                    # Terraform configuration
├── deploy.yml                 # Ansible playbook
├── terraform.tfstate          # Terraform state file
├── terraform.tfstate.backup   # Terraform state backup
├── alexander                     # SSH private key
├── alexander.pub                 # SSH public key
└── webpage/
    ├── Dockerfile             # Docker image configuration
    ├── docker-compose.yaml    # Docker Compose configuration
    └── index.html             # Static website content
```

##  Configuration Details

### Terraform Configuration

**Provider**: AWS (us-west-2 region)

**Resources**:
- **Security Group** (`alexander_security_group`)
  - Port 80 (HTTP) - open to all
  - Port 443 (HTTPS) - open to all
  - Port 22 (SSH) - open to all
  - All outbound traffic allowed

- **EC2 Instance** (`alexander_server`)
  - AMI: `ami-0c5204531f799e0c6` (Amazon Linux 2)
  - Instance Type: `t3.micro`
  - Auto-provisioned with Ansible

- **Key Pair** (`alexander_key`)
  - Uses local public key file

### Ansible Playbook

The `deploy.yml` playbook performs the following tasks:

1. Installs Python 3.8
2. Installs Docker
3. Installs Docker Compose
4. Installs and configures Nginx as reverse proxy (port 80 → 8080)
5. Deploys the web application using Docker Compose

### Docker Configuration

- **Base Image**: nginx:alpine
- **Exposed Port**: 8080 (mapped to 80 internally)
- **Container Name**: mywebsite
- **Restart Policy**: always

##  Security Considerations

### Current Setup
 **Warning**: The current configuration has security groups open to the internet (0.0.0.0/0). This is suitable for development/testing but **NOT recommended for production**.

### Recommended Improvements

1. **Restrict SSH Access**:
```hcl
cidr_blocks = ["YOUR_IP/32"]  # Replace with your IP
```

2. **Use AWS Secrets Manager** for sensitive data
3. **Enable HTTPS** with SSL/TLS certificates
4. **Implement AWS IAM roles** instead of access keys
5. **Use private subnets** for better isolation
6. **Enable CloudWatch monitoring**

##  Management Commands

### SSH into the Instance

```bash
ssh -i alexander ec2-user@<instance_public_ip>
```

### View Docker Logs

```bash
ssh -i alexander ec2-user@<instance_public_ip> "cd /home/ec2-user/webpage && docker-compose logs"
```

### Restart the Application

```bash
ssh -i alexander ec2-user@<instance_public_ip> "cd /home/ec2-user/webpage && docker-compose restart"
```

### Update the Website

1. Modify `webpage/index.html`
2. Run:
```bash
terraform apply
```

Or manually copy files and restart:
```bash
scp -i alexander webpage/index.html ec2-user@<instance_public_ip>:/home/ec2-user/webpage/
ssh -i alexander ec2-user@<instance_public_ip> "cd /home/ec2-user/webpage && docker-compose up -d --build"
```

##  Cleanup

To destroy all resources and avoid AWS charges:

```bash
terraform destroy
```

Type `yes` when prompted to confirm.

##  Cost Estimation

Approximate AWS costs (us-west-2 region):
- **t3.micro instance**: ~$0.0104/hour (~$7.50/month)
- **Data transfer**: First 1GB free, then $0.09/GB
- **Total estimated cost**: ~$8-10/month for light usage

##  Troubleshooting

### Issue: Terraform times out during deployment

**Solution**: The playbook includes a 30-second sleep to allow the instance to fully boot. If issues persist, increase the sleep duration in `main.tf`:

```hcl
command = "sleep 60 && ANSIBLE_HOST_KEY_CHECKING=False ..."
```

### Issue: Cannot connect to the website

**Checks**:
1. Verify security group rules in AWS Console
2. Check instance state: `aws ec2 describe-instances`
3. SSH into instance and check Docker: `docker ps`
4. Check Nginx status: `systemctl status nginx`

### Issue: Permission denied (publickey)

**Solution**: Ensure the private key has correct permissions:
```bash
chmod 400 alexander
```

### Issue: Ansible playbook fails

**Solution**: SSH into the instance manually and check logs:
```bash
ssh -i alexander ec2-user@<instance_public_ip>
sudo journalctl -xe
```

##  Customization

### Change AWS Region

Edit `main.tf`:
```hcl
provider "aws" {
  region = "us-east-1"  # Change to your preferred region
}
```

**Note**: You'll also need to update the AMI ID to match the new region.

### Modify Website Content

Edit `webpage/index.html` to customize the website content, styling, or structure.

### Change Instance Type

Edit `main.tf`:
```hcl
resource "aws_instance" "alexander_server" {
  instance_type = "t3.small"  # Upgrade for better performance
  # ...
}
```

##  Contributing

Feel free to fork this project and customize it for your needs. Some ideas for enhancements:

- Add HTTPS support with Let's Encrypt
- Implement CI/CD with GitHub Actions
- Add monitoring with CloudWatch
- Use Auto Scaling Groups for high availability
- Implement blue-green deployments
- Add a database backend


##  Support

For issues or questions:
1. Check the troubleshooting section above
2. Review AWS and Terraform documentation
3. Check Ansible logs on the instance

---

