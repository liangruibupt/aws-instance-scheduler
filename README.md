# AWS Instance Scheduler

## Description

Scheduler for Cross-Account and Cross-Region scheduling for EC2 and RDS instances
This repo is forked from https://github.com/awslabs/aws-instance-scheduler and made update for china region based on v1.3.0 version

## Setup

Deploys from Cloudformation template generate by makefile
1. You can git clone this repo
2. run make command, you can specify your {s3_bucket} and {region}
```
cd aws-instance-scheduler/source/code/
pip install pytz
pytz_location=$(pip show pytz | grep Location | cut -d':' -f 2 | tr -d " ")
cp -r ${pytz_location}/pytz .
# build
make bucket=$(bucket) solution=a$(solution) version=$(version) region=$(region)
# for example: make bucket=solutions-scheduler solution=aws-instance-scheduler version=v1.3.0 region=cn-northwest-1
# deploy
make deploy bucket=$(bucket) solution=a$(solution) version=$(version) region=$(region)
# for example: make deploy bucket=solutions-scheduler solution=aws-instance-scheduler version=v1.3.0 region=cn-northwest-1
rm -r pytz
```
3. After make successfully excuted:
- A new S3 bucket $(bucket)-$(region) will be automatically created if it is not existed
- The resources will be automatically uploaded to s3://$(bucket)-$(region)/$(solution)/$(version)/
- The S3 bucket public access block policy as: BlockPublicAcls=false,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

***

Copyright 2019 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at

    http://www.apache.org/licenses/

or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the License for the specific language governing permissions and limitations under the License.
