# Deploy Directus on AWS Elastic Container Service with Blue/Green Deployment using CodePipeline.
## Create Directus Image with your custom codes and eveyrthing.
- If you want to use pure directus then you do not need to create custom image, there is already image for directus:
  ```
  docker pull directus/directus:version
  ```
- **What if you have custom extentions and codes?** In this case, we will create custom docker image from node image.
- Make sure your code and Package.json files are properly configured to install directus and run build 
