# CloudFormation Template Usage
## Introduction
As part of my submission for SIT706 Task 6.1P - Final Project, three AWS CloudFormation templates have been developed in order to automate the deployment of the provided architecture.

Three stacks were created to automate different parts of the architecture:

- `network-template.yaml`: creates the core networking infrastructure (including subnets and route tables), security groups and a key pair.
- `app-template.yaml`: creates an SSM automation to create an AMI, an EC2 instance from which WordPress is installed, an RDS instance and an S3 bucket.
- `scaling-template.yaml`: creates a launch template, an ASG and a scaling policy for the ASG.

The stacks assume the provided IAM role `LabRole`. This provides the required permissions to deploy and configure all necessary components. In a real-world scenario, several IAM roles may be created to reduce their scope and enhance security, however, this could not be done due to the AWS Learner Lab environment preventing the creation and modification of IAM roles.

## Step 1 - Deploying `218275923-network-template.yaml`
This CloudFormation template, created as `network-stack`, creates 25 resources within the region that the stack is created (in my case, this was `us-west-2`).

These resources include the critical network infrastructure, such as the VPC itself, subnets, route tables, security groups, load balancer, target group and more.

It has 10 unique outputs that are used in the following stacks. These outputs include the subnets, security groups, VPC ID and target group.

## Step 2 - Deploying `218275923-app-template.yaml`
This CloudFormation template, created as `app-stack`, creates the key components required to deploy the initial `ami-server` EC2 instance and to create the AMI of this server.

A Systems Manager Automation is defined within the template to create an AMI based on the provided EC2 instance. The template then defines the `ami-server` EC2 instance, which takes in user data to install the web server's required files and then launches the SSM automation to create the AMI.

Additionally, this template creates a subnet group for the RDS instance, the RDS instance, ensuring that it is Multi-AZ, and the RDS read replica. Finally, it creates an S3 bucket.

As an output, this template provides the public IPv4 address of the `ami-server` EC2 instance.

Note that after creating the `app-stack` **but before creating the `scaling-stack`**, it is advised to do the following, otherwise the auto-scaled instances will not successfully host the WordPress website or connect to the RDS instance:

1. Wait for the `ami-server` instance to report a `2/2 checks passed` status check
2. Connect to the `ami-server` via it's public IPv4 address
3. Follow the WordPress installation on this server
4. Install the `WP Offload Media Lite` plugin, and configure it to connect to the S3 bucket as necessary
5. Connect, via Session Manager, to the `ami-server` instance and modify the `wp_options` table to point to the RDS instance, instead of the `ami-server`:

```mysql
mysql --user=admin --password={password} --host={RDS endpoint}

use wordpress

update wp_options
set option_value = 'http://{load balancer DNS name}'
where option_name = 'siteurl';

update wp_options
set option_value = 'http://{load balancer DNS name}'
where option_name = 'home';
```

6. Run the SSM Automation via the following commands to create the `Server AMI` AMI:

```bash
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

aws ssm start-automation-execution \
--document-name "CreateImageFromInstance" \
--parameters "instanceId=$INSTANCE_ID,imageName=app-image" \
--region us-west-2 \
--output json
```

## Step 3 - Deploying `218275923-scaling-template.yaml`
Once the `app-stack` has been deployed, the WordPress site has been configured and the AMI's status shows `Complete`, the web server is effectively up and running as expected. However, to ensure high-availability, the `scaling-stack` can be deployed using this third and final template.

This template has two input parameters; the name of the network stack (`network-stack`) and the AMI ID of the `Server AMI` created in the previous stage.

With these inputs, the template creates three key components:

* A launch template for the application server noting key configuration items such as the AMI ID, instance size and key-pair used.
* An Auto Scaling Group (ASG) which uses the launch template to maintain a minimum of 1 application server and maximum of 3 within the two private subnets. It uses the previously defined `app-target-group` as a source for connections.
* A Scaling Policy which is applied to the ASG using target tracking scaling to maintain an average CPU utilisation of 70%.

Once created, the scaling group will automatically create one instance which, when connecting to the Load Balancer created in the network stack, will be the server hosting the WordPress website.
