---
layout: post
title:  "Create and configure AWS ECR with AWS CLI"
date:   2021-12-29 14:20:03 +0100
categories: aws
---
Using AWS Console for creating AWS cloud resources is very convenient for exploring the platform, doing smaller actions with aws resources, but when something advanced is necessary [AWS CLI][aws-cli] jumps in. Here I will try to show how with a couple of simple commands ECR can be created and managed 

To create repository simply run:
```shell
aws ecr create-repository --repository-name foo-repo
```

It is recommended to configure vulnerabilities scanning to be performed on every push.
```shell
aws ecr put-image-scanning-configuration --repository-name foo-repo --image-scanning-configuration scanOnPush=true
```

To check the configuration and review basic information of the registry run the command:
```shell
aws ecr describe-repositories --repository-names foo-repo
```

After registry is configured docker images can be pushed to it using docker commands. Currently I will not describe process of building an image but I will imply that image exists locally and describe the part of tagging and pushing to ECR. Pushing to ECR can be done only if docker is logged in. Instructions for authorizing with docker can be found on my [`docker login` post]({% post_url 2021-12-29-docker-login %}).

Local image needs to be tagged with remote repository and then pushed to it.
```shell
docker tag local_image_name:latest 1234567890.dkr.ecr.us-west-1.amazonaws.com/foo-image:latest

docker push 1234567890.dkr.ecr.us-west-1.amazonaws.com/foo-image:latest
```

To find real repository links run previous command for describing registry.

[aws-cli]: https://aws.amazon.com/cli/