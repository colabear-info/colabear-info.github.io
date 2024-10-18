---
title: "Use AWS S3 + Lambda + Rekognition for image background removal"
last_modified_at: 2024-07-20
categories:
  - Blog
tags:
  - aws
  - backend
  - record
---




## Basic element 
### S3 

### Lambda 


### debug中用到的tricks们
1. 有的时候不太会写test case，那么就先按照正常的程序写，然后在logging中发现输入的json格式，然后把这个复制到test case中，再调试，举个例子
```json
{
    "Records": [
        {
            "eventVersion": "2.1",
            "eventSource": "aws:s3",
            "awsRegion": "us-east-1",
            "eventTime": "2024-07-19T21:08:12.592Z",
            "eventName": "ObjectCreated:Put",
            "userIdentity": {
                "principalId": "A1DDWAZWEDD54U"
            },
            "requestParameters": {
                "sourceIPAddress": "98.42.221.147"
            },
            "responseElements": {
                "x-amz-request-id": "83TEXSJG9RPKHDSD",
                "x-amz-id-2": "YsRZEClEz2Xjq/0jPsWZktYHgbpzDk4vEcZXj0UNjgSP7Rl/lv7asOzReWjWtsxJ3hrguJZvjmpTL8YXUrBRKI2Y9YTPCGX+3oRALR06tXA="
            },
            "s3": {
                "s3SchemaVersion": "1.0",
                "configurationId": "68e17aae-9888-4b3f-87ce-7e544cb14235",
                "bucket": {
                    "name": "original-images-bucket-ailog",
                    "ownerIdentity": {
                        "principalId": "A1DDWAZWEDD54U"
                    },
                    "arn": "arn:aws:s3:::original-images-bucket-ailog"
                },
                "object": {
                    "key": "ComfyUI+temp+%288%29.png",
                    "size": 759264,
                    "eTag": "8533c3473d3c69aefdd93e24ed9dfab5",
                    "sequencer": "00669AD5BC6C361646"
                }
            }
        }
    ]
}
```

## Problems I met
### Add layer & Import error
Things I tryied:
- install packages using `pip install -t` and upload the whole folder to lambda
- install packages using aws docker image and upload the whole folder to lambda
- setup different paths for the package, e.g. `python/lib/python3.8/site-packages` and `python/`, turns out the first one is correct.

[solution](https://stackoverflow.com/questions/57197283/aws-lambda-cannot-import-name-imaging-from-pil)

### AWS ClientError: An error occurred (InvalidS3ObjectException) when calling the DetectLabels operation: Unable to get object metadata from S3. Check object key, region and/or access permissions.
当然可以用中文交流，我很抱歉之前没有用中文回答您。让我们用中文来分析您的代码和遇到的问题。

根据您之前提到的错误信息（"InvalidS3ObjectException"），问题很可能出现在调用Rekognition的DetectLabels API时。让我们来看看可能的原因和解决方法：

1. S3对象存在性检查：
   在调用Rekognition之前，先检查S3对象是否存在且可访问。在定义`source_bucket`和`source_key`后添加以下代码：

   ```python
   try:
       s3.head_object(Bucket=source_bucket, Key=source_key)
       print(f"S3对象存在: s3://{source_bucket}/{source_key}")
   except ClientError as e:
       print(f"检查S3对象时出错: {e}")
       raise
   ```

2. Rekognition客户端区域：
   确保您的Rekognition客户端与S3桶在同一区域。修改Rekognition客户端创建代码：

   ```python
   rekognition = boto3.client('rekognition', region_name=os.environ.get('AWS_REGION'))
   ```

3. IAM权限：
   再次检查Lambda函数的执行角色是否有足够的权限访问S3和Rekognition。该角色至少需要以下权限：
   - s3:GetObject（源桶）
   - s3:PutObject（目标桶）
   - rekognition:DetectLabels

4. 错误处理：
   修改错误处理代码以捕获特定的Rekognition异常：

   ```python
   from botocore.exceptions import ClientError
   from boto3.exceptions import Boto3Error
   
   try:
       # ... (您现有的代码)
   except ClientError as ce:
       error_code = ce.response['Error']['Code']
       error_message = ce.response['Error']['Message']
       print(f"AWS ClientError: {error_code} - {error_message}")
       return {
           'statusCode': 500,
           'body': json.dumps(f'AWS ClientError: {error_code} - {error_message}')
       }
   except Boto3Error as be:
       print(f"Boto3Error: {str(be)}")
       return {
           'statusCode': 500,
           'body': json.dumps(f'Boto3Error: {str(be)}')
       }
   ```

5. 日志记录：
   在代码中添加更详细的日志，特别是在调用Rekognition API周围：

   ```python
   print(f"正在为 s3://{source_bucket}/{source_key} 调用Rekognition DetectLabels API")
   try:
       response = rekognition.detect_labels(
           Image={'S3Object': {'Bucket': source_bucket, 'Name': source_key}},
           MaxLabels=10
       )
       print(f"Rekognition API响应: {json.dumps(response)}")
   except ClientError as ce:
       print(f"Rekognition API ClientError: {ce}")
       raise
   ```

6. 环境变量：
   确保所有必要的环境变量在Lambda函数配置中正确设置，包括`OUTPUT_BUCKET`和`AWS_REGION`。

在进行这些更改后，重新部署您的Lambda函数并再次测试。这些额外的日志和错误处理应该能提供更多关于错误发生位置和性质的信息。

如果您仍然遇到问题，请再次运行函数并分享CloudWatch日志输出。这将有助于准确定位错误的位置和原因。

试了之后，还是有一点问题，于是又debug了一下，根据错误信息，问题似乎确实出在从s3获得metadata的时候。


S3对象键（Key）编码问题：
您的source_key包含URL编码的字符（如'+'和'%28'）。在访问S3对象时，这可能会导致问题。让我们先解码这个键：
```python
import urllib.parse

source_key = urllib.parse.unquote_plus(record['s3']['object']['key'])
print(f"Decoded Source key: {source_key}")
```

检查S3对象是否存在：
在调用Rekognition之前，先验证S3对象是否存在：
```python
    s3.head_object(Bucket=source_bucket, Key=source_key)
    print(f"S3对象存在: s3://{source_bucket}/{source_key}")
except ClientError as e:
    print(f"检查S3对象时出错: {e}")
    raise
```
改了之后就成功了。

## Little complain
As a new user to AWS, I didn't find perfect experience using AWS. The most problems I met are basic problems, such as import error of a common python package 'PIL'. After several hours spent on searching and debugging, I finally sort help from third-party layer solutions. I saw similar problems complained by people 3 years ago. 


