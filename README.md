# AWS Instance Scheduler

English in Front and Chinese after

英文在前，中文在后

## Description

Scheduler for Cross-Account and Cross-Region scheduling for EC2 and RDS instances
This repo is forked from https://github.com/awslabs/aws-instance-scheduler and made update for china region based on v1.3.0 version

## 描述
用于EC2和RDS实例的跨帐户和跨区域调度的按计划启停调度。
此存储库来自 https://github.com/awslabs/aws-instance-scheduler 并基于v1.3.0版本在中国区域进行了更新

## Setup 部署
Deploys from Cloudformation template generate by makefile
通过Makefile生成的Cloudformation模板进行部署
1. You can git clone this repo 

   你可以git克隆这个仓库

2. run make command, you can specify your {s3_bucket} and {region} 

   运行make命令，您可以指定{s3_bucket}和{region}

   Make sure your machine have the zip command installed

   确保你的运行机器安装了zip

```bash
cd aws-instance-scheduler/source/code/
pip install pytz
pytz_location=$(pip show pytz | grep Location | cut -d':' -f 2 | tr -d " ")
cp -r ${pytz_location}/pytz .
# build
## define variable or specify the value of bucket, solution, version, region
make bucket=${bucket} solution=${solution} version=${version} region=${region}
## for example: make bucket=solutions-scheduler solution=aws-instance-scheduler version=v1.3.0 region=cn-northwest-1

# set s3 bucket PublicAccessBlock configuration, make sure you use upgrade your aws cli > 1.18
aws s3api put-public-access-block \
    --bucket ${bucket} \
    --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true" --region ${region}

# deploy
make deploy bucket=${bucket} solution=${solution} version=${version} region=${region}
# for example: make deploy bucket=solutions-scheduler solution=aws-instance-scheduler version=v1.3.0 region=cn-northwest-1
rm -r pytz
```

3. After make successfully excuted （make执行成功后的操作）
- A new S3 bucket ${bucket}-${region} will be automatically created if it is not existed
- 如果S3存储桶不存在，那么新的S3存储桶 ${bucket}-${region} 将会被创建
- The resources will be automatically uploaded to s3://${bucket}-${region}/${solution}/${version}/
- 资源将自动上传到 s3://${bucket}-${region}/${solution}/${version}/
- The S3 bucket public access block policy as: BlockPublicAcls=false,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
- S3存储桶公共访问阻止策略为：BlockPublicAcls = false，IgnorePublicAcls = true，BlockPublicPolicy = true，RestrictPublicBuckets = true

4. 参考下列文档部署 Cloudformation 和执行相关测试
[测试效果](Testing.md)

## Solution Overview
https://aws.amazon.com/solutions/instance-scheduler/

The AWS Instance Scheduler is a solution that automates the starting (when resources capacity is needed) and stopping
(resources that are not in use) of Amazon Elastic Compute Cloud (Amazon EC2) and Amazon Relational Database Service
(Amazon RDS) instances.
The Instance Scheduler leverages AWS resource tags and AWS Lambda to automatically stop and restart instances
across multiple AWS Regions and accounts on a customer-defined schedule.

## 方案概述
https://aws.amazon.com/solutions/instance-scheduler/

本解决方案，可自动启动和停止Amazon Elastic Compute Cloud（Amazon EC2）和Amazon Relational Database Service（Amazon RDS）实例。
实例计划程序利用AWS资源标签和AWS Lambda自动停止和重新启动实例
根据客户定义的时间计划执行，支持跨AWS中国多个区域和帐户。

![](resource/images/instance-scheduler-architecture.png)


***

Copyright 2019 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at

    http://www.apache.org/licenses/

or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the License for the specific language governing permissions and limitations under the License.
