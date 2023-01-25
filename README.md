# Fargate-ECSCluster-Cloudformation
AWS Cloudformation Managed Complete ECS Infrastructure Including CI/CD Pipeline From Github to ECS


Github to AWS CodePipeline Trigger
Instead of manually creating an AWS CodePipeline, an ECS Fargate Cluster infrastructure and configuring a CI/CD Pipeline with Github, Let’s configure a Cloudformation template to automate all of that for us.

AWS Cloudformation is used to manage AWS infrastructure as code. It is recommended to develop your application using Cloudformation templates due to following advantages,

Enables version controlling for the infrastructure itself.
Seamlessly deploy multiple stages (Dev, QA and Production) creating replicas of the infrastructure.
Greatly simplifies the deployment.
Can be easily integrated with CI/CD pipelines.
No need to manage configurations for each AWS service separately. It can all be included in the Cloudformation template itself.
There are many frameworks that are built on top of Cloudformation. Serverless Framework is one of them specializing in developing Lambda based serverless applications. Please refer to this blog post where I go into full detail about it.

Prerequisites
You need to have an AWS account and some basic knowledge working with AWS services. Following AWS services will be utilised throughout this guide.

ECS Service
Cloudformation
CodeBuild
CodeDeploy
CodePipeline
VPC
ECR
If you’re not familiar with these services, please refer to above links before proceeding forward.

You’ll also need a Github Account.

You will learn
How to leverage AWS Cloudformation to set up a complete AWS Infrastructure with a CI/CD pipeline for your application.

The Demo Application
This blog will demonstrate how to deploy a complete Fargate ECS cluster and a CI/CD Pipeline with following components,

AWS CodePipeline
AWS ECR
AWS VPC with 2 Subnets
Application Load Balancer
Internet Gateway
Fargate ECS Cluster
The sample container application is a NodeJS hello world app.

You can find the complete source code in this Github Repo.

Main Challenge
It’s not possible to create all of these AWS Resources using a single Cloudformation template simply because in order to create an ECS Cluster, the Docker image needs to be in the ECR.

Running all of these creations in a single Cloudformation template would result in a failure during the ECS cluster creation simply because there would be no Docker image available in the ECR as the ECR itself is also being created by the same Cloudformation template.

Solution
The solution is to deploy only the CI/CD Pipeline using a Cloudformation template and once it succeeds on creating the Docker image, deploy the next Cloudformation template containing the Fargate ECS Cluster infrastructure using this pipeline itself.

Cloudformation Multi-Stage Compatibility
In order to make the Cloudformation template compatible for multiple stages (Dev, QA or any other), all resources defined in the template will have their name combined with stage name and AWS account Id appended to make sure the uniqueness of the resource.

For example a resource name would consist of following,

Stage name + AWS Account Id + Resource name

AWS CodePipeline
AWS CodePipeline will be used to build and deploy the application along with the AWS infrastructure needed.

AWS CodePipeline will be created using a Cloudformation template and this will be done by the user manually in the console. This will trigger the CI/CD Pipeline deployment and then later the deployment of Cloudformation template consisting of the Fargate ECS Cluster.

There are 3 stages in AWS CodePipeline.

1. Source Stage
Source provider will be Github in this case. This stage authenticates with Github via a user provided Github access token and pulls the source code from the Github repository.

2. Build Stage
Build provider will be AWS CodeBuild in this case. it will be used for following tasks,

Build the Docker image.
Push the Docker image to the ECR.
Output Docker ECR image URL to a JSON file saved in a S3 bucket so the deploy stage can use it.
You can find more details about AWS CodeBuild from this link.

3. Deploy Stage
Deploy provider will be Cloudformation in this case. This stage will receive the source code pulled from Github as an input artifact which contains the Cloudformation template it should deploy. This template contains the complete Fargate ECS Cluster with a VPC, 2 Subnets and a Load balancer.

Parameters that are required (ECR Image URL from the build stage) for this Cloudformation template will also get passed into the template at this stage.

You can find more details about AWS CodePipelin from this link.

Please refer to the application architecture below for a better understanding.

Application Architecture

AWS Application Architecture
The main steps of the deployment process can be identified as below,

User manually deploys the initial Cloudformation template containing the CodePipeline infrastructure (CodePipeline.yml).
Cloudformation starts the deployment of the AWS CodePipeline infrastructure.
After the Cloudformation deployment, CodePipeline pulls the source code from Github in the source stage.
CodePipeline builds the docker image and pushes to the ECR in the build stage using AWS CodeBuild service.
CodePipeline starts the deployment of the Cloudformation template (Fargate-Cluster.yml) containing Fargate ECS Cluster in the deploy stage.
To summarise, there will be 2 Cloudformation template deployments. The first one will be done by the user which will create an AWS CodePipeline and then the second one will be done by the AWS CodePipeline itself.

AWS CodePipeline Infrastructure in Cloudformation (CodePipeline.yml)
Let’s see how to create each resource in Cloudformation syntax that is needed for the complete CI/CD infrastructure. In summary following resources will be created.

ECR Repository
S3 Bucket
IAM Role for CodePipeline Execution
IAM Role for CodeBuild Execution
IAM Role for Cloudformation Execution
AWS BuildProject
AWS CodePipeline
ECR Repository

ECR Repository Cloudformation
Here the RepositoryName is defined in the template using Join Function. Using this it’s possible to define names dynamically concatenating stage name, account id and resource name.

S3 Bucket

S3 Bucket Cloudformation
This is defined similarly to ECR.

AWS CodeBuild

AWS CodeBuild Cloudformation
This is the AWS CodeBuild service AWS CodePipeline will use to build the project.

Artifacts -> Type is set to CODEPIPELINE to let the AWS CodePipeline manage build artifacts.

Environment contains the configurations for the build environment.

Please note that ECR location to push the build image is passed via EnvironmentVariables property as ECR_REPOSITORY_URI.

ServiceRole property provides an AWS Role with permissions for AWS CodeBuild service to manipulate AWS resources. This is also defined in the same Cloudformation template as below and referenced in the ServiceRole property in AWS CodeBuild template.


AWS CodeBuildExecutionRole Cloudformation
Source -> Type is also set to CODEPIPELINE in order to let the AWS CodePipeline service to manage artifacts.

Source -> BuildSpec is set to value buildspec.yml. This file resides in the Github repository with commands to build and push the docker image to the ECR as displayed below.


buildspec.yml File
AWS CodeBuild service will execute these instructions within the specified build environment.

Note that after Docker image is built and pushed to ECR, artifacts -> files specifies an output artifact file called imageDetail.json. This file gets created by the following command in the buildspec.yml file.

printf ‘{“ImageURI”:”%s:%s”}’ $ECR_REPOSITORY_URI $IMAGE_TAG > imageDetail.json

This command outputs the ECR image repository url to the imageDetail.json file in order for the Cloudformation to use it during the ECS Cluster deployment.

AWS CodePipeline
AWS CodePipeline is the CI/CD pipeline that manages pulling the source code, executing AWS CodeBuild and then deploying the infrastructure needed.


AWS CodePipeline Cloudformation
ArtifactStore -> Location specifies the S3 bucket name defined in the Cloudformation template as the artifact location. CodePipeline will use that S3 bucket to store the downloaded source code from Github and build artifacts from CodeBuild service.

RoleArn property specifies the AWS role that gives permission to AWS CodePipeline service to manipulate AWS resources. This is also defined in the Cloudformation template itself as displayed below.


AWS CodePipeline ExecutionRole
Configuring Stages in AWS CodePipeline
1. Source Stage Cloudformation


AWS CodePipeline Source Stage Cloudformation
Provider: GitHub specifies the provider as Github and Configuration section defines Repo, Branch, Owner and OAuthToken. These values are specified as parameter values for the Cloudformation template which you can see below. You can specify values to these parameters when deploying the template.


Owner of the Github repo needs to generate a token and specify here in order for AWS CodeBuild service to have permission to pull the source code. Refer to this Github link on how to do that.

OutputArtifacts property specifies the path for the source code to be downloaded in the S3 bucket that was configured in ArtifactStore -> Location.

2. Build Stage Cloudformation


AWS CodePipeline Build Stage Cloudformation
Provider: CodeBuild specifies this stage will be using the AWS CodeBuild that was configured earlier in “AWS CodeBuild” section. This stage takes the output artifacts that was specified as OutputArtifacts in source stage as an input in InputArtifacts property.

During the execution of this stage, AWS CodePipeline will be running the AWS CodeBuild. Build commands specified in the buildspec.yml file will be executed by AWS CodeBuild to build, push the Docker image to ECR and finally write the ECR image URL to imageDetail.json and output it to S3 bucket as a Build Artifact.

3. Deploy Stage Cloudformation


AWS CodePipeline Deploy Stage Cloudformation
This is where the complete infrastructure for the Fargate ECS Cluster will be deployed. Including VPC, Subnets, Load Balancers, Internet Gateway, Routing Tables etc.

Provider: CloudFormation specifies that this deployment is done via Cloudformation.

InputArtifacts are provided from both Source and Build stage.

Source stage provides the source-output-artifacts which contains the Cloudformation template (Fargate-Cluster.yml) to be deployed.

Build stage provides the build-output-artifacts which contains the ECR Image URI inside the imageDetail.json file.

ActionMode: CREATE_UPDATE specifies that Cloudformation will be creating or updating resources.

Capabilities: CAPABILITY_NAMED_IAM allows Cloudformation to create IAM roles defined in the template itself.

ParameterOverrides section passes parameter values from AWS CodePipeline (CodePipeline.yml) to the Cloudformation deployment template (Fargate-Cluster.yml) such as ImageURI, Stage and ContainerPort. All of these are defined as parameters in the Fargate Cluster Cloudformation template as well which you’ll see below.


Please note that ImageURI is extracted from the imageDetail.json file using the function Fn::GetParam.

RoleArn property specifies the AWS role that gives permission to AWS Cloudformation service to manipulate AWS resources. This is also defined in the Cloudformation template itself as displayed below.


AWS ExecutionRole for Cloudformation Deployment
Finally, TemplatePath specifies the Cloudformation template (Fargate-Cluster.yml) which contains in the source-output-artifacts which defines the Fargate ECS Cluster infrastructure which we’ll discuss in the next section.

AWS Fargate ECS Cluster Infrastructure in Cloudformation (Fargate-Cluster.yml)
Following are the resources that are needed for the Fargate ECS Cluster. Please refer to the code in Fargate-Cluster.yml file inside the Github repository.

ECS Cluster
VPC
2 Subnets
RouteTable
Route
2 SubnetRouteTableAssociations
InternetGateway
VPCGatewayAttachment
IAM Role for ECS Task Execution
ECS TaskDefinition
2 SecurityGroups
LoadBalancer
TargetGroup
Listener
ECS Service
I won’t be going in to the details of defining all the resources in Cloudformation as most of them are out of the scope of this blog. But you can find more information on how you can create and configure resources from this link.

But we will have a look at the ECS Service configuration.

ECS Service

AWS ECS Service Cloudformation
Above is the Cloudformation syntax for the ECS Service.

DependsOn: LoadBalancerListener configuration lets Cloudformation know that this resource needs to be created only after the LoadBalancer is created. You can specify dependencies this way in Cloudformation.

DesiredCount: 2 specifies there should be 2 instances running.

NetworkConfiguration -> Subnets specifies the 2 subnets the instances should be running. In this case 1 instance will be running in both subnets adhering to the rule of DesiredCount: 2.

LoadBalancers -> ContainerPort defines the port of the container. Note that the value for this parameter is passed by the AWS CodePipeline’s Deploy stage as discussed in previous section.

LoadBalancers -> TargetGroupArn points to the TargetGroup of the LoadBalancer that will be used for this ECS Service.

TaskDefinition points to the TaskDefinition which contains container related information. Following is the Cloudformation template for the TaskDefinition.


AWS TaskDefinition Cloudformation
ContainerDefinitions -> Image specifies the image URL of the ECR. Note that the value for this parameter is passed by the AWS CodePipeline’s Deploy stage as discussed in previous section.

Let’s see how we can deploy this.

Deployment
Go to the Github repo link and fork it to your Github account.
You need to provide access from Github to AWS. This can be done using an Access token. Please refer to this link and generate a token for your Github account.
Download the file CodePipeline.yml file under the Cloudformation directory and open it.
Replace USERNAME, BRANCH andGITHUB ACCESS TOKEN values with your values.
Go to Cloudformation services in AWS and create a new stack.
Upload the CodePipeline.yml file you just edited as displayed below and click Next.
Please make sure the region you are deploying in has at least two availability zones as the 2 subnets inside the VPC (defined in Fargate-Cluster.yml) will be deployed in “a” and “b” availability zones respectively. You can find a list of availability zones from this link.


7. Enter a stack name and click Next. I have specified “CodePipelineStack”.

8. In Review section, click the following checkbox as displayed below.


This is due to the fact that CodePipeline.yml Cloudformation template will be creating IAM roles on our behalf and we need to give our explicit permission for that.

9. Finally hit Create Stack and go to Cloudformation service page and you should be able to see your CodePipeline stack being created as displayed below.


After the CodePipeline stack is created, Head over to AWS CodePipeline service and it should have already started the build stage and start pulling container as displayed below.


Once the CodePipeline Build stage is finished, it will start the Deploy stage and start the deployment of the Fargate-Cluster.yml file as displayed below.


After the CodePipeline has finished, if you go to Cloudformation stacks, you’ll see the Fargate-Cluster.yml file stack will be created as a new stack as below. Which means the Fargate ECS Cluster infrastructure was successfully deployed.


Testing The ECS Cluster Service
After the CodePipeline has finished, Fargate ECS Cluster should be up and running now. We can access the Container service using the Load balancer DNS address.

Go to EC2 Service -> Load Balancers and find the DNS address as displayed below.


Open the DNS address in the browser and you’ll see the following.


Testing The AWS CodePipeline
Now let’s commit a change to the Docker container code (you can edit the output message of the server.js file) in Github and the CodePipeline will automatically update the ECS Cluster as displayed below.


After the CodePipeline deployment is completed, open the DNS address in the browser and you’ll see the ECS Cluster Container is now updated.

How CI/CD Happens Only For The ECS
You must be wondering what’s the point of re-deploying the whole infrastructure if we’re only updating the docker container right? Well that’s not the case.

Let’s dive into our Cloudformation template Fargate-Cluster.yml again.

ContainerDefinitions section contains the Image property which contains the Image URL. Whenever there’s a new push made to the Github repo, this property value gets changes. Note the following in AWS Documentation.


Update requires Replacement meaning,

AWS CloudFormation recreates the resource during an update, which also generates a new physical ID. AWS CloudFormation creates the replacement resource first, changes references from other dependent resources to point to the replacement resource, and then deletes the old resource. (Source)

So basically this means the TaskDefinition will be re-created whenever the Image property value is changed.

Now head over to TaskDefinition property of ECS Service section (Link).


Update requires: No Interruption means,

AWS CloudFormation updates the resource without disrupting operation of that resource and without changing the resource’s physical ID.

This means even though the TaskDefinition gets re-created, the ECS Service itself will not be recreated and it will transition from old TaskDefinition to new TaskDefinition smoothly without any down time.

Bottom line is, the complete infrastructure defined in the Cloudformation template will not be re-created during an update, only the resources containing the property “Update requires Replacement” will be re-created.

Multi Stage Deployments
With the power of Cloudformation, you can easily deploy any number of stages as long as your AWS limits don’t exceed and you have defined the template in a way that supports it.

You only need to change the Stage parameter when you create a new stack uploading the CodePipeline.yml file.

I have created 2 new stacks with stages “prod” and “qa” as displayed below. Please make sure you’re not using capital letters as the stage is appended to S3 buckets and ECR repositories that will be created and their name’s won’t allow capital letters. So keep it simple.


Now you have 2 complete Fargate ECS Cluster infrastructures with their own CI/CD CodePipelines.

Ideally you’d want to point to a separate Github branch as well for your stages. That way when you push to Dev branch, “dev” CodePipeline will fire up and update it’s infrastructure and whenever a push is made to the QA branch, “qa” CodePipeline will fire up and update it’s infrastructure.

Resource Clean up
You can completely delete the infrastructure you created by simple deleting the stacks. You need to delete the Fargate ECS Cluster stack first and then the CodePipeline stack.

Please note, before deleting the CodePipeline stacks, you need to empty the S3 buckets and ECR repositories that were created as it’s not allowed to delete them if they are not empty or you stack deletion will fail.

Summary
You can see how you can leverage Cloudformation and it’s powerful features to Automate things and make your life easy in a Developer and DevOps perspective.

