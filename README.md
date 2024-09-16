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
- Remove image name form this task and replace with:
  ```
  image": "<IMAGE1_NAME>",
  ```
- Remove tags from this json.
  ```
  "tags": []
  ```
