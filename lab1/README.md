# AWS Container Immersion Day: Lab 1

## Overview of lab

This lab introduces the basics of working with microservices and [ECS](https://aws.amazon.com/ecs/). This includes: preparing two microservice container images, setting up the initial ECS cluster, and deployment of the containers with traffic routed through an [ALB](https://aws.amazon.com/elasticloadbalancing/applicationloadbalancer/).

You&#39;ll need to [have a working AWS account](https://aws.amazon.com/) to use this lab.

#

## 1. Setting up the VPC

We will create a new VPC for our entire infrastructure. We need 2 public subnets, for our developer workstation, ECS cluster and the ALB.

**Note** : If students in this lab are using a shared AWS account &amp; VPC, skip this step of creating the VPC.  When using a shared AWS account, to avoid confusion and conflicts, be sure to name/tag AWS resources (security groups, IAM roles, instances, clusters, repositories, Docker image tags, etc.), according to your organizations naming conventions or at the very least, choose descriptive names to distinguish your resources from the other students&#39; resources (i.e. prefix the resource names with your name).

Skip to step 2 if you&#39;re using an existing VPC.  Otherwise, configure a VPC with the following requirements:

| **field** | **value** |
| --- | --- |
| Name tag | ECS Lab VPC |
| IPv4 CIDR | 10.0.0.0/16 |
| **Subnet a** |   |
| Name tag | Public subnet a |
| CIDR | 10.0.0.0/24 |
| **Subnet b** |   |
| Name tag | Public subnet b |
| CIDR | 10.0.1.0/24 |

## 2. Setting up the IAM user and roles

In order to work with ECS from our workstation, we will need the appropriate permissions for our developer workstation instance.   Go to the [IAM Console](https://console.aws.amazon.com/iam/home), **Roles** \&gt; **Create New Role \&gt; AWS Service \&gt; EC2.** We will later assign this role to our workstation instance.

Click **Next.**

Enter **AmazonEC2ContainerRegistryFullAccess** in the Filter text field.

Click **Next.**

Enter **ecslabworkstationprofile** for the Role name and click **Create Role**.

Use the same process to create another new role so that EC2 instances in the ECS cluster have appropriate permissions to access the container registry, auto-scale, etc. We will later assign this role to the EC2 instances in our ECS cluster.

In the Create Role screen, enter AmazonEC2ContainerServiceforEC2Role AmazonEC2ContainerServiceAutoscaleRole in the text field (without a comma) and select the two policies.

In the Review screen, enter **ecslabinstanceprofile** for the Role name and click **Create Role**.



**Note** : By default, the ECS first run wizard creates ecsInstanceRole for you to use. However, it&#39;s a best practice to create a specific role for your use so that we can add more policies in the future when we need to.

## 3. Launching the Cluster

Next, let&#39;s launch the ECS cluster which will host our container instances. We&#39;re going to put these instances in the public subnets since they&#39;re going to be hosting public microservices.

Create a new security group by navigating to [the EC2 console \&gt; Security Group](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#SecurityGroups:sort=tag:Name) and create sgecslabpubliccluster. Keep the defaults.  Make sure the correct VPC is selected when creating the security group.

Navigate to the [ECS console](https://console.aws.amazon.com/ecs/) and click Create Cluster. **Choose the**  **EC2 Linux + Networking**  **cluster template. Click**  **Next Step****.**

**In the next screen, configure the cluster as follows:**

| **Field Name** | **Value** |
| --- | --- |
| Cluster Name | EcsLabPublicCluster |
| Provisioning Model | On-Demand Instance |
| EC2 instance type | t2.micro |
| Number of instances | 2 |
| EBS storage | 22 |
| Keypair | none |
| **Networking Section** |   |
| VPC | ECS Lab VPC [or name of shared VPC] |
| Subnets | pick 2 public subnets |
| Security Group | sgecslabpubliccluster |
| IAM Role | ecslabinstanceprofile |

Click **Create**.  It will take a few minutes to create the cluster.

­

## 4. Launching the Workstation

Next, let&#39;s launch our developer workstation. Think of this as the developer&#39;s machine which runs Docker and has access to our Git repository.

Navigate to the [EC2 Console](https://console.aws.amazon.com/ec2) **\&gt; Launch Instance**

| **Field Name** | **Value** |
| --- | --- |
| **Step 1:** AMI: | Amazon Linux AMI 2018.03.0 (HVM) [or the latest Amazon Linux AMI] |
| **Step 2** : Instance type: | t2.micro |
| **Step 3: Configure Instance Details** | |
| Network: | ECS Lab VPC or your shared VPC |
| Subnet: | one of the public subnets |
| Auto-assign Public IP: | enable |
| IAM Role: | ecslabworkstationprofile |
| **Next, Step 4: Storage** | (leave default) |
| **Next, Step 5:Tags** | Add Tag |
| Name: | ecs-lab-workstation |
| **Next, Step 6:Security Group** | create a new security group |
| Name: | sgecslabworkstation |
| Inbound rules: | SSH TCP 22 Source: My IP |
| **Step 7: Review and launch** | Choose an existing keypair or generate a new one |

Once the instance is running, SSH into it via its public DNS:

$ ssh -i cert.pem ec2-user@[public DNS]

Update to the latest AWS CLI:

$ sudo yum update -y

Install docker:

$ sudo yum install -y docker

$ sudo service docker start

Add ec2-user to the docker group so you can execute Docker commands without using sudo:

$ sudo usermod -a -G docker ec2-user

Exit and SSH in again to pick up the new permissions.

Verify docker is configured correctly:

$ docker info

Containers: 0

 Running: 0

 Paused: 0

 Stopped: 0

Images: 0

Server Version: 17.03.1-ce

Storage Driver: overlay2

 Backing Filesystem: extfs

 Supports d\_type: true

 Native Overlay Diff: true

Logging Driver: json-file

Cgroup Driver: cgroupfs

Plugins:

 Volume: local

 Network: bridge host macvlan null overlay

Swarm: inactive

Runtimes: runc

Default Runtime: runc

Init Binary: docker-init

containerd version:  (expected: 4ab9917febca54791c5f071a9d1f404867857fcc)

runc version: N/A (expected: 54296cf40ad8143b62dbcaa1d90e520a2136ddfe)

init version: N/A (expected: 949e6facb77383876aeff8a6944dde66b3089574)

Security Options:

 seccomp

  Profile: default

Kernel Version: 4.9.32-15.41.amzn1.x86\_64

Operating System: Amazon Linux AMI 2017.03

…

We now have a working developer workstation.

## 5. Prepping the Docker images

At this point, we&#39;re going to pretend that we&#39;re the developers of both the web and api microservices, and we will get the latest from our source repo. In this case we will just be using the plain old curl, but just pretend you&#39;re using git:

$ curl -O [https://s3-us-west-2.amazonaws.com/apn-bootcamps/microservice-ecs-2017/ecs-lab-code-20170524.tar.gz](https://s3-us-west-2.amazonaws.com/apn-bootcamps/microservice-ecs-2017/ecs-lab-code-20170524.tar.gz)

$ tar -xvf ecs-lab-code-20170524.tar.gz

Our first step is to build and test our containers locally. If you&#39;ve never worked with Docker before, there are a few basic commands that we&#39;ll use in this workshop, but you can find a more thorough list in the [Docker &quot;Getting Started&quot; documentation](https://docs.docker.com/get-started/).

To build your first container, go to the web directory. This folder contains our web Python Flask microservice:

$ cd \&lt;path/to/project\&gt;/aws-microservices-ecs-bootcamp-v2/web

To build the container:

$ docker build -t ecs-lab/web .

This should output steps that look something like this:

Sending build context to Docker daemon 4.096 kB

Sending build context to Docker daemon

Step 0 : FROM ubuntu:latest

 ---\&gt; 6aa0b6d7eb90

Step 1 : MAINTAINER widha@amazon.com

 ---\&gt; Using cache

 ---\&gt; 3f2b91d4e7a9

If the container builds successfully, the output should end with something like this:

 Removing intermediate container d2cd523c946a

 Successfully built ec59b8b825de

To view the image that was just built:

 $ docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE

ecs-lab/web         latest              2b849343f6be        13 seconds ago      452MB

ubuntu              latest              113a43faa138        12 days ago         81.2MB

To run your container:

 $ docker run -d -p 3000:3000 ecs-lab/web

This command runs the image in daemon mode and maps the docker container port 3000 with the host (in this case our workstation) port 3000. We&#39;re doing this so that we can run both microservices on a single host without port conflicts.

To check if your container is running:

 $ docker ps

This should return a list of all the currently running containers. In this example, it should just return a single container, the one that we just started:

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES

7b0d04f4502c        ecs-lab/web         &quot;python app.py&quot;     9 seconds ago       Up 9 seconds        0.0.0.0:3000-\&gt;3000/tcp   eloquent\_noether

To test the actual container output:

 $ curl localhost:3000/web

This should return:

 \&lt;html\&gt;\&lt;head\&gt;...\&lt;/head\&gt;\&lt;body\&gt;hi!  i&#39;m served via Python + Flask.  i&#39;m a web endpoint. ...\&lt;/body\&gt;\&lt;/html\&gt;

Repeat the same steps with the api microservice. Change directory to /api and repeat the same steps above:

$ cd ../api

$ docker build -t ecs-lab/api .

$ docker images

$ docker run -d -p 8000:8000 ecs-lab/api

$ curl localhost:8000/api

The API container should return:

{ &quot;response&quot; : &quot;hi!  i&#39;m ALSO served via Python + Flask.  i&#39;m an API.&quot; }

We now have two working microservice containers.

## 6. Creating container registries with ECR

Once images are built, it&#39;s useful to share them and this is done by pushing the images to a container registry.  Let&#39;s create two repositories in Amazon EC2 Container Registry ([ECR](https://aws.amazon.com/ecr/)).

Navigate to the [ECS console](https://console.aws.amazon.com/ecs/), and select **Repositories** and choose **Create repository**.

Name your first repository **ecs-lab-web** :

Once you&#39;ve created the repository, it will display the push commands. Take note of these, as you&#39;ll need them in the next step. The push commands should like something like this:

Once you&#39;ve created the ecs-lab-web repository, repeat the process for the **ecs-lab-api** repository. Take note of the push commands for this second repository. Push commands are unique per repository.

## 7. Configuring the AWS CLI

On our workstation, we will use the AWS CLI to push images to ECR. Let&#39;s configure the CLI by running:

$ aws configure

This should drop you into a set of prompts.  Since our workstation is an EC2 instance pre-configured in an IAM role, the only information required is your preferred region:

$ aws configure

AWS Access Key ID: \&lt;leave empty\&gt;

AWS Secret Access Key: \&lt;leave empty\&gt;

Default region name [us-east-1]: us-east-1

Default output format [json]: \&lt;leave empty\&gt;

You can confirm that your CLI is setup correctly by running the command to obtain an ECR authentication token.

$ aws ecr get-login

This should output something like:

docker login -u AWS -p AQECAHhwm0YaISJeRtJm5n1G6uqeekXuoXXPe5UFce9Rq8/14wAAAy0wggMpBgkqhkiG9w0BBwagggMaMIIDFgIBADCCAw8GCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQM+76slnFaYrrZwLJyAgEQgIIC4LJKIDmvEDtJyr7jO661//6sX6cb2jeD/RP0IA03wh62YxFKqwRMk8gjOAc89ICxlNxQ6+cvwjewi+8/W+9xbv5+PPWfwGSAXQJSHx3IWfrbca4WSLXQf2BDq0CTtDc0+payiDdsXdR8gzvyM7YWIcKzgcRVjOjjoLJpXemQ9liPWe4HKp+D57zCcBvgUk131xCiwPzbmGTZ+xtE1GPK0tgNH3t9N5+XA2BYYhXQzkTGISVGGL6Wo1tiERz+WA2aRKE+Sb+FQ7YDDRDtOGj4MwZ3/uMnOZDcwu3uUfrURXdJVddTEdS3jfo3d7yVWhmXPet+3qwkISstIxG+V6IIzQyhtq3BXW/I7pwZB9ln/mDNlJVRh9Ps2jqoXUXg/j/shZxBPm33LV+MvUqiEBhkXa9cz3AaqIpc2gXyXYN3xgJUV7OupLVq2wrGQZWPVoBvHPwrt/DKsNs28oJ67L4kTiRoufye1KjZQAi3FIPtMLcUGjFf+ytxzEPuTvUk4Xfoc4A29qp9v2j98090Qx0CHD4ZKyj7bIL53jSpeeFDh9EXubeqp6idIwG9SpIL9AJfKxY7essZdk/0i/e4C+481XIM/IjiVkh/ZsJzuAPDIpa8fPRa5Gc8i9h0bioSHgYIpMlRkVmaAqH/Fmk+K00yG8USOAYtP6BmsFUvkBqmRtCJ/Sj+MHs+BrSP7VqPbO1ppTWZ6avl43DM0blG6W9uIxKC9SKBAqvPwr/CKz2LrOhyqn1WgtTXzaLFEd3ybilqhrcNtS16I5SFVI2ihmNbP3RRjmBeA6/QbreQsewQOfSk1u35YmwFxloqH3w/lPQrY1OD+kySrlGvXA3wupq6qlphGLEWeMC6CEQQKSiWbbQnLdFJazuwRUjSQlRvHDbe7XQTXdMzBZoBcC1Y99Kk4/nKprty2IeBvxPg+NRzg+1e0lkkqUu31oZ/AgdUcD8Db3qFjhXz4QhIZMGFogiJcmo= -e none https://\&lt;account\_id\&gt;.dkr.ecr.us-east-1.amazonaws.com

To register ECR as your Docker repository, copy and paste that output or run:

$ `aws ecr get-login --region us-east-1`

Your shell will execute the output of that command and respond:

Login Succeeded

If you are unable to login to ECR, check your IAM permissions.

## 8. Pushing our tested images to ECR

Now that we&#39;ve tested our images locally, we need to tag and push them to ECR. This will allow us to use them in Task Definitions that can be deployed to an ECS cluster.

You&#39;ll need your push commands that you saw during registry creation. You can find them again by going back to the repository ( **ECS Console** \&gt; **Repositories** \&gt; Select the Repository you want to see the commands for \&gt; **View Push Commands)**.

To tag and push to the web repository (if you&#39;re using a shared account, use your name in the tag: fred-ecs-lab:latest):

$ docker tag ecs-lab/web:latest \&lt;account\_id\&gt;.dkr.ecr.us-east-1.amazonaws.com/ecs-lab-web:latest

$ docker push \&lt;account\_id\&gt;.dkr.ecr.us-east-1.amazonaws.com/ecs-lab-web:latest

This should return something like this:

The push refers to a repository [\&lt;account\_id\&gt;.ecr.us-east-1.amazonaws.com/ecs-lab-web] (len: 1)

ec59b8b825de: Image already exists

5158f10ac216: Image successfully pushed

860a4e60cdf8: Image successfully pushed

6fb890c93921: Image successfully pushed

aa78cde6a49b: Image successfully pushed

Digest: sha256:fa0601417fff4c3f3e067daa7e533fbed479c95e40ee96a24b3d63b24938cba8

To tag and push to the api repository:

$ docker tag ecs-lab/api:latest \&lt;account\_id\&gt;.dkr.ecr.us-east-1.amazonaws.com/ecs-lab-api:latest

$ docker push \&lt;account\_id\&gt;.dkr.ecr.us-east-1.amazonaws.com/ecs-lab-api:latest

**Note** : why :latest? This is the actual image tag. In most production environments, you&#39;d tag images for different schemes, for example, you might tag the most up-to-date image with :latest, and all other versions of the same container with a commit SHA from a CI job. If you push an image without a specific tag, it will default to :latest, and untag the previous image with that tag. For more information on Docker tags, see the Docker [documentation](https://docs.docker.com/engine/reference/commandline/tag/).

You can see your pushed images by viewing the repository in the [ECS Console](https://console.aws.amazon.com/ecs). Alternatively, you can use the CLI:

$ aws ecr list-images --repository-name=ecs-lab-api

{

    &quot;imageIds&quot;: [        {

            &quot;imageTag&quot;: &quot;latest&quot;,

            &quot;imageDigest&quot;: &quot;sha256:f0819d27f73c7fa6329644efe8110644e23c248f2f3a9445cbbb6c84a01e108f&quot;

        }

    ]

}

**You have successfully completed Lab 1..  Keep all the infrastructure you have built running. You will be buildin**** g on this in Lab 2**
