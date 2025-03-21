# Domain Monitor Infrastructure

Infrastructure as Code (IaC) for deploying the Domain Monitoring System, including Terraform configurations, Ansible roles, and Kubernetes manifests.

## Project Overview

This repository contains the complete infrastructure configuration for the Domain Monitoring System, including:

- Terraform configurations for AWS resources
- Ansible roles for configuring EC2 instances
- Kubernetes manifests for containerized deployments

## Ansible Role Details

### ansible-agent Role

This role prepares an EC2 instance to serve as a deployment agent by:

1. Installing Ansible and required dependencies
2. Copying security files from `roles/ansible-agent/files/`:
   - SSH key (your private key file) for connecting to other instances
   - AWS credentials for dynamic inventory
   - AWS inventory configuration
3. Cloning the deployment repository
4. Setting up necessary permissions

The ansible-agent serves as the execution environment for the deployment playbooks and is triggered by the Jenkins pipeline during the CI/CD process.

## Required Customization for Your Environment

Before deploying the infrastructure, you'll need to modify several configuration elements to match your environment:

1. **SSH Key Name**: The current configuration uses a variable for the SSH key name:
   ```terraform
   variable "key_name" {
     description = "Name of the AWS rsa key to use"
     type        = string
     default     = "MoniNordic"
   }
   ```
   You must either:
   - Create a key pair with the name "MoniNordic" in AWS, OR
   - Change the default value to match your existing key name, OR
   - Override the variable when running Terraform:
     ```bash
     terraform apply -var="key_name=YOUR_KEY_NAME"
     ```

2. **AWS Region**: Update the region in both files:
   - In `main.tf`:
     ```terraform
     provider "aws" {
       region = "us-west-2"  # Change to your preferred region
     }
     ```
   - In `inventory_aws_ec2.yaml`: Update the regions section as shown in the AWS Inventory customization section

3. **Security Groups**: The current configuration uses a hardcoded security group ID:
   ```terraform
   vpc_security_group_ids = ["sg-02b3d29bdcd49a0cc"]  # Change to your security group ID
   ```
   You'll need to:
   - Create a security group with appropriate rules for your instances, OR
   - Update this ID to match an existing security group you want to use

4. **AMI ID**: The current configuration uses a specific Amazon Linux AMI:
   ```terraform
   ami = "ami-05d38da78ce859165"  # Change to an appropriate AMI for your region
   ```
   AMI IDs vary by region, so you'll need to:
   - Find an appropriate AMI ID for your chosen region
   - Update this value in all instance resources

### File Naming

When using your own SSH key and credentials:

1. Place your private key file in the appropriate locations:
   ```bash
   # Replace YOUR_KEY_NAME.pem with your actual key filename
   cp /path/to/your/YOUR_KEY_NAME.pem .
   cp /path/to/your/YOUR_KEY_NAME.pem roles/ansible-agent/files/
   chmod 400 YOUR_KEY_NAME.pem
   chmod 400 roles/ansible-agent/files/YOUR_KEY_NAME.pem
   ```

2. Update references in code to match your key name:
   - Search for "MoniNordic" throughout the codebase and replace with your key name
   - Check Ansible playbooks for hard-coded key paths

## Components

### Terraform (main.tf)

Terraform configuration provisions the AWS infrastructure including:

- EC2 instances for various roles (Jenkins, agents, production)
- Security groups
- Application Load Balancer
- Networking components

### Ansible Roles

The Ansible roles configure the provisioned infrastructure:

- **common_server**: Base configuration for all EC2 instances
- **jenkins**: Jenkins server setup with required plugins and configurations
- **docker_agent**: Build agent for Docker image creation and testing
- **ansible_agent**: Deployment agent that pulls and runs the deployment repository

### Kubernetes

Kubernetes manifests for containerized deployments:

- Backend API (BE.yaml)
- Frontend Web UI (FE.yaml)
- Database (db.yaml)
- Supporting services and configurations

## Initial Setup

1. Clone the repository
   ```bash
   git clone https://github.com/RazielRey/domain-monitor-infra.git
   cd domain-monitor-infra
   ```

2. Place your AWS SSH key in the required locations
   ```bash
   # Replace with your actual key filename
   cp /path/to/your/YOUR_KEY_NAME.pem .
   cp /path/to/your/YOUR_KEY_NAME.pem roles/ansible-agent/files/
   chmod 400 YOUR_KEY_NAME.pem
   chmod 400 roles/ansible-agent/files/YOUR_KEY_NAME.pem
   ```

3. Configure AWS credentials
   ```bash
   # Create or copy your AWS credentials file
   cp ~/.aws/credentials roles/ansible-agent/files/aws_credentials
   ```

4. Update key names and region references as described in the "Required Customization" section

5. Prepare for CI/CD pipeline by gathering:
   - Docker Hub credentials (username and password/token)
   - GitHub credentials (if using private repositories)
   - These will be added to Jenkins after infrastructure deployment

## Security-Sensitive Files

For security reasons, the following files should be created locally and **NOT committed to the repository**:

1. **Your SSH private key** (.pem file): For AWS instance access
2. **aws_credentials**: AWS API credentials file
3. **aws_accessKeys.csv**: AWS access key information
4. **terraform.tfstate**: Terraform state file (contains sensitive information)

> **IMPORTANT**: Never commit security credentials to Git repositories. The `.gitignore` file should exclude these sensitive files.

## Usage

### Terraform Deployment

1. Initialize Terraform
   ```bash
   terraform init
   ```

2. Plan the infrastructure
   ```bash
   terraform plan
   ```

3. Apply the configuration
   ```bash
   terraform apply
   ```

### Ansible Configuration

After Terraform has provisioned the infrastructure:

1. Run the Ansible playbook
   ```bash
   ansible-playbook -i inventory_aws_ec2.yaml playbook.yaml
   ```

### Jenkins Configuration

After the infrastructure is provisioned and Jenkins is running:

1. Access the Jenkins UI using the public IP of the Jenkins instance
2. Navigate to Manage Jenkins > Manage Credentials > System > Global credentials
3. Add the required credentials:
   - Add Docker Hub credentials with ID 'docker'
   - Add any other credentials needed for the pipeline

### Kubernetes Deployment

For containerized deployments:

1. Configure kubectl to access your cluster
   ```bash
   kubectl config use-context your-cluster-context
   ```

2. Apply the Kubernetes manifests
   ```bash
   kubectl apply -f k8s/yamls/
   ```

## Workflow Integration

This infrastructure repository integrates with other components:

1. **Terraform** provisions the AWS resources
2. **Ansible roles** configure the EC2 instances
3. The **ansible_agent** role clones the deployment repository
4. **Jenkins** pipeline in the deployment repo orchestrates application deployment
5. **Kubernetes** manifests provide an alternative containerized deployment option

## Terraform Resources

Key resources defined in main.tf:

- **EC2 Instances**: Jenkins server, Docker agent, Ansible agent, Production servers
- **Application Load Balancer**: Distributes traffic to production instances
- **Security Groups**: Controls access to instances
- **Target Groups**: Routes traffic to appropriate instances

## Prerequisites

- [Terraform](https://www.terraform.io/downloads.html) v1.0.0+
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/index.html) 2.9+
- [kubectl](https://kubernetes.io/docs/tasks/tools/) for Kubernetes deployments
- AWS CLI configured with appropriate credentials
- AWS SSH key pair (you will need to update references in the code to match your key name)

## Kubernetes Configuration

The Kubernetes manifests include:

- Deployments for each application component
- Services for internal and external access
- RBAC configuration for permissions
- Secrets for sensitive configuration

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -am 'Add my feature'`
4. Push to the branch: `git push origin feature/my-feature`
5. Submit a pull request

## License

[MIT License](LICENSE)