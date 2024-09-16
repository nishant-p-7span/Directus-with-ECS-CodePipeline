# Deploy Directus on AWS Elastic Container Service with Blue/Green Deployment using CodePipeline.
## Create Directus Image with your custom codes and eveyrthing.
- If you want to use pure directus then you do not need to create custom image, there is already image for directus:
  ```
  docker pull directus/directus:version
  ```
- **What if you have custom extentions and codes?** In this case, we will create custom docker image from node image.
- Make sure your code and Package.json files are properly configured to install directus on `npm i command` and run build command will create build in all the folders.
- Instead of using directus docker image for custom extention, it is preferted to use node image to gain more controll like prefered node version we can choose, which is hard to manage in directus image.
- let's create dockerfile for our custom directus:
  1. Decide which node version is required for you extentions to work. ( I will use `18.17.1 alpine image`)
  2. Decide which package manager require `npm` or `pnpm`. (Here I will user `pnpm`)
  3. Port to expose.
  4. command to run directus.
  5. `curl` command to do health check.
- For better security we will push this node image to ECR and pull it form there. With this we won't need internet access for codebuild and more safer for out project.
- Step:
  1. Create Private ECR repo name node.
  2. Install `awscli` on your device.
  3. Create Secrate access key for account with ecr access.
  4. Using `aws configure` set this access key.
  5. Pull Docker Image `docker pull node:18.17.1-alpine`.
  6. Push this image to ECR repo. Push command can found on aws console and use linux command it will work fine on windows. but make sure to manage tag.
- Docker file:
  ```
  FROM xxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/node:18.17.1-alpine
  
  ENV PNPM_HOME="/pnpm"
  ENV PATH="$PNPM_HOME:$PATH"
  RUN corepack enable
  
  WORKDIR /app
  
  RUN apk update && apk upgrade
  RUN apk --no-cache add curl
  
  COPY . .
  
  RUN npm run start
  
  EXPOSE 8055
  
  CMD ["npx", "directus", "start"]
  ```
- Build Docker image:
  ```
  docker build -t directus .
  ```
- Create another ECR repo for this directus image.
- push this image to ECR as well.
- Now we need to create `TASK DEFINITION`.
## How to manage Sensitive Environment Variables?
- Go to the Parameter Store and Save crucial Environment variables.
- Make sure while storing Secure string is used.
## Task Definition Creation.
- Before we create Task Defi. We need to create IAM roles for Task to access parameter store.
- ECS Task execution role.
  1. Create Role.
  2. ECS --> Elastic Container Service Task.
  3. Select `AmazonECSTaskExecutionRolePolicy`.
  4. Add custom policy:
     ```
     {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Effect": "Allow",
                  "Action": [
                      "ssm:GetParameter",
                      "ssm:GetParameters",
                      "ssm:GetParametersByPath"
                  ],
                  "Resource": [
                      "arn:aws:ssm:us-east-1:xxxxxxxx:parameter/parameter",
                      "arn:aws:ssm:us-east-1:xxxxxxxx:parameter/parameter/*",
                      "arn:aws:ssm:us-east-1:xxxxxxxx:parameter/parameter2",
                      "arn:aws:ssm:us-east-1:xxxxxxxx:parameter/parameter2/*"
                  ]
              }
          ]
      }
     ```
- ECS Task role:
  1. Same method but this time don't include Task Execution policy.
  2. Just add above custom policy for parameter store.
- Create TASK DEFI.:
  1. Select Fargate (I am using FARGATE)
  2. Select OS and architecture and CPU/RAM, and Roles.
  3. Container name, image, Port:8055, protocol:tcp.
  4. Remove last 2 log configuration.
  5. Add Environemnt Variables
  6. For normal environment variable just provide `key`, `Type:value` and `Value:Value`.
  7. For secrates stored in Parameter Store: `Key`, `Type:valueFrom` and `Value:ARN of Parameter`.
  8. Make sure add all the directus ENV variable that is required by your app.
  9. Set up HealthCheck: Command `CMD-SHELL,curl -f http://localhost:8055/server/ping || exit 1` ( Directus helthcheck path is `/server/ping`)
  10. Interval 5 Timeout 5 Start period 30 Retries 3.
  11. Create.
- Copy JSON of this task defi. file we are require this file for codepipeline.
- Create file in the root of your code: `taskdef-stag/prod.json`
- Remove image name form this task and replace with:
  ```
  image": "<IMAGE1_NAME>",
  ```
- Remove tags from this json.
  ```
  "tags": []
  ```
- Here we are done with TASK DEF. Let's create cluster.
## Cluster and service.
- Go to ECS and create ECS cluster with `FARGATE` capacity provider.
- Create security group with HTTP and HTTPs inbound. Add another inbound where all tcp allowed from that security group itself.
- Create service -> Capacity provider strategy, FARGATE 0 1.
- Application type: Service, Family: Task Definition, Service type: replica, Deployment type: Blue/Green. (Create codedeploy role for ECS)
- Netwroking: Select VPC, Choose public subnets only (atleast 2), Select security group that created + RDS ec2-rds security group, Public IP on.
- Loadbalancer: application, Container: directus, Healthcheck: 30, listener 443, target group, blue/green, http, http, /server/ping.
- create. (If everything is alright then your container should be up and running in no time.
- **PLease make sure you make Directus evn varibale `host=0.0.0.0` if it is set to localhost then load balancer will fail in health check.**
## CodeBuild Set up.
- Create one codebuild IAM role with `EC2InstanceProfileForImageBuilderECRContainerBuilds` policy.
- Create Codebuild, Select Service role(enable upadate role option), source: none, Select Compute and other reosurce.
- Set environement variable:
  ```
  AWS_DEFAULT_REGION=us-east-1
  ECR_REPOSITORY_URI=imageuri
  ```
- Create Buildspec:
  ```
  version: 0.2
  phases:
    pre_build:
      commands:
        - echo Logging in to Amazon ECR...
        - aws --version
        - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPOSITORY_URI
        - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
        - IMAGE_TAG=${COMMIT_HASH:=latest}
    build:
      commands:
        - echo Build started on `date`
        - echo set version and release date
        - echo "$COMMIT_HASH | `date`" > version.txt
        - echo Building the Docker image...
        - docker build -t directus .
        - docker tag directus $ECR_REPOSITORY_URI:$IMAGE_TAG
        - docker tag directus $ECR_REPOSITORY_URI:latest
    post_build:
      commands:
        - echo Build started on `date`
        - echo Pushing the Docker images...
        - docker push $ECR_REPOSITORY_URI:latest
        - docker push $ECR_REPOSITORY_URI:$IMAGE_TAG
        - printf '{"Name":"directus","ImageURI":"%s:%s"}' $ECR_REPOSITORY_URI $IMAGE_TAG > imageDetail.json
        - cat imageDetail.json
        - echo Build completed on `date`
  artifacts:
    files: 
      - 'image*.json'
      - 'appspec.yaml'
      - 'taskdef-stag.json'
      - 'taskdef-prod.json'
    secondary-artifacts:
      DefinitionArtifact:
        files:
          - appspec.yaml
          - taskdef.json
          - imageDetail.json
      ImageArtifact:
        files:
          - imageDetail.json
  ```
- Create CodeBuild.
## CodeDeploy and Appspec.
- Go to codedeploy applications, you will find codedeploy created by ECS. edit deployment group. Set Original revision termination to 5 min.
- Create `appspec.yaml` file at root of the code.
  ```
  version: 0.0
  Resources:
    - TargetService:
        Type: AWS::ECS::Service
        Properties:
          TaskDefinition: <TASK_DEFINITION>
          LoadBalancerInfo:
            ContainerName: "directus" # copy from task definition under containerDefinitions.name
            ContainerPort: 8055 # an application container port
  ```
## CodePipeline.
- Create pipeline, Choose source accourding to your need.
- Select CodeBuild that created as build stage.
- Create pipeline.
- Edit pipeline and create stage, from service choose `Amazone ECS (Blue/Green)`.
- Choose codedeploy application for your project.
- source:buildartifact, TaskDefinitionTemplateArtifact:BuildArtifact, TaskDefinitionTemplatePath:taskdef-stag.json
- Image1ArtifactName:BuildArtifact, Image1ContainerName: IMAGE1_NAME.
- AppSpecTemplateArtifact: BuildArtifact, keep name as it is.
- Variable namespace: DeployVariables.
- we are done. Release the changes and check if everything is getting success or not.
