# Domain Monitoring System - Infrastructure Repository

This repository contains the infrastructure automation code for deploying and managing the Domain Monitoring System. The infrastructure is provisioned on AWS and configured using Ansible, with Jenkins handling the CI/CD pipeline.

## Architecture Overview

The infrastructure consists of several AWS EC2 instances with different roles:

1. **Jenkins Master** - Orchestrates the CI/CD pipeline
2. **Docker Agent** - Builds and tests Docker images
3. **Ansible Agent** - Handles deployment to production servers
4. **Production Servers** - Hosts the application

## Repository Structure

```
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
├── inventory_aws_ec2.yaml
├── playbook.yaml
├── main.tf
└── README.md
```

## Deployment Flow

The infrastructure is provisioned and configured in the following sequence:

1. Terraform (`main.tf`) creates EC2 instances with appropriate tags
2. Ansible dynamic inventory (`inventory_aws_ec2.yaml`) discovers instances based on tags
3. Ansible playbook (`playbook.yaml`) configures the instances according to their roles
4. Jenkins pipelines are automatically created during Jenkins master configuration

## Terraform Resources

The Terraform configuration (`main.tf`) provisions:

- 1 Jenkins master instance
- 1 Docker agent instance
- 1 Ansible agent instance
- 2 Production instances
- 1 Application Load Balancer with appropriate target groups

Each instance is tagged with a `Purpose` tag that Ansible uses to identify its role.

## Ansible Dynamic Inventory

The dynamic inventory (`inventory_aws_ec2.yaml`) uses the AWS EC2 plugin to discover instances based on their tags:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-west-2
filters:
  tag:Name: raziel_*
  instance-state-name: running
keyed_groups:
  - key: tags.Purpose
    prefix: tag_Purpose
```

This creates dynamic host groups like `tag_Purpose_jenkins`, `tag_Purpose_docker_agent`, etc.

## Ansible Playbook

The main playbook (`playbook.yaml`) configures each group of servers:

1. Sets private IPs as facts for all hosts
2. Configures the Jenkins master
3. Configures the Docker agent
4. Configures the Ansible agent

### Jenkins Master Configuration

The Jenkins master is set up with Docker to run Jenkins in a container. Key configuration steps:

1. Install Docker and prerequisites
2. Create Jenkins home directory
3. Create init scripts and plugin configuration
4. Run Jenkins container
5. Create agent nodes
6. Store agent secrets in AWS Secrets Manager
7. Create pipelines using Groovy scripts
8. Add Docker Hub credentials

### Pipeline Creation Process

The Jenkins pipelines are created automatically using Groovy scripts during configuration:

1. `docker-pipeline-creation.groovy.j2` creates the Docker build pipeline
2. `ansible-pipeline-creation.groovy.j2` creates the Ansible deployment pipeline

These scripts:
- Define the pipeline job
- Configure Git SCM to fetch the repository
- Point to the Jenkinsfile path
- Assign the pipeline to run on the appropriate agent
- Trigger an initial build

### Agent Configuration

Both Docker and Ansible agents are configured to connect to the Jenkins master using JNLP. The agent setup includes:

1. Installing required software (Java, Docker, Ansible)
2. Creating the agent workspace
3. Downloading the agent JAR from the Jenkins master
4. Fetching agent secrets from AWS Secrets Manager
5. Creating and starting the agent service

## Application Deployment

The Domain Monitoring System deployment follows this flow:

1. Developer pushes code to Git repository
2. Jenkins Docker pipeline:
   - Clones the repository
   - Builds Docker images for frontend and backend
   - Runs tests
   - Pushes images to Docker Hub
3. Jenkins Ansible pipeline:
   - Deploys the application to production servers
   - Configures the load balancer

## AWS Integration

The infrastructure makes use of AWS services:

- **EC2** for compute instances
- **ELB** for load balancing
- **Secrets Manager** for storing Jenkins agent secrets
- **IAM** for permissions

## Setup Instructions

### Prerequisites

- AWS CLI configured with appropriate permissions
- Terraform installed
- Ansible installed with required collections:
  ```
  ansible-galaxy collection install amazon.aws
  ```
- SSH key pair for AWS EC2 instance access
- Docker Hub account (for storing container images)

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
   This key is used by the Ansible agent to SSH into the production servers during deployment.

2. **Docker Credentials**: Create a file `roles/jenkins/vars/docker_credentials.yml`:
   ```yaml
   ---
   docker_username: "your-dockerhub-username"
   docker_password: "your-dockerhub-password-or-token"
   ```
   This file is used by the Jenkins role to configure Docker Hub credentials in Jenkins.

3. **AWS Credentials**: Create a credentials file for the Ansible agent:
   ```bash
   # Copy your AWS credentials file
   cp ~/.aws/credentials roles/ansible-agent/files/aws_credentials
   ```
   The file should have this format:
   ```ini
   [default]
   aws_access_key_id = YOUR_ACCESS_KEY
   aws_secret_access_key = YOUR_SECRET_KEY
   ```
   These credentials are used by the Ansible agent to access the AWS EC2 dynamic inventory.

4. **AWS EC2 Inventory**: The `inventory_aws_ec2.yaml` file in the `roles/ansible-agent/files/` directory is used by the Ansible agent to discover EC2 instances. This file is copied to the Ansible agent during setup.

### Deployment Steps

1. **Initialize Terraform:**
   ```
   terraform init
   ```

2. **Create infrastructure with Terraform:**
   ```
   terraform apply
   ```

3. **Run Ansible playbook:**
   ```
   ansible-playbook -i inventory_aws_ec2.yaml playbook.yaml
   ```

4. **Access Jenkins:**
   
   After deployment, access Jenkins at:
   ```
   http://<jenkins-master-public-ip>:8080
   ```
   
   Login with:
   - Username: admin
   - Password: admin

5. **Access the application:**

   The application is accessible through the load balancer:
   ```
   http://<load-balancer-dns>
   ```

## Ansible Agent Configuration

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

### Deployment Workflow

The deployment process works as follows:

1. The Ansible agent connects to Jenkins as a node
2. When a deployment pipeline is triggered in Jenkins, it:
   - Runs on the Ansible agent (via `agent { label 'ansible' }` in Jenkinsfile)
   - Checks out the deployment repository code
   - Executes the deployment playbook against production servers

3. The deployment playbook:
   - Pulls the Docker images for frontend and backend
   - Runs containers with proper configuration
   - Verifies application health

## Maintenance

### Adding New Agents

To add a new agent:

1. Add a new EC2 instance with appropriate tags in Terraform
2. Update the Ansible playbook to include configuration for the new agent
3. Create a new agent in Jenkins

### Docker Hub Credentials Management

The Docker Hub credentials are used to push and pull Docker images during the CI/CD process:

1. **Initial Setup**: In `roles/jenkins/vars/docker_credentials.yml`, provide your Docker Hub credentials:
   ```yaml
   docker_username: "your-dockerhub-username"
   docker_password: "your-dockerhub-password-or-token"
   ```

2. **Credential Usage**: These credentials are automatically added to Jenkins during setup using this code in `roles/jenkins/tasks/main.yaml`:
   ```yaml
   - name: Add Docker credentials in Jenkins
     jenkins_script:
       script: |
         def dockerCreds = new com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl(
           com.cloudbees.plugins.credentials.CredentialsScope.GLOBAL,
           'docker',
           'Docker Hub Credentials',
           '{{ docker_username }}',
           '{{ docker_password }}'
         )
         # ... additional code to store credentials ...
   ```

3. **Pipeline Usage**: In the Jenkinsfile, the credentials are used like this:
   ```groovy
   withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
       sh """
           echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin
           docker push ${IMAGE_NAME}:${TAG}
       """
   }
   ```

4. **Credential Rotation**: To update Docker credentials:
   - Update the `docker_credentials.yml` file
   - Re-run the Ansible playbook to update Jenkins
   - Alternatively, manually update in the Jenkins UI: Manage Jenkins > Manage Credentials

### Updating Pipelines

To update the CI/CD pipelines:

1. Modify the Groovy template files in the Jenkins templates directory
2. Re-run the Ansible playbook to apply changes

## Troubleshooting

### Common Issues

1. **Jenkins agent connection issues:**
   - Verify agent secrets in AWS Secrets Manager
   - Check agent service status: `systemctl status jenkins-agent`
   - Verify network connectivity between master and agent

2. **Pipeline failures:**
   - Check Jenkins logs: `docker logs jenkins`
   - Verify Docker credentials are properly configured
   - Ensure Git repository is accessible

3. **Load Balancer issues:**
   - Check target group health
   - Verify security group settings
   - Confirm application health checks are passing

### Logs

- Jenkins logs: `docker logs jenkins`
- Agent logs: `journalctl -u jenkins-agent`
- Application logs: Check Docker container logs on production servers