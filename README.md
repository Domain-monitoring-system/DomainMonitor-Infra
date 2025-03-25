# Domain Monitor Infrastructure

Infrastructure as Code (IaC) for deploying the Domain Monitoring System, including Terraform configurations and Ansible roles.

## Project Overview

This repository contains the complete infrastructure configuration for the Domain Monitoring System, including:

- Terraform configurations for AWS resources
- Ansible roles for configuring EC2 instances

## Repository Structure

```
domain-monitor-infra/
├── roles/
│   ├── jenkins/
│   │   ├── tasks/
│   │   │   └── main.yaml
│   │   ├── templates/
│   │   │   ├── init-script.groovy.j2
│   │   │   ├── install-plugins.groovy.j2
│   │   │   ├── node-creation.groovy.j2
│   │   │   ├── docker-pipeline-creation.groovy.j2
│   │   │   └── ansible-pipeline-creation.groovy.j2
│   │   └── vars/
│   │       └── main.yaml
│   ├── docker-agent/
│   │   ├── tasks/
│   │   │   └── main.yaml
│   │   └── templates/
│   │       ├── service.sh.j2
│   │       └── jenkins-agent.service.j2
│   └── ansible-agent/
│       ├── tasks/
│       │   └── main.yaml
│       ├── templates/
│       │   ├── service.sh.j2
│       │   └── jenkins-agent.service.j2
│       └── files/
│           ├── aws_credentials           # AWS credentials file (not committed to repository)
│           ├── MoniNordic.pem            # SSH private key (not committed to repository)
│           └── inventory_aws_ec2.yaml    # AWS EC2 dynamic inventory file
├── main.tf                              # Main Terraform configuration
├── inventory_aws_ec2.yaml               # Ansible dynamic inventory configuration
├── playbook.yaml                        # Main Ansible playbook
├── .gitignore
└── README.md                            # This documentation
```

## Components

### Terraform Configuration

The Terraform configuration (`main.tf`) provisions:

- EC2 instances for various roles (Jenkins, agents, production)
- Security groups
- Application Load Balancer
- Networking components

### Ansible Roles

The Ansible roles configure the provisioned infrastructure:

- **jenkins**: Jenkins server setup with required plugins and configurations
- **docker-agent**: Build agent for Docker image creation and testing
- **ansible-agent**: Deployment agent that pulls and runs the deployment repository

## Setup and Usage

### Prerequisites

- [Terraform](https://www.terraform.io/downloads.html) v1.0.0+
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/index.html) 2.9+
- AWS CLI configured with appropriate credentials
- AWS SSH key pair

### Required Customization

Before deploying the infrastructure, you need to customize several configuration elements:

1. **SSH Key Name**: The current configuration uses a key named "MoniNordic":
   ```terraform
   key_name = "MoniNordic"
   ```
   You must either:
   - Create a key pair with this name in AWS, OR
   - Change the key name in `main.tf` to match your existing key

2. **AWS Region**: Update the region in both files:
   - In `main.tf`:
     ```terraform
     provider "aws" {
       region = "us-west-2"  # Change to your preferred region
     }
     ```
   - In `inventory_aws_ec2.yaml`: Update the regions section to match

3. **Security Groups**: Update the security group IDs in `main.tf`:
   ```terraform
   vpc_security_group_ids = ["sg-02b3d29bdcd49a0cc"]  # Change to your security group ID
   ```

4. **AMI ID**: Update the AMI ID for your region:
   ```terraform
   ami = "ami-05d38da78ce859165"  # Change to appropriate AMI for your region
   ```

### Security-Sensitive Files

For security reasons, these files should be created locally and **NOT committed to the repository**:

1. **SSH private key** (.pem file): For AWS instance access
2. **Docker credentials** (docker_credentials.yml): For Docker Hub access
3. **AWS credentials** (aws_credentials): For AWS API access
4. **Terraform state files** (terraform.tfstate): Contains sensitive infrastructure information

Create these files:

1. **SSH Key**: Ensure your key is available and has the right permissions:
   ```bash
   # Copy your key into the ansible-agent files directory
   cp /path/to/your/YOUR_KEY_NAME.pem roles/ansible-agent/files/
   chmod 400 roles/ansible-agent/files/YOUR_KEY_NAME.pem
   ```

2. **Docker Credentials**: Create a file `roles/jenkins/vars/docker_credentials.yml`:
   ```yaml
   ---
   docker_username: "your-dockerhub-username"
   docker_password: "your-dockerhub-password-or-token"
   ```

3. **AWS Credentials**: Create a credentials file for the Ansible agent:
   ```bash
   # Copy your AWS credentials file
   cp ~/.aws/credentials roles/ansible-agent/files/aws_credentials
   ```

### Deployment Workflow

The infrastructure deployment workflow consists of:

1. **Terraform Deployment**

   ```bash
   # Initialize Terraform
   terraform init

   # Plan the infrastructure
   terraform plan

   # Apply the configuration
   terraform apply
   ```

2. **Ansible Configuration**

   After Terraform has provisioned the infrastructure:
   ```bash
   # Run the Ansible playbook
   ansible-playbook -i inventory_aws_ec2.yaml playbook.yaml
   ```

## Integration with Tests and Deployment

This infrastructure repository works in conjunction with other repositories:

### Test Integration with domain-monitor-tests
1. Tests are maintained in the domain-monitor-tests repository
2. The Selenium test container is built from the test repository code
3. Tests are executed as part of the CI/CD pipeline before deployment
4. Test results determine whether deployment proceeds

### Deployment Integration with domain-monitor-deploy
1. The deployment repository contains Ansible playbooks and Jenkinsfile
2. Jenkins pipelines built by this infrastructure execute those deployment configurations
3. The ansible-agent role is configured to run those deployment steps

### Kubernetes Integration with domain-monitor-k8s
1. The separate Kubernetes repository contains all K8s configurations
2. The CI/CD pipeline can update the Kubernetes manifests with new image tags
3. The ansible-agent can apply Kubernetes changes as needed

## Ansible Agent Architecture

The ansible-agent plays a crucial role in the deployment workflow:

### Ansible Agent Directory Structure

The `roles/ansible-agent/` directory contains:

```
ansible-agent/
├── tasks/
│   └── main.yaml             # Main tasks for setting up the agent
├── templates/
│   ├── service.sh.j2         # Template for the agent service script
│   └── jenkins-agent.service.j2  # Systemd service configuration
└── files/
    ├── aws_credentials       # AWS credentials for dynamic inventory
    ├── MoniNordic.pem        # SSH private key for server access
    └── inventory_aws_ec2.yaml  # AWS EC2 dynamic inventory configuration
```

### Ansible Agent Setup

The Ansible agent is configured with:

1. **Installation**: 
   - Java (for Jenkins agent connectivity)
   - Ansible (for running deployment playbooks)
   - Docker (for container operations)
   - Git and other dependencies

2. **Configuration**: 
   - AWS credentials are copied from `files/aws_credentials` to `/home/ubuntu/.aws/credentials` and `/root/.aws/credentials`
   - SSH key is copied from `files/MoniNordic.pem` to `/etc/ansible/MoniNordic.pem`
   - EC2 inventory is copied from `files/inventory_aws_ec2.yaml` to `/etc/ansible/inventory_aws_ec2.yaml`

3. **Jenkins Agent Connectivity**:
   - Agent JAR is downloaded from the Jenkins master
   - Agent secret is fetched from AWS Secrets Manager
   - A systemd service is created to run the agent and connect to Jenkins

## Docker Hub Credentials Management

The Docker Hub credentials are used to push and pull Docker images during the CI/CD process:

1. **Initial Setup**: In `roles/jenkins/vars/docker_credentials.yml`, provide your Docker Hub credentials:
   ```yaml
   docker_username: "your-dockerhub-username"
   docker_password: "your-dockerhub-password-or-token"
   ```

2. **Credential Usage**: These credentials are automatically added to Jenkins during setup

3. **Pipeline Usage**: In the Jenkinsfile, the credentials are used to authenticate with Docker Hub

4. **Credential Rotation**: To update Docker credentials:
   - Update the `docker_credentials.yml` file
   - Re-run the Ansible playbook to update Jenkins
   - Alternatively, manually update in the Jenkins UI: Manage Jenkins > Manage Credentials

## Troubleshooting

### Common Issues

#### Ansible Connection Problems
- Check that EC2 instances are running
- Verify security groups allow SSH access
- Ensure the inventory file correctly identifies hosts

#### Jenkins Pipeline Failures
- Check Jenkins console output for specific error messages
- Verify Docker credentials are correctly configured
- Ensure EC2 instances have sufficient resources

#### Testing Issues
- Review test logs for specific failure details
- Check if the Selenium container can access the application
- Verify that test dependencies are correctly installed

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -am 'Add my feature'`
4. Push to the branch: `git push origin feature/my-feature`
5. Submit a pull request

## License

[MIT License](LICENSE)