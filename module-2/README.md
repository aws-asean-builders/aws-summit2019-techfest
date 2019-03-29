# Module 2: Creating a Service with AWS Fargate

![Architecture](/images/module-2/architecture-module-2.png)

**Time to complete:** 40 minutes

**Services used:**
* [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
* [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/)
* [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/)
* [Amazon Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
* [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/)
* [AWS Fargate](https://aws.amazon.com/fargate/)
* [AWS Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)


### Overview

In Module 2, you will create a new microservice hosted using [AWS Fargate](https://aws.amazon.com/fargate/) on [Amazon Elastic Container Service](https://aws.amazon.com/ecs/) so that your Mythical Mysfits website can have a application backend to integrate with. AWS Fargate is a deployment option in Amazon ECS that allows you to deploy containers without having to manage any clusters or servers. For our Mythical Mysfits backend, we will use Python and create a Flask app in a Docker container behind a Network Load Balancer. These will form the microservice backend for the frontend website to integrate with.

### Core Infrastructure

You can check your stack you created earlier either via the AWS Console or by running the command:

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsCoreStack
```

**You will be using values from the output of this command throughout the rest of the workshop.  You can run the following command to directly output the above `describe-stacks` command to a new file in your IDE that will be stored as `cloudformation-core-output.json`:**

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsCoreStack > ~/environment/cloudformation-core-output.json
```

## Module 2a: Deploying a Service with AWS Fargate

### Creating a Flask Service Container

#### Building A Docker Image

Next, you will create a docker container image that contains all of the code and configuration required to run the Mythical Mysfits backend as a microservice API created with Flask.  We will build the docker container image within Cloud9 and then push it to the Amazon Elastic Container Registry, where it will be available to pull when we create our service using Fargate.

All of the code required to run our service backend is stored within the `/module-2/app/` directory of the repository you've cloned into your Cloud9 IDE.  If you would like to review the Python code that uses Flask to create the service API, view the `/module-2/app/service/mythicalMysfitsService.py` file.

Docker comes already installed on the Cloud9 IDE that you've created, so in order to build the docker image locally, all we need to do is run the following commands in the Cloud9 terminal:

* Navigate to `~/environment/module-2/app`

```
cd ~/environment/aws-summit2019-techfest/module-2/app
```

* You can get your account ID and default region from the output of the previous CloudFormation **describe-stacks**

* Replace *REPLACE_ME_ACCOUNT_ID* with your account ID and *REPLACE_ME_REGION* with your default region in the following command to build the docker image using the file *Dockerfile*, which contains Docker instructions. The command tags the Docker image, using the `-t` option, with a specific tag format so that the image can later be pushed to the [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) service.

```
docker build . -t REPLACE_ME_ACCOUNT_ID.dkr.ecr.REPLACE_ME_REGION.amazonaws.com/mythicalmysfits/service:latest
```

You will see docker download and install all of the necessary dependency packages that our application needs, and output the tag for the built image.  **Copy the image tag for later reference. Below the example tag shown is: 111111111111.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest**

```
Successfully built 8bxxxxxxxxab
Successfully tagged 111111111111.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest
```

#### Testing the Service Locally

Let's test our image locally within Cloud9 to make sure everything is operating as expected. Copy the image tag that resulted from the previous command and run the following command to deploy the container “locally” (which is actually within your Cloud9 IDE inside AWS!):

```
docker run -p 8080:8080 REPLACE_ME_WITH_DOCKER_IMAGE_TAG
```

As a result you will see docker reporting that your container is up and running locally:

```
 * Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
```

To test our service with a local request, we're going to open up the built-in web browser within the Cloud9 IDE that can be used to preview applications that are running on the IDE instance.  To open the preview web browser, select **Preview > Preview Running Application** in the Cloud9 menu bar:

![preview-menu](/images/module-2/preview-menu.png)

This will open another panel in the IDE where the web browser will be available.  Append /mysfits to the end of the URI in the address bar of the preview browser in the new panel and hit enter:

![preview-menu](/images/module-2/address-bar.png)

If successful you will see a response from the service that returns the JSON document stored at `/aws-summit2019-techfest/module-2/app/service/mysfits-response.json`

When done testing the service you can stop it by pressing CTRL-c on terminal.

#### Pushing the Docker Image to Amazon ECR

With a successful test of our service locally, we're ready to create a container image repository in [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) (Amazon ECR) and push our image into it.  In order to create the registry, run the following command, this creates a new repository in the default AWS ECR registry created for your account.

```
aws ecr create-repository --repository-name mythicalmysfits/service
```

The response to this command will contain additional metadata about the created repository.
In order to push container images into our new repository, we will need to obtain authentication credentials for our Docker client to the repository.  Run the following command, which will return a login command to retrieve credentials for our Docker client and then automatically execute it (include the full command including the $ below). 'Login Succeeded' will be reported if the command is successful.

```
$(aws ecr get-login --no-include-email)
```

Next, push the image you created to the ECR repository using the copied tag from above. Using this command, docker will push your image and all the images it depends on to Amazon ECR:

```
docker push REPLACE_ME_WITH_DOCKER_IMAGE_TAG
```

Run the following command to see your newly pushed docker image stored inside the ECR repository:

```
aws ecr describe-images --repository-name mythicalmysfits/service
```

### Configuring the Service Prerequisites in Amazon ECS

#### Create an ECS Cluster

Now,  we have an image available in ECR that we can deploy to a service hosted on Amazon ECS using AWS Fargate.  The same service you tested locally via the terminal in Cloud9 as part of the last module will now be deployed in the cloud and publicly available behind a Network Load Balancer.  

First, we will create a **Cluster** in the **Amazon Elastic Container Service (ECS)**. This represents the cluster of “servers” that your service containers will be deployed to.  Servers is in "quotations" because you will be using **AWS Fargate**. Fargate allows you to specify that your containers be deployed to a cluster without having to actually provision or manage any servers yourself.

To create a new cluster in ECS, run the following command:

```
aws ecs create-cluster --cluster-name MythicalMysfits-Cluster
```

#### Create an AWS CloudWatch Logs Group

Next, we will create a new log group in **AWS CloudWatch Logs**.  AWS CloudWatch Logs is a service for log collection and analysis. The logs that your container generates will automatically be pushed to AWS CloudWatch logs as part of this specific group. This is especially important when using AWS Fargate since you will not have access to the server infrastructure where your containers are running.

To create the new log group in CloudWatch logs, run the following command:

```
aws logs create-log-group --log-group-name mythicalmysfits-logs
```

#### Register an ECS Task Definition

Now that we have a cluster created and a log group defined for where our container logs will be pushed to, we're ready to register an ECS **task definition**.  A task in ECS is a set of container images that should be scheduled together. A task definition declares that set of containers and the resources and configuration those containers require.  You will use the AWS CLI to create a new task definition for how your new container image should be scheduled to the ECS cluster we just created.  

A JSON file has been provided that will serve as the input to the CLI command.

Open `~/environment/aws-summit2019-techfest/module-2/aws-cli/task-definition.json` in the IDE.

Replace the indicated values with the appropriate ones from your created resources.  

These values will be pulled from the CloudFormation response you copied earlier as well as the docker image tag that you pushed earlier to ECR, eg: `REPLACE_ME_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest`

Once you have replaced the values in `task-defintion.json` and saved it. Execute the following command to register a new task definition in ECS:

```
aws ecs register-task-definition --cli-input-json file://~/environment/aws-summit2019-techfest/module-2/aws-cli/task-definition.json
```

### Enabling a Load Balanced Fargate Service

#### Create a Network Load Balancer

With a new task definition registered, we're ready to provision the infrastructure needed in our service stack. Rather than directly expose our service to the Internet, we will provision a [**Network Load Balancer (NLB)**](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html) to sit in front of our service tier.  This would enable our frontend website code to communicate with a single DNS name while our backend service would be free to elastically scale in-and-out, in multiple Availability Zones, based on demand or if failures occur and new containers need to be provisioned.

To provision a new NLB, execute the following CLI command in the Cloud9 terminal (retrieve the subnetIds from the CloudFormation output you saved):

```
aws elbv2 create-load-balancer --name mysfits-nlb --scheme internet-facing --type network --subnets REPLACE_ME_PUBLIC_SUBNET_ONE REPLACE_ME_PUBLIC_SUBNET_TWO > ~/environment/nlb-output.json
```

When this command has successfully completed, a new file will be created in your IDE called `nlb-output.json`. You will be using the `DNSName`, `VpcId`, and `LoadBalancerArn` in later steps.

#### Create a Load Balancer Target Group

Next, use the CLI to create an NLB **target group**. A target group allows AWS resources to register themselves as targets for requests that the load balancer receives to forward.  Our service containers will automatically register to this target so that they can receive traffic from the NLB when they are provisioned. This command includes one value that will need to be replaced, your `vpc-id` which can be found as a value within the earlier saved `MythicalMysfitsCoreStack` output returned by CloudFormation.

```
aws elbv2 create-target-group --name MythicalMysfits-TargetGroup --port 8080 --protocol TCP --target-type ip --vpc-id REPLACE_ME_VPC_ID --health-check-interval-seconds 10 --health-check-path / --health-check-protocol HTTP --healthy-threshold-count 3 --unhealthy-threshold-count 3 > ~/environment/target-group-output.json
```

When this command completes, its output will be saved to `target-group-output.json` in your IDE. You will reference the `TargetGroupArn` value in a subsequent step.

#### Create a Load Balancer Listener

Next, use the CLI to create a load balancer **listener** for the NLB.  This informs that load balancer that for requests received on a specific port, they should be forwarded to targets that have registered to the above target group. Be sure to replace the two indicated values with the appropriate ARN from the TargetGroup and the NLB that you saved from the previous steps:

```
aws elbv2 create-listener --default-actions TargetGroupArn=REPLACE_ME_NLB_TARGET_GROUP_ARN,Type=forward --load-balancer-arn REPLACE_ME_NLB_ARN --port 80 --protocol TCP
```

### Creating a Service with Fargate

#### Creating a Service Linked Role for ECS

If you have already used ECS in the past you can skip over this step and move on to the next step.  If you have never used ECS before, we need to create an **service linked role** in IAM that grants the ECS service itself permissions to make ECS API requests within your account.  This is required because when you create a service in ECS, the service will call APIs within your account to perform actions like pulling docker images, creating new tasks, etc.

Without creating this role, the ECS service would not be granted permissions to perform the actions required.  To create the role, execute the following command in the terminal:

```
aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com
```

If the above returns an error about the role existing already, you can ignore it, as it would indicate the role has automatically been created in your account in the past.


#### Create the Service

With the NLB created and configured, and the ECS service granted appropriate permissions, we're ready to create the actual ECS **service** where our containers will run and register themselves to the load balancer to receive traffic.  We have included a JSON file for the CLI input that is located at: `~/environment/aws-summit2019-techfest/module-2/aws-cli/service-definition.json`.  This file includes all of the configuration details for the service to be created, including indicating that this service should be launched with **AWS Fargate** - which means that you do not have to provision any servers within the targeted cluster.  The containers that are scheduled as part of the task used in this service will run on top of a cluster that is fully managed by AWS.

Open ```~/environment/aws-summit2019-techfest/module-2/aws-cli/service-definition.json``` in the IDE and replace the indicated values of `REPLACE_ME`. Save it, then run the following command to create the service:

```
aws ecs create-service --cli-input-json file://~/environment/aws-summit2019-techfest/module-2/aws-cli/service-definition.json
```

After your service is created, ECS will provision a new task that's running the container you've pushed to ECR, and register it to the created NLB.  

#### Test the Service

Verify your Fargate service is up and running using AWS console. Navigate to ECS and select `MythicalMysfots-Cluster`. You should see one task running against the MythicalMysfits-Service.

![fargate-service](/images/module-2/fargate-service.png)

Copy the DNS name you saved when creating the NLB (in nlb-output.json) and send a request to it using the preview browser in Cloud9 (or by simply any web browser, since this time our service is available on the Internet). Try sending a request to the mysfits resource:

```
http://mysfits-nlb-xxxxx.elb.REPLACE_ME_REGION.amazonaws.com/mysfits
```

A response showing the same JSON response we received earlier when testing the docker container locally in Cloud9 means your Flask API is up and running on AWS Fargate.

>Note: This Network Load Balancer only supports HTTP (http://) requests since no SSL/TLS certificates are installed on it. For this tutorial, be sure to submit requests using http:// only, https:// requests will not work properly.

### Update Mythical Mysfits to Call the NLB

#### Replace the API Endpoint
Next, we need to integrate our website with your new API backend instead of using the hard coded data that we previously uploaded to S3.  You'll need to update the following file to use the same NLB URL for API calls (do not inlcude the /mysfits path): `/module-2/web/index.html`

Open the file in Cloud9 and replace the highlighted area below between the quotes with the NLB URL:

![before replace](/images/module-2/before-replace.png)

After pasting, the line should look similar to below:

![after replace](/images/module-2/after-replace.png)

#### Upload to S3
To upload this file to your S3 hosted website, use the bucket name again that was created during Module 1, and run the following command:

```
aws s3 cp ~/environment/aws-summit2019-techfest/module-2/web/index.html s3://REPLACE_ME_BUCKET_NAME/index.html
```

Open your website using the same CloudFront URL used at the end of Module 1 in order to see your new Mythical Mysfits website, which is retrieving JSON data from your Flask API running within a docker container deployed to AWS Fargate!

**Note: If the page does not load completely, your browser could be preventing the loading of mixed content. This is because CloudFront connection is over https while NLB connection is over http. You can configure your browser to ignore this for now. We will sort this by using https end to end in Module 4.**


## Module 2b: Automating Deployments using AWS Code Services

![Architecture](/images/module-2/architecture-module-2b.png)


### CI/CD Pipeline
We have created the CI/CD pipeline as part of the core CloudFormation stack. Here is an overview of the different components of the pipeline.

#### S3 Bucket for Pipeline Artifacts

Now that you have a service up and running, you may think of code changes that you'd like to make to your Flask service.  It would be a bottleneck for your development speed if you had to go through all of the same steps above every time you wanted to deploy a new feature to your service. That's where Continuous Integration and Continuous Delivery or CI/CD come in!

First, we have already created a S3 bucket that will be used to store the temporary artifacts that are created in the middle of our CI/CD pipeline executions - mythicalmysfitscorestack-artifactbucket-xxxxxx

Next, this bucket needs a bucket policy to define permissions for the data stored within it. But unlike our website bucket that allowed access to anyone, only our CI/CD pipeline should have access to this bucket. On the AWS S3 console, select the bucket and then choose "Permissions". Have a look at the bucket policy

#### CodeCommit Repository

You'll need a place to push and store your code in. We use [**AWS CodeCommit Repository**](https://aws.amazon.com/codecommit/) for this purpose.

#### CodeBuild Project

With a repository to store our code in, and an S3 bucket that will be used for our CI/CD artifacts, the missing piece is a way for a service build to occur.  This is accomplished by creating an [**AWS CodeBuild Project**](https://aws.amazon.com/codebuild/).  Any time a build execution is triggered, AWS CodeBuild will automatically provision a build server to our configuration and execute the steps required to build our docker image and push a new version of it to the ECR repository we created (and then spin the server down when the build is completed).  The steps for our build (which package our Python code and build/push the Docker container) are included in the `~/environment/aws-summit2019-techfest/module-2/app/buildspec.yml` file.  The **buildspec.yml** file is what you create to instruct CodeBuild what steps are required for a build execution within a CodeBuild project.

#### CodePipeline Pipeline

Finally, we need a way to *continuously integrate* our CodeCommit repository with our CodeBuild project so that builds will automatically occur whenever a code change is pushed to the repository.  Then, we need a way to *continuously deliver* those newly built artifacts to our service in ECS.  [**AWS CodePipeline**](https://aws.amazon.com/codepipeline/) is the service that glues these actions together in a **pipeline**.  

Anytime a code change is pushed into your CodeCommit repository, CodePipeline will deliver the latest code to your AWS CodeBuild project so that a build will occur. When successfully built by CodeBuild, CodePipeline will perform a deployment to ECS using the latest container image that the CodeBuild execution pushed into ECR.

Navigate to the CodePipeline console and spend a few minutes to explore it. You may see the source stage has failed. You can ignore this as our repository is empty now.

#### Enable Automated Access to ECR Image Repository

We have one final step before our CI/CD pipeline can execute end-to-end successfully. With a CI/CD pipeline in place, you won't be manually pushing container images into ECR anymore.  CodeBuild will be pushing new images now. We need to give CodeBuild permission to perform actions on your image repository with an **ECR repository policy**.  The policy document needs to be updated with the specific ARN for the CodeBuild role created by the MythicalMysfitsCoreStack, and the policy document is located at `~/environment/aws-summit2019-techfest/module-2/aws-cli/ecr-policy.json`.  You will find the role arn in cloudformation-core-output.json. Update and save this file and then run the following command to create the policy:

```
aws ecr set-repository-policy --repository-name mythicalmysfits/service --policy-text file://~/environment/aws-summit2019-techfest/module-2/aws-cli/ecr-policy.json
```

When that has been created successfully, you have a working end-to-end CI/CD pipeline to deliver code changes automatically to your service in ECS.

### Test the CI/CD Pipeline

#### Using Git with AWS CodeCommit
To test out the new pipeline, we need to configure git within your Cloud9 IDE and integrate it with your CodeCommit repository.

AWS CodeCommit provides a credential helper for git that we will use to make integration easy.  Run the following commands in sequence the terminal to configure git to be used with AWS CodeCommit (neither will report any response if successful):

```
git config --global user.name "REPLACE_ME_WITH_YOUR_NAME"
```

```
git config --global user.email REPLACE_ME_WITH_YOUR_EMAIL@example.com
```

```
git config --global credential.helper '!aws codecommit credential-helper $@'
```

```
git config --global credential.UseHttpPath true
```

Next change directories in your IDE to the environment directory using the terminal:

```
cd ~/environment/
```

Now, we are ready to clone our repository using the following terminal command:

```
git clone https://git-codecommit.REPLACE_ME_REGION.amazonaws.com/v1/repos/MythicalMysfitsService-Repository
```

This will tell us that our repository is empty!  Let's fix that by copying the application files into our repository directory using the following command:

```
cp -r ~/environment/aws-summit2019-techfest/module-2/app/* ~/environment/MythicalMysfitsService-Repository/
```

#### Pushing a Code Change

Now the completed service code that we used to create our Fargate service in Module 2 is stored in the local repository that we just cloned from AWS CodeCommit.  Let's make a change to the Flask service before committing our changes, to demonstrate that the CI/CD pipeline we've created is working. In Cloud9, open the file stored at `~/environment/MythicalMysfitsService-Repository/service/mysfits-response.json` and change the age of one of the mysfits to another value and save the file.

After saving the file, change directories to the new repository directory:

```
cd ~/environment/MythicalMysfitsService-Repository/
```

Then, run the following git commands to push in your code changes.  

```
git add .
git commit -m "I changed the age of one of the mysfits."
git push
```

After the change is pushed into the repository, you can open the CodePipeline service in the AWS Console to view your changes as they progress through the CI/CD pipeline. After committing your code change, it will take about 5 to 10 minutes for the changes to be deployed to your live service running in Fargate.  During this time, AWS CodePipeline will orchestrate triggering a pipeline execution when the changes have been checked into your CodeCommit repository, trigger your CodeBuild project to initiate a new build, and retrieve the docker image that was pushed to ECR by CodeBuild and perform an automated ECS [Update Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/update-service.html) action to connection drain the existing containers that are running in your service and replace them with the newly built image.  Refresh your Mythical Mysfits website in the browser to see that the changes have taken effect.

You can view the progress of your code change through the CodePipeline console here (no actions needed, just watch the automation in action!):
[AWS CodePipeline](https://console.aws.amazon.com/codepipeline/home)

This concludes Module 2.

[Proceed to Module 3](/module-3)


## [AWS Developer Center](https://developer.aws)
