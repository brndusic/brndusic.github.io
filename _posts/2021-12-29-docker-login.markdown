---
layout: post
title:  "docker login"
date:   2021-12-29 13:20:03 +0100
categories: docker
---
Using docker commands in local environment with public repositories like docker hub is pretty easy because it does not require authorization. When talking about private repositories authorization is needed and there we need `docker login` command.

Full reference for `docker login` command can be found on [official docker documentation][official-docker-documentation]

But let's see how it will probably look like when authorizing to AWS ECR with.

```shell
aws --profile [profile-name] ecr get-login-password | docker login --username AWS --password-stdin [account-id].dkr.ecr.us-west-1.amazonaws.com/[ecr-name]
```

# Lets explain the whole command and parameters. 

Here we can see this one line command where we need tu substitute some placeholder marked with `[]` brackets. 

First `aws --profile [profile-name] ecr get-login-password` is executed where `[profile-name]` should be replaced or with real profile or if default is used `--profile [profile-name]` should be omitted. Result of this command is a base64 string which is piped to the next step. In

Next step is `docker login --username AWS --password-stdin [account-id].dkr.ecr.us-west-1.amazonaws.com/[ecr-name]` where `[account-id]` should be replaced with account id under which AWS ECR is created and `[ecr-name]` should be replaced with ECR name. This link can be obtained easily from AWS Console or AWS CLI.

If everything goes well `Login Succeeded` should be printed in CLI.

# How to login with username and password?

In some cases we have just plain username and passwords and in that case a simple command is enough to login.
```shell
docker login --username foo --password
```

# Credentials store is supported

Native keychain of the operating system can be used for securely storing docker credentials. It should be properly configured and instructions can be found on [official docker documentation][official-docker-documentation].

[official-docker-documentation]: https://docs.docker.com/engine/reference/commandline/login/