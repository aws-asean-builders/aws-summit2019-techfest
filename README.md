# Build a Modern Application on AWS (Python)

![mysfits-welcome](/images/module-1/mysfits-welcome.png)

### Welcome to the **Python** version of the Build a Modern Application on AWS Workshop!

**AWS Experience: Beginner**

**Time to Complete: 3-4 hours**

**Cost to Complete: Many of the services used are included in the AWS Free Tier. For those that are not, the sample application will cost, in total, less than $1/day.**

**Tutorial Prereqs:**

* **An AWS Account and Administrator-level access to it**

Please be sure to terminate all of the resources created during this workshop to ensure that you are no longer charged.

**Note:**  Estimated workshop costs assume little to no traffic will be served by your demo website created as part of this workshop.

### Application Architecture

![Application Architecture](/images/arch-diagram.png)

The Mythical Mysfits website serves it's static content from Amazon S3 with Amazon CloudFront, provides a microservice API backend deployed as a container through AWS Fargate on Amazon ECS, stores data in a managed NoSQL database provided by Amazon DynamoDB, with authentication and authorization for the application enabled through AWS API Gateway and it's integration with Amazon Cognito.

You will be creating and deploying changes to this application completely programmatically. You will use the AWS Command Line Interface to execute commands that create the required infrastructure components, which includes a fully managed CI/CD stack utilizing AWS CodeCommit, CodeBuild, and CodePipeline.  Finally, you will complete the development tasks required all within your own browser by leveraging the cloud-based IDE, AWS Cloud9.

### Core Infrastructure

Before we can create our service, we need to create the core infrastructure environment that the service will use, including the networking infrastructure in [Amazon VPC](https://aws.amazon.com/vpc/), and the [AWS Identity and Access Management](https://aws.amazon.com/iam/) Roles that will define the permissions that ECS and our containers will have on top of AWS.  We have used [AWS CloudFormation](https://aws.amazon.com/cloudformation/) to accomplish this. AWS CloudFormation is a service that can programmatically provision AWS resources that you declare within JSON or YAML files called *CloudFormation Templates*, enabling the common best practice of *Infrastructure as Code*.  

You can find the CloudFormation template to create all of the necessary Network and Security resources [here](module-2/cfn/core.yml). Save this template locally as core.yml. This template will set up the following resources:

* [**An Amazon VPC**](https://aws.amazon.com/vpc/) - a network environment that contains four subnets (two public and two private) in the 10.0.0.0/16 private IP space, as well as all the needed Route Table configurations.  The subnets for this network are created in separate AWS Availability Zones (AZ) to enable high availability across multiple physical facilities in an AWS Region. Learn more about how AZs can help you achieve High Availability [here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html).
* [**Two NAT Gateways**](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) (one for each public subnet, also spanning multiple AZs) - allow the containers we will eventually deploy into our private subnets to communicate out to the Internet to download necessary packages, etc.
* [**A DynamoDB VPC Endpoint**](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/vpc-endpoints-dynamodb.html) - our microservice backend will eventually integrate with [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) for persistence (as part of module 3).
* [**A Security Group**](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) - Allows your docker containers to receive traffic on port 8080 from the Internet through the Network Load Balancer.
* [**IAM Roles**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) - Identity and Access Management Roles are created. These will be used throughout the workshop to give AWS services or resources you create access to other AWS services like DynamoDB, S3, and more.

This web application can be deployed in any AWS region that supports all the services used in this application. For this workshop, select one of the regions below from the dropdown in the upper right corner of the AWS Management Console.

* us-east-1 (N. Virginia)
* us-west-2 (Oregon)

Navigate to CloudFormation on your AWS console and click on "Create stack". Select "Upload a template file" and choose the template you saved.
![cfn-upload-template](/images/cfn-upload-template.png)

Click "Next" and specify stack name as MythicalMysfitsCoreStack. Click "Next" and leave stack options unchanged. Ont the next screen, acknowledge the creation of IAM respurces and click "Create stack".
![cfn-create-stack](/images/cfn-create-stack.png)

## Begin the Modern Application Workshop

### [Proceed to Module 1](/module-1)

## [AWS Developer Center](https://developer.aws)
