---
title: "AWS CloudFront with Signed URL"
date: 2021-05-08T15:27:23+08:00
draft: false
categories:
- AWS
- Golang
tags:
- AWS
- S3
- CloudFront
- Golang
---

Amazon CloudFront is a fast content delivery network (CDN) service managed by AWS. It serves your contents across edge locations around the globe with high transfer speeds and low latency under secured connections.

In this post, we will set up an Amazon CloudFront distribution that serves private contents on your S3 bucket in order to speed up your content retrival while fully controlling user access permissions.

![](https://i.imgur.com/VVroqoy.png)
<!--more-->
## Setting up CloudFront
Create a CloudFront distribution with orgin being your S3 bucket `mybucket`:

![](https://i.imgur.com/e6TDyeC.png)
![](https://i.imgur.com/LyXoeo0.png)

Check your CloudFront distribution status [here](https://console.aws.amazon.com/cloudfront/home#distributions:). Wait until its status becomes `Deployed`. Then you will see the domain name of your CloudFront distribution. 

We enabled the CloudFront bucket access restriction so that clients cannot access your object using S3 URL but only CloudFront URLs. This restriction mechanism works by creating a CloudFront origin access identity and adding it to your S3 permission policy.

Whenever your users access your Amazon S3 objects through CloudFront, the CloudFront origin access identity retrieves the objects on behalf of your users. If your users request objects directly by using Amazon S3 URLs, he or she will be denied. The procedure can be summarized in the following figure:

![](https://i.imgur.com/uMDqXdz.png)
## Signed URL Creation Example
You can see the complete source code example on [my Github](https://github.com/minghsu0107/cloudFront-signed-url).

In this example, we will use [Golang AWS SDK](https://github.com/aws/aws-sdk-go) to upload `hello.txt` to S3 bucket `mybucket` with key `mysubpath/hello.txt`. Then, we will create its signed URL, which has a 1 hour expiration.
### Steps
Create a new client session:
```go
creds := credentials.NewStaticCredentials(accessKey, secretKey, "")

config := &aws.Config{
    Credentials: creds,
    Region:      aws.String(s3Region),
    MaxRetries:  aws.Int(3),
}
session, err := session.NewSession(config)
```
Upload `hello.txt` to the S3 bucket:
```go
fromFile, err := os.Open(uploadFrom)
if err != nil {
    exitErrorf("Unable to open file %q, %v", uploadFrom, err)
}
defer fromFile.Close()

uploader := s3manager.NewUploader(session)
var output *s3manager.UploadOutput
output, err = uploader.UploadWithContext(context.Background(), &s3manager.UploadInput{
    Bucket:  aws.String(s3Bucket),
    Key:     aws.String(objKey),
    Body:    fromFile,
})
```
Sign the object URL of `hello.txt` with your CloudFront key ID and private key:
```go
var priKeyFile *os.File
priKeyFile, err = os.Open(cfPrikeyPath)
if err != nil {
    exitErrorf("Unable to open file %q, %v", cfPrikeyPath, err)
}

var priKey *rsa.PrivateKey
priKey, err = sign.LoadPEMPrivKey(priKeyFile)
if err != nil {
    exitErrorf("err loading private key, %v", err)
}

var signedURL string
signer := sign.NewURLSigner(cfAccessKey, priKey)
signedURL, err = signer.Sign(output.Location, time.Now().Add(1*time.Hour))
if err != nil {
    exitErrorf("Failed to sign url, err: %v", err)
}
```
Finally, replace the S3 endpoint with your CloudFront endpoint:
```go
u, err := url.Parse(signedURL)
if err != nil {
    exitErrorf("Failed to parse signed url %v, err: %v", signedURL, err)
}
u.Host = cfDomain
signedURL = u.String()
fmt.Printf("Get signed URL %q\n", signedURL)
```
The object URL `https://my-s3-bucket.s3.us-east-2.amazonaws.com/mysubpath/hello.txt` will be signed, and the result will be printed in the standard output. Users can now access the object via this signed URL until it expires 1 hour later.
## Reference
- https://medium.com/@ratulbasak93/serving-private-content-of-s3-through-cloudfront-signed-url-593ede788d0d