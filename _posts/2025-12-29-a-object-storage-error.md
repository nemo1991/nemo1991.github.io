---
title: a object storage error
---

my application occour one error. 

use ibm cos. my sdk use amazone.s3 .net 

```
Amazon.S3.AmazonS3Exception: The difference between the request time and the server's time is too large. ---> Amazon.Runtime.Internal.HttpErrorResponseException: 远程服务器返回错误: (403) 已禁止。 ---> System.Net.WebException: 远程服务器返回错误: (403) 已禁止。
```

i guess the server time had a 15min difference. but is not .


```
The reason for this problem is that Amazon S3 allows only a small time stamp variation of up to 15 minutes between the server and its requesting client (user pc). Since Amazon is a big backup server of large number of users, security does matter a lot.
造成此问题的原因是，Amazon S3 只允许服务器与其请求客户端（用户电脑）之间的时间戳存在不超过 15 分钟的偏差。由于 Amazon 是一个服务于大量用户的大型备份服务器，因此安全性至关重要。
```
https://github.com/aws/aws-sdk-java-v2/issues/4122#issuecomment-1687098058
https://docs.aws.amazon.com/pdfs/AmazonS3/latest/API/s3-api.pdf#ErrorResponses
https://docs.aws.amazon.com/zh_cn/filegateway/latest/files3/storagegateway-s3file-ug.pdf


https://web.archive.org/web/20170606231417/http://www.bucketexplorer.com/documentation/amazon-s3--difference-between-requesttime-currenttime-too-large.html
