# Serverless Content Delivery (aka Livin' on the Edge)

The lines between static and dynamic content are blurred, and it’s more difficult than ever to choose the right technologies for your requirements and budget. [Symphonia's](https://www.symphonia.io) [John Chapin](https://twitter.com/johnchapin) takes you on a step-by-step journey from hosting static content on AWS S3 to deploying dynamic, complex business logic mere milliseconds away from your users with AWS CloudFront, Lambda@Edge, and more. Along the way, John explores complementary features such as custom DNS, SSL encryption, and metrics and logging.

In addition to those technologies, John addresses operational concerns like deployment and updates, cost control (especially given the scalability of the technology involved), and security from both an application and an AWS perspective. John concludes by detailing some of the idiosyncrasies of the approach, including the inherent limitations of the Lambda@Edge execution environment. You’ll leave with functioning, parameterized infrastructure templates to build your own serverless content stack.

While the examples will be primarily oriented around AWS, the concepts are applicable to other cloud platforms.

## Phase 1 - S3

Create an S3 bucket and bucket policy in your AWS account.

```
$ cd phase1
$ aws cloudformation create-stack \
    --template-body file://cfn.yml \
    --stack-name serverless-content
```

Get the bucket name.

```
$ export BUCKET=$(aws cloudformation describe-stack-resource \
    --stack-name serverless-content \
    --logical-resource-id Bucket \
    --query 'StackResourceDetail.PhysicalResourceId' \
    --output text)
```

Visit the bucket URL via a web browser, observe an HTTP 404 error response.

```
$ open "http://${BUCKET}.s3-website-us-east-1.amazonaws.com"
```

Upload some content to the bucket

```
$ aws s3 sync content s3://${BUCKET}
```

Visit the bucket URL via a web browser, observe the index page.

```
$ open "http://${BUCKET}.s3-website-us-east-1.amazonaws.com"
```

$ Observe the latency

```
$ curl -o /dev/null -s -w "%{time_total}\n" "http://${BUCKET}.s3-website-us-east-1.amazonaws.com"
```

## Phase 2 - CloudFront

Update the CloudFormation stack

```
$ cd phase2
$ aws cloudformation update-stack \
    --template-body file://cfn.yml \
    --stack-name serverless-content
```

Get the CloudFront distribution id and domain name.

```
$ export CLOUDFRONT_ID=$(aws cloudformation describe-stack-resource \
    --stack-name serverless-content \
    --logical-resource-id CloudFrontDistribution \
    --query 'StackResourceDetail.PhysicalResourceId' \
    --output text)

$ export CLOUDFRONT_DOMAIN=$(aws cloudfront get-distribution \
    --id ${CLOUDFRONT_ID} \
    --query 'Distribution.DomainName' \
    --output text)
```

Visit the domain.

```
open "http://${CLOUDFRONT_DOMAIN}"
```

Check out the latency.

```
curl -o /dev/null -s -w "%{time_total}\n" "http://${CLOUDFRONT_DOMAIN}"
```

## Phase 3 - Route 53

*Note that you must have a pre-existing Hosted Zone for this phase*

Update the CloudFormation stack

```
$ cd phase3
$ aws cloudformation update-stack \
    --template-body file://cfn.yml \
    --stack-name serverless-content \
    --parameters ParameterKey=DomainName,ParameterValue=hello.serverless-content.symphonia.io \
                ParameterKey=HostedZoneName,ParameterValue=serverless-content.symphonia.io.
```

Open the site using the domain name

```
$ open "http://hello.serverless-content.symphonia.io"
```

## Phase 4 - SSL

Update the CloudFormation stack

```
$ cd phase4
$ aws cloudformation update-stack \
    --template-body file://cfn.yml \
    --stack-name serverless-content \
    --parameters ParameterKey=DomainName,ParameterValue=hello.serverless-content.symphonia.io \
                ParameterKey=HostedZoneName,ParameterValue=serverless-content.symphonia.io. \
                ParameterKey=ValidationDomain,ParameterValue=symphonia.io
```

Validate the certificate (via email, or DNS)

Open the site using the HTTPS protocol

```
$ open "https://hello.serverless-content.symphonia.io"
```

## Phase 5 - Lambda @ Edge

Deploy the SAM bootstrap stack

```
$ cd phase5
$ aws cloudformation create-stack \
    --stack-name sam-bootstrap \
    --template-body file://sam-bootstrap-cfn.yml
```

Get the S3 bucket name

```
$ export SAM_BUCKET=$(aws cloudformation describe-stack-resource \
    --stack-name sam-bootstrap \
    --logical-resource-id Bucket \
    --query 'StackResourceDetail.PhysicalResourceId' \
    --output text)
```

Deploy CloudFormation stack using SAM package/deploy.

Note the missing `file://` syntax. The `package` and `deploy` commands are for the Serverless Application Model, and have some slight differences from the normal CloudFormation commands.

```
$ aws cloudformation package \
    --s3-bucket ${SAM_BUCKET} \
    --template-file cfn.yml \
    --output-template-file cfn-packaged.yml
```

Note that by using `deploy`, we don't have to specify unchanged parameter values.

```
$ aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --template-file cfn-packaged.yml \
    --stack-name serverless-content
```

Upload the new "secure" content

```
$ aws s3 sync content s3://${BUCKET}
```

Try to load the secret file using the S3 bucket URL:

```
$ open "http://${BUCKET}.s3-website-us-east-1.amazonaws.com/secure/secret.html"
```

Visit the secure area of the site:

```
$ open "https://hello.serverless-content.symphonia.io/secure/secret.html"
```