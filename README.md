
# Productioning a Model with Docker and SageMaker

## Introduction

In this codealong, we'll see an example of the steps needed in order to productionize our own model with AWS SageMaker. 

## How to Use This Notebook

This notebook contains a lot of _boilerplate code_ provided by Amazon which you'll need to make use of pretty much anytime you need to productionize a model. Much of this can be copied and pasted over, although it's likely that some modifications will be needed in order on a project-by-project basis. Going forward, use this notebook as a desk reference to remember the necessary steps for productionizing a model with AWS SageMaker--you are _not_ expected to remember or even understand most of the code in this notebook, as you aren't yet familiar with AWS and its quirks. That's okay--notebooks like these are meant to help you through productionizing a model until you've done it a few times and start to understand the process and the necessary code needed to make it all work!

### NOTE: To use this notebook, you'll need to upload it as a notebook inside of SageMaker and run it there!

This notebook borrows it's general structure, as well as all boilerplate code, from this [training repo](https://github.com/aws-samples/amazon-sagemaker-keras-text-classification) provided publicly by AWS. For more examples of how to use various AWS services, check out the [AWS-Samples Repository]()!

## Step 1: Building and Registering the Container

If you at the `container` folder within this repo, you'll see it contains some docker images. These are docker images that you'll need to make sure you use in your own projects when productionizing a model on AWS. 

Everything in the cell below is boilerplate code that uses the `containers` folder in order to create and register the docker image needed on AWS. 


```sh
%%sh

# The name of our algorithm
algorithm_name=sagemaker-keras-text-classification

cd container

chmod +x sagemaker_keras_text_classification/train
chmod +x sagemaker_keras_text_classification/serve

account=$(aws sts get-caller-identity --query Account --output text)

# Get the region defined in the current configuration (default to us-west-2 if none defined)
region=$(aws configure get region)
region=${region:-us-west-2}

fullname="${account}.dkr.ecr.${region}.amazonaws.com/${algorithm_name}:latest"

# If the repository doesn't exist in ECR, create it.

aws ecr describe-repositories --repository-names "${algorithm_name}" > /dev/null 2>&1

if [ $? -ne 0 ]
then
    aws ecr create-repository --repository-name "${algorithm_name}" > /dev/null
fi

# Get the login command from ECR and execute it directly
$(aws ecr get-login --region ${region} --no-include-email)

# Build the docker image locally with the image name and then push it to ECR
# with the full name.

# On a SageMaker Notebook Instance, the docker daemon may need to be restarted in order
# to detect your network configuration correctly.  (This is a known issue.)
if [ -d "/home/ec2-user/SageMaker" ]; then
  sudo service docker restart
fi

docker build  -t ${algorithm_name} .
docker tag ${algorithm_name} ${fullname}

docker push ${fullname}
```

## Step 2: Setting Up the Environment

Once we've created the container, we'll need to set up the environment. The cell below contains more boilerplate code, which is used to handle a couple sticking points in order to set up the environment. 




```python
# S3 prefix
prefix = 'sagemaker-keras-text-classification'

# Define IAM role
import boto3
import re

import os
import numpy as np
import pandas as pd
from sagemaker import get_execution_role

role = get_execution_role()
```

## Step 3: Creating the Session

Now that we've created the container and set up our environment, the next step is to create a SageMaker session. 


```python
import sagemaker as sage
from time import gmtime, strftime

sess = sage.Session()
```

## Step 4: Upload the Data for Training

Steps 4 and 5 are where you'll add the code unique to your project. In this step, you first upload your own data, and then make 


```python
WORK_DIRECTORY = 'data'

data_location = sess.upload_data(WORK_DIRECTORY, key_prefix=prefix)
```

## Step 5: Fitting the Model

# Step 6: Deploying the Model


```python
from sagemaker.predictor import json_serializer
predictor = tree.deploy(1, 'ml.m5.xlarge', serializer=json_serializer)
```

# Step 7: Cleanup  (IMPORTANT!)


```python
sess.delete_endpoint(predictor.endpoint)
```
