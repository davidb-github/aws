

# AWS DevSecOps Modernization Workshop

In this workshop, you will learn how to add security testing to a CI/CD pipeline of a dockerized .net (Unicorn Store) application using AWS CodeCommit, AWS CodeBuild, and AWS CodePipeline. The modules contained in this workshop will provide you with step-by-step instructions for committing, building, testing, and deploying software in an automation fashion. You will also learn about some basic security tests and where to instrument them in the software development lifecycle. 

For the step by step instructions for completing this workshop please go to https://devsecops.modernize.awsworkshop.io

## Workshop notes below this line since the workshop contains errors and omissions.  

Workshop notes:
https://554309730032.signin.aws.amazon.com/console

I used custom tags for all deployed resources.  
    tags:
    type: workshop

docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]


### docker tag has to be manually built as the repo name in the walkthough and the actual repo commands does not match. 
docker tag aws-modernization-devsecops_unicornstore:latest 554309730032.dkr.ecr.us-west-2.amazonaws.com/modernization-devsecops-workshop:latest

### query the aws ecr describe-repositories to validate repo name matches command string below.
### run this first to login
    eval $(aws ecr get-login --no-include-email) 
### then run docker push command
    docker push $(aws ecr describe-repositories --repository-name modernization-devsecops-workshop --query=repositories[0].repositoryUri --output=text):latest
### Deploy Fargate Service
    aws cloudformation create-stack --stack-name UnicornECS --template-body file://unicorn-store-ecs.yaml --capabilities CAPABILITY_NAMED_IAM

### status check loop
    until [[ `aws cloudformation describe-stacks --stack-name "UnicornECS" --query "Stacks[0].[StackStatus]" --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";   sleep 30; done && echo "The Stack is built at `date` - Please proceed"

### To test, run the following query and copy the URL you obtain from the output into the address bar of a web browser. You should see a photo of a unicorn.
    aws elbv2 describe-load-balancers --names="UnicornStore-LB" --query="LoadBalancers[0].DNSName" --output=text
  ##### expected output from describe command should similar to below:
    UnicornStore-LB-269031093.us-west-2.elb.amazonaws.com

### To deploy the pipeline, run the following commands in Cloud9’s terminal.
    aws cloudformation create-stack --stack-name UnicornPipeline --template-body file://unicorn-store-pipeline.yaml --capabilities CAPABILITY_NAMED_IAM

#### status check optional
    until [[ `aws cloudformation describe-stacks --stack-name "UnicornPipeline" --query "Stacks[0].[StackStatus]" --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";   sleep 30; done && echo "The Stack is built at `date` - Please proceed"

#### Expect a StackID output similar to below.
    {
    "StackId": "arn:aws:cloudformation:us-west-2:554309730032:stack/UnicornPipeline/c12a9a00-ed2c-11ea-b5bf-02f5117d25a9"
    }

#### Inspect the functioning CI/CD pipeline
At this point you should have a fully functioning CI/CD CodePipeline. If you head over to CodePipeline in the AWS console and click on the pipeline that begins with the name UnicorePipeline-Pipeline you will see a pipeline named UnicornPipeline-Pipeline*

#### May see the error below.
        [Container] 2020/09/02 15:00:11 Command did not exit successfully trufflehog --regex --max_depth 1 $APP_SOURCE_REPO_URL exit status 1
    [Container] 2020/09/02 15:00:11 Phase complete: BUILD State: FAILED
    [Container] 2020/09/02 15:00:11 Phase context status code: COMMAND_EXECUTION_ERROR Message: Error while executing command: trufflehog --regex --max_depth 1 $APP_SOURCE_REPO_URL. Reason: exit status 1
    [Container] 2020/09/02 15:00:11 Entering phase POST_BUILD
    [Container] 2020/09/02 15:00:11 Running command echo Build completed on `date`
    Build completed on Wed Sep 2 15:00:11 UTC 2020

    [Container] 2020/09/02 15:00:11 Running command echo Pushing the Docker images...
    Pushing the Docker images...

    [Container] 2020/09/02 15:00:11 Running command docker push $REPOSITORY_URI:latest
    The push refers to repository [554309730032.dkr.ecr.us-west-2.amazonaws.com/modernization-devsecops-workshop]
    An image does not exist locally with the tag: 554309730032.dkr.ecr.us-west-2.amazonaws.com/modernization-devsecops-workshop

    [Container] 2020/09/02 15:00:11 Command did not exit successfully docker push $REPOSITORY_URI:latest exit status 1
    [Container] 2020/09/02 15:00:11 Phase complete: POST_BUILD State: FAILED
    [Container] 2020/09/02 15:00:11 Phase context status code: COMMAND_EXECUTION_ERROR Message: Error while executing command: docker push $REPOSITORY_URI:latest. Reason: exit status 1
    [Container] 2020/09/02 15:00:12 Expanding base directory path: .
    [Container] 2020/09/02 15:00:12 Assembling file list
    [Container] 2020/09/02 15:00:12 Expanding .
    [Container] 2020/09/02 15:00:12 Expanding file paths for base directory .
    [Container] 2020/09/02 15:00:12 Assembling file list
    [Container] 2020/09/02 15:00:12 Expanding imagedefinitions.json
    [Container] 2020/09/02 15:00:12 Skipping invalid file path imagedefinitions.json
    [Container] 2020/09/02 15:00:12 Phase complete: UPLOAD_ARTIFACTS State: FAILED
    [Container] 2020/09/02 15:00:12 Phase context status code: CLIENT_ERROR Message: no matching artifact paths found



### unicorn-store/buildspec.yml
    version: 0.2

    phases:
    install:
        runtime-versions:
        docker: 18
        dotnet: 2.2
        python: 3.7
        ruby: 2.6
    pre_build:
        commands:
        - $(aws ecr get-login --region $AWS_DEFAULT_REGION --no-include-email)
        - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
        - IMAGE_TAG=${COMMIT_HASH:=latest}
        - echo Setting CodeCommit Credentials
        - git config --global credential.helper '!aws codecommit credential-helper $@'
        - git config --global credential.UseHttpPath true
        - echo Installing TruffleHog
        - pip install TruffleHog

    build:
        commands:
        - echo Running TruffleHog Secrets Scan
        - trufflehog --regex --max_depth 1 $APP_SOURCE_REPO_URL
        - echo Scanning with Hadolint
        - docker run --rm -i hadolint/hadolint < Dockerfile
        - echo Build started on `date`
        - echo Building the Docker image...
        - docker build -t $REPOSITORY_URI:latest .
        - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG

    post_build:
        commands:
        - echo Build completed on `date`
        - echo Pushing the Docker images...
        - docker push $REPOSITORY_URI:latest
        - docker push $REPOSITORY_URI:$IMAGE_TAG
        - echo Writing image definitions file...
        - printf '[{"name":"modernization-devsecops-workshop_unicornstore","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
    artifacts:
    files: imagedefinitions.json




## CI/CD pipeline being deployed in this workshop

1. Development and local testing: The developer will work on coding tasks with the Cloud9 IDE environment. Once completed with the task the developer will commit her/his changes to the local git repository and test the changes.
2. Push to remote master branch: When the developer is satisfied with the software changes, the developer will push those changes to the remote master branch. In this workshop this is the AWS CodeCommit Repo.
3. AWS CodePipeline - Commit Event: CodePipeline will monitor AWS CodeCommit for any new commits. When a new commit (code change) is detected a CodeBuild job will be triggered.
4. AWS CodePipeline - Build: Within CodeBuild a series of security tests will be instrumented to validate that code changes are not adding security risks to the application. If security issues are detect, this phase of the CodePipeline process is ended and the pipeline process reports a failure status. If no security issues are detected the build process continues by building and packaging the application.
5. AWS CodePipeline - Postbuilld: Runs additional tests and if those test pass the container is pushed to Amazon ECR.
6. AWS CodePipeline - Deploy: If the build process is successful, Codepipeline will update Elastic Container Service that there is a new image. The new image will be deployed and monitored. If all healthchecks pass the new deployment will become the primary service and the previous deployment will be gracefully shutdown.
7. Monitor: ECS and the Application Load Balancer will continually monitor the health of the container and the application and will make the required adjustments to keep the minimum number of healthly container tasks running at all times.



<!-- 
REPOSITORY_URI 554309730032.dkr.ecr.us-west-2.amazonaws.com/modernization-devsecops-workshop

Try changing to:
554309730032.dkr.ecr.us-west-2.amazonaws.com/modernization-devsecops-workshop:latest




 -->




<!-- {
    "repositories": [
        {
            "repositoryArn": "arn:aws:ecr:us-west-2:554309730032:repository/modernization-devsecops-workshop",
            "registryId": "554309730032",
            "repositoryName": "modernization-devsecops-workshop",
            "repositoryUri": "554309730032.dkr.ecr.us-west-2.amazonaws.com/modernization-devsecops-workshop",
            "createdAt": 1599054048.0,
            "imageTagMutability": "MUTABLE",
            "imageScanningConfiguration": {
                "scanOnPush": false
            },
            "encryptionConfiguration": {
                "encryptionType": "AES256"
            }
        }
    ]
}

docker tag modernization-devsecops-workshop_unicornstore:latest $(aws ecr describe-repositories --repository-name modernization-devsecops-workshop --query=repositories[0].repositoryUri --output=text):latest

docker tag modernization-devsecops-workshop_unicornstore:latest 554309730032.dkr.ecr.us-west-2.amazonaws.com/modernization-devsecops-workshop:latest -->
