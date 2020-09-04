# ECS Demo
Deploy a container based application to AWS ECS using Fargate. Additionally, build CI/CD pipeline to automate deployments when new commits are made in Github. 

### Prerequisites  
- aws account with membership in a group w/ AdministratorAccess policy applied
- awscli installed and configured  
- Docker installed and running  
``` docker-machine restart```  
    Probably has a new ip if machine was rebooted since last use.  
``` docker-machine env ```  
``` eval $(docker-machine env)```  






### AWS Service Elements 
- ECS
- ECR
- CodePipeline

### Demo Sections
1. Container application and ECR creation
2. Create VPC & Subnets
3. Connect all services and test
4. Build CI/CD Pipeline to automate deployment

## Container app testing and AWS ECR creation
### Build the local image
1. Fork and clone the repo down locally.  
``` git clone https://github.com/davidb-github/cicd-container.git ```
2. Build the Docker image  
``` docker build -t demo1-local:latest .```  
3. docker images command should show the newly built image.  
``` REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE ```
``` demo1-local         latest              fdf3a26dfd98        5 minutes ago       58.7MB ```
4. Issue the docker run command using the image id from the docker images command.  
``` docker run -it -p 80:3000 fdf3a26dfd98```  

### Push the image to Amazon ECR  
1. Login to the aws console  
2. Create a new ECR repository  
3. Authenticate to the newly created registry.  Note: Use AWS as the --username in the docker login cmd. AWS must be all caps. This is a strange work-around on the aws side.  
```  aws --region us-east-2 ecr get-login-password | docker login --username AWS --password-stdin 554309730032.dkr.ecr.us-east-2.amazonaws.com/demo1```  

You will see a warning message similar to the text below.  
```WARNING! Your password will be stored unencrypted in /Users/davidb/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store
```







```{
  "Version": "2008-10-17",
  "Statement": [
    {
      "Sid": "AllowPushPull",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::554309730032:user/admin-cli",
          "arn:aws:iam::554309730032:user/Administrator"
        ]
      },
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:CompleteLayerUpload",
        "ecr:GetDownloadUrlForLayer",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ]
    }
  ]
} 














1. fork repo so codepipeline can be wired up.


## Create ECS Cluster







## Codepipeline
1. Create new pipeline



##### Reference:  
- https://github.com/mjzone/cicd-container  
- 
