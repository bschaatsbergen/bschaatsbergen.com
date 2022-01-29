---
title: "Delete old SageMaker resources"
date: 2022-01-15T10:10:15+00:00
tags: ["AWS", "Machine Learning"]
draft: true
---

Imagine creating hundreds of SageMaker resources a day (models, pipelines, etc). You'll quickly end up with a very polluted environment where searching for a specific resource quickly turns into a tedious thing to do.

After running into this problem several times a day it became essential that we sould tidy up. We decided to create a Lambda function that takes care of deleting the SageMaker models using boto3, invoked through a scheduled EventBridge rule. It's pretty straight forward, see below.

![EventBridge and Lambda](/delete-sagemaker-models.png)

To give you an idea, lets say we would just focus on deleting models from time to time (we obviously don't want to delete any recent registered models).

As with anything access related, stick to the principle of least privileged. Meaning, the Lambda function only requires the `sagemaker:ListModels` and `sagemaker:DeleteModel` actions to be attached to the associated execution role policy.

```json
{
  "Effect": "Allow",
  "Action": [
    "sagemaker:ListModels", 
    "sagemaker:DeleteModel"
    ],
  "Resource": "*"
}
```

Assuming you've setup a execution role and a policy with the above statement we can now jump into the Lambda function. Btw, for tasks like this I prefer to use Python Lambda functions as the Python SDK (boto3) is part of the Python runtime in Lambda.

The below code is the `lambda_function.py`. It uses a paginator with a `CreationTimeBefore` parameter to iterate through the models and adds them to a list, we then loop through the list and delete the models.

```python
"""
AWS Lambda function to delete SageMaker models after X amount of days.
"""

import logging
import os

import boto3
from botocore.exceptions import ClientError
from datetime import datetime, timedelta


logger = logging.getLogger()
logger.setLevel(logging.INFO)

globalVars = {}
globalVars["DAYS_TO_SUBSTRACT"] = os.getenv("DAYS_TO_SUBSTRACT")

client = boto3.client("sagemaker")

days_to_subtract = int(globalVars["DAYS_TO_SUBSTRACT"])

# Calculate the minimal deletion date
deletion_date = datetime.now().astimezone() - timedelta(days=days_to_subtract)


def lambda_handler(event, context):
    model_names = []

    print("Deleting models before: {}".format(deletion_date))

    for key in paginate(client.list_models):
        model_names.append(key["ModelName"])

    # Initiate the deletion of the models
    delete_models(model_names)


# Delete any model passed in the list
def delete_models(model_names):
    for model_name in model_names:
        print("Deleting model: {}".format(model_name))
        client.delete_model(ModelName=model_name)


# Using a paginator to iterate through all the pages,
# note the: `CreationTimeBefore` parameter.
def paginate(method, **kwargs):
    client = method.__self__
    paginator = client.get_paginator(method.__name__)
    for page in paginator.paginate(
        CreationTimeBefore=deletion_date,
        SortOrder="Ascending",
    ).result_key_iters():
        for result in page:
            yield result
```

If you run into rate limit issues you can pass a `PaginationConfig` to the paginator (be sure to tweak it to your needs):

```python
PaginationConfig={
    "MaxItems": 100, 
    "PageSize": 50,
},
```

As mentioned earlier, if you want to delete model package groups or pipelines you can easily do so by tweaking the above code.