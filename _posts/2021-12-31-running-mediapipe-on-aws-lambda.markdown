---
layout: post
title:  "Running MediaPipe on AWS Lambda"
date:   2021-12-31 13:20:03 +0100
categories: aws
---
MediaPipe is a framework for building multimodal (eg. video, audio, any time series data), cross platform (i.e Android, iOS, web, edge devices) applied ML pipelines. With MediaPipe, a perception pipeline can be built as a graph of modular components, including, for instance, inference models (e.g., TensorFlow, TFLite) and media processing functions. In some cases there is a need to run tools like MediaPipe on demand and not in regular basis where AWS Lambda looks like a perfect solution. 

I will try to go through whole process of creating docker image with Python and MediaPipe, then will walk through setting up docker image as a AWS Lambda function.

# AWS Lambda unction code

A basic handler function which reads files for processing with MediaPipe from AWS S3 looks like:

```python
# application.py
from typing import Any
import cv2
import json
import os
import boto3
from google.protobuf.json_format import MessageToDict
import mediapipe as mp

from aws_lambda_typing import context as context_, events

mp_pose = mp.solutions.mediapipe.python.solutions.holistic

class Response:
    statusCode: str
    body: Any

    def __init__(self, status: str, data: Any = None, message: str = None) -> None:
        self.status = status
        self.data = data
        self.message = message

    def toJSON(self):
        return json.dumps(self, default=lambda o: o.__dict__,
                          sort_keys=True, indent=4)

    def toLambdaResponse(self):
        return {"statusCode": self.status, "body": {"message": self.message, "data": self.data}}


def handler(event: events.APIGatewayProxyEventV2, context: context_.Context):
    pose = mp_pose.Holistic(
        static_image_mode=True,
        model_complexity=2,
        enable_segmentation=True,
        min_detection_confidence=0.5)

    bucket = event.get('bucket') or os.environ.get('S3_BUCKET_NAME')
    key = event.get('key')

    if bucket is None or key is None:
        return Response(409, message="Bad request").toLambdaResponse()

    local_file_path = f'/tmp/{key}'

    boto3.client('s3').download_file(bucket, key, local_file_path)

    image = cv2.imread(local_file_path)

    # Convert the BGR image to RGB before processing.
    results = pose.process(cv2.cvtColor(image, cv2.COLOR_BGR2RGB))

    if not results.pose_landmarks:
        return Response(500, message="Pose not detected").toLambdaResponse()

    return Response(200, {
        "pose_landmarks": MessageToDict(results.pose_landmarks),
        "pose_world_landmarks": MessageToDict(results.pose_world_landmarks),
        "face_landmarks": MessageToDict(results.face_landmarks)
    }).toLambdaResponse()


if __name__ == "__main__":
    print(handler(None, None))

```

Next thing I need is a function which will be invoked during the docker image building and it will trigger the download of the models used in MediaPipe. I did not found any other way for downloading this eve tough I expected some different way of accomplishing this. It is desirable that models are downloaded during image building to not wait for them during the runtime.

```python
import cv2
import mediapipe as mp

mp_pose = mp.solutions.mediapipe.python.solutions.holistic

pose = mp_pose.Holistic(
    static_image_mode=True,
    model_complexity=2,
    enable_segmentation=True,
    min_detection_confidence=0.5)


file = './example-pose.png'
image = cv2.imread(file)
# Convert the BGR image to RGB before processing.
results = pose.process(cv2.cvtColor(image, cv2.COLOR_BGR2RGB))

print('Initialization finished')
```

# Preparing docker image

First decision should be made on a [base runtime image][python-image]. I will start with Python 3.8 image. The only system requirement so far is `mesa-libGLw` which is installed in the second step. 

```Dockerfile
FROM public.ecr.aws/lambda/python:3.8

RUN yum install -y mesa-libGLw
```

Python `requirements.txt` should contain this dependencies:

```text
mediapipe==0.8.8.1
numpy==1.21.2
boto3==1.20.23
boto3-stubs>=1.0
aws_lambda_typing==2.9.2
```

Next step is to install all the requirements in the `LAMBDA_TASK_ROOT` which is the path to your Lambda function code.

```Dockerfile
COPY requirements.txt  .
RUN  pip3 install -r requirements.txt --target "${LAMBDA_TASK_ROOT}"
```

At the end `pthread_shim.so` is copied and it's path will be used later when setting up AWS Lambda function. For further understanding of the purpose of this file check [source repository][nameshim-source] and one of the MediaPipe [issues][nameshim-issue]. There also function code is copied to image and MediaPipe is triggered while building image so it downloads models and they are stored in the image. CMD is set to match the python file and name of the handler function in it.

```Dockerfile
# https://github.com/mitchellharper12/lambda-pthread-nameshim
COPY --from=make-pthread-nameshim /pthread-nameshim/pthread_shim.so /opt/pthread_shim.so

# Copy function code
COPY src/ ${LAMBDA_TASK_ROOT}

# Download models
RUN python init_models.py

# Set the CMD to your handler (could also be done as a parameter override outside of the Dockerfile)
CMD [ "application.handler" ] 
```

# Building docker image

Next step is to build docker image and push it to AWS ECR to make it available for use with AWS lambda by running a command `docker build -t aws_lambda_mediapipe`. Before pushing image an [AWS ECR repository should be created]({% post_url 2021-12-29-create-aws-ecr-with-aws-cli %}) and [docker authorized]({% post_url 2021-12-29-docker-login %}) to push to it.

Docker image can also be tested locally by running it:
```shell
docker run --rm --env AWS_ACCESS_KEY_ID="***" --env AWS_SECRET_ACCESS_KEY="***" -p 9000:8080 aws_lambda_mediapipe:latest
```

and executing an HTTP request to it: 
```shell
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'
# or
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{"bucket": "bucket-name", "key": "image10.jpg"}'
```

Now docker image should be tagged with remote repository and pushed to it:

```shell
docker tag aws_lambda_mediapipe:latest 1234567890.dkr.ecr.us-west-1.amazonaws.com/aws_lambda_mediapipe:latest

docker push 1234567890.dkr.ecr.us-west-1.amazonaws.com/aws_lambda_mediapipe:latest
```

# Setting up AWS Lambda function

Now having a docker image on AWS ECR, AWS Lambda function should be created. Before creating function AWS Lambda execution role is [needs to be created][lambda-execution-role] and then its reference user as a `--role` parameter while creating function.

```shell
aws lambda create-function \
    --function-name aws-lambda-mediapipe \
    --package-type Image \
    --code ImageUri=1234567890.dkr.ecr.us-west-1.amazonaws.com/aws_lambda_mediapipe:latest \
    --role arn:aws:iam::1234567890:role/lambda-execution-role
```

For running mediapipe test `--memory-size` should be increased and I think that 5120MB for this test is enough. This can later be changed to match the use case. The same applies to `--timeout`. Here also a path to `pthread_shim.so` [trick][ld-preload-trick] is set through `--envoironment` variables.

```shell
aws lambda update-function-configuration \
    --function-name aws-lambda-mediapipe \
    --timeout 30 \
    --memory-size 5120 \
    --environment "Variables={LD_PRELOAD=/opt/pthread_shim.so}"
```

AWS Lambda function can also be tested with AWS CLI. Here are the examples. For better understanding of the commands look at [official documentation][aws-cli-docs].

```shell
aws lambda invoke --function-name aws-lambda-mediapipe response.json
aws lambda invoke-async --invoke-args ./args.json --function-name aws-lambda-mediapipe
```

# What next?
This post tries to give idea on how to run MediaPipe on AWS Lambda. Next steps should be implementing additional logic to handler functions or using AWS Lambda results. AWS Lambda function can be invoked through HTTP using AWS API Gateway.

[python-image]: https://docs.aws.amazon.com/lambda/latest/dg/python-image.html
[nameshim-source]: https://github.com/mitchellharper12/lambda-pthread-nameshim
[nameshim-issue]: https://github.com/google/mediapipe/issues/1599
[lambda-execution-role]: https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html
[ld-preload-trick]: https://stackoverflow.com/questions/426230/what-is-the-ld-preload-trick
[aws-cli-docs]: https://docs.aws.amazon.com/cli/latest/reference/lambda/invoke.html