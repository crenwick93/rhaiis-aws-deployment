# Red Hat AI Inference Server (RHAIIS) Deployment Playbooks

This repository contains Ansible playbooks for provisioning AWS infrastructure and deploying Red Hat AI Inference Server (RHAIIS) on Red Hat Enterprise Linux.

## Overview

These playbooks enable you to:
1. Provision an AWS EC2 instance with RHEL 9.5
2. Install NVIDIA drivers and CUDA
3. Deploy RHAIIS with your chosen foundation model
4. Tear down the infrastructure when no longer needed

## Prerequisites

- Ansible Core 2.15 or higher
- AWS account with appropriate permissions
- AWS CLI configured with access credentials
- Python 3.9 or higher with required packages

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/yourusername/rhaiis-playbooks.git
   cd rhaiis-playbooks
   ```

2. Install required Ansible collections:
   ```bash
   ansible-galaxy collection install -r collections/requirements.yml
   ```

## Configuration

1. Copy the sample variables file and customize it for your environment:
   ```bash
   cp sample_vars.yml vars.yml
   ```

2. Edit `vars.yml` to set your specific configuration:
   - AWS instance details (type, region, AMI)
   - Instance name and tags
   - SSH key name
   - Root volume size
   - Hugging Face token
   - RHAIIS model selection

## Usage

### Provisioning AWS Infrastructure

To provision the AWS infrastructure:

```bash
ansible-playbook provision_aws.yml -e @vars.yml
```

This will:
1. Create a VPC, subnet, security group, and internet gateway
2. Launch an EC2 instance with the specified instance type
3. Configure a 500GB root volume (or your specified size)
4. Create an instance-specific inventory file in the `instances/<instance_name>/` directory
5. Create a symlink to the current instance's inventory at `inventory.ini`

### Deploying RHAIIS

To install and configure RHAIIS on the provisioned instance:

```bash
ansible-playbook rhaiis.yml -i inventory.ini -e @vars.yml
```

This will:
1. Set up NVIDIA drivers and CUDA
2. Install RHAIIS components
3. Deploy the specified model (default: RedHatAI/Llama-3.1-8B-Instruct)
4. Configure the service for production use

### Testing the Deployed Model

After deploying the model, you can test it using curl commands. The API requires an authentication token which is set by the `setup_rhaiis_on_rhel_api_token` variable in your vars.yml file.

#### Check if the Model Server is Running
```bash
# Replace <INSTANCE_IP> with your EC2 instance's public IP address
curl -s http://<INSTANCE_IP>:8000/v1/models \
  -H "Authorization: Bearer testtoken" | jq
```

#### Test Text Generation
```bash
curl -X POST http://<INSTANCE_IP>:8000/v1/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer testtoken" \
  -d '{
    "model": "RedHatAI/Llama-3.1-8B-Instruct",
    "prompt": "Write a short poem about Linux",
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

#### Test Chat Completion
```bash
curl -X POST http://<INSTANCE_IP>:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer testtoken" \
  -d '{
    "model": "RedHatAI/Llama-3.1-8B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What are the key features of RHEL 9?"}
    ],
    "temperature": 0.7,
    "max_tokens": 150
  }'
```

### Tearing Down Infrastructure

To remove all AWS resources when you're done:

```bash
ansible-playbook teardown_aws.yml -e @vars.yml
```

This will:
1. Terminate the EC2 instance
2. Delete associated resources (security group, subnet, internet gateway, VPC)
3. Remove the key pair from AWS
4. Delete the local PEM file
5. Remove the instance-specific directory and inventory symlink

## Key Variables

| Variable | Description | Default Value |
|----------|-------------|---------------|
| instance_name | Name for your EC2 instance | your-rhaiis-instance |
| region | AWS region | us-east-1 |
| ami_id | AMI ID for RHEL 9.5 | ami-0673a5faaca9a47cb |
| instance_type | AWS instance type | g5.2xlarge |
| key_name | SSH key name | your-key-name |
| vpc_cidr | VPC CIDR block | 10.0.0.0/16 |
| subnet_cidr | Subnet CIDR block | 10.0.1.0/24 |
| volume_size | Root volume size in GB | 500 |
| instance_tags | Dictionary of tags to apply to the EC2 instance | See below |
| setup_rhaiis_on_rhel_hf_token | HuggingFace token | your-huggingface-token |
| setup_rhaiis_on_rhel_vllm_model | Hugging Face model to deploy | RedHatAI/Llama-3.1-8B-Instruct |
| setup_rhaiis_on_rhel_api_token | API token for accessing deployed model | testtoken |

### Instance Tagging

You can add custom tags to your EC2 instance by defining the `instance_tags` variable in your `vars.yml` file:

```yaml
# Instance tags
instance_tags:
  Name: "your-rhaiis-instance"  # Will be overridden by instance_name
  owner: "your-username"        # Owner tag
  Environment: "Development"    # Environment tag
  Project: "RHAIIS"             # Project tag
```

The `Name` tag will always be set to the value of `instance_name`, but you can add any additional tags you need.

## Multi-Instance Management

The playbooks support managing multiple instances by:
1. Creating instance-specific directories under `instances/<instance_name>/`
2. Storing each instance's inventory file separately
3. Creating a symlink from `inventory.ini` to the current instance's inventory

This allows you to switch between instances by simply updating the symlink.

## Repository Structure

- `provision_aws.yml` - Playbook for AWS infrastructure provisioning
- `rhaiis.yml` - Playbook for RHAIIS installation and configuration
- `teardown_aws.yml` - Playbook for infrastructure cleanup
- `inventory.ini.j2` - Template for Ansible inventory
- `instances/` - Directory for instance-specific inventory files
- `roles/` - Ansible roles for specific tasks
  - `setup_nvidia_cuda/` - Role for NVIDIA driver and CUDA setup
  - `setup_rhaiis_on_rhel/` - Role for RHAIIS installation
- `collections/` - Ansible collection requirements

## Notes

- The g5.2xlarge instance type has 24GB of GPU memory, which is sufficient for running the Llama-3.1-8B model.
- Make sure your AWS credentials are properly configured before running the playbooks.
- The inventory.ini file is automatically generated during provisioning with the correct SSH key and host settings.
- If you already have a PEM file matching your key_name, the playbook will use it instead of creating a new key pair.

## Troubleshooting

- If SSH connection fails, verify that your AWS security group allows SSH access.

- For model deployment issues:
  - Check the RHAIIS logs on the instance: `sudo cat /tmp/rhaiis.log`
  - Common issues include:
    - GPU memory errors: Check if your instance has enough GPU memory for the model
    - Authentication failures: Verify your Hugging Face token is valid
    - Model download issues: Ensure network connectivity to Hugging Face
    - If model download appears stuck (no progress in logs), try:
      ```bash
      sudo systemctl stop rhaiis.service
      sudo systemctl start rhaiis.service
      # Monitor the logs to ensure download resumes
      sudo tail -f /tmp/rhaiis.log
      ```
  - You can restart the RHAIIS service with: `sudo systemctl restart rhaiis.service`
  - View real-time logs with: `sudo tail -f /tmp/rhaiis.log`

- If the playbook fails during provisioning, you may need to manually run the teardown playbook to clean up resources.
