---
tags: aws lambda
categories: aws lambda
author: Nicolas Mugnier
---

Lambda is a compute service that lets you run code without provisioning or managing servers.

 Official documentation can be found here : [https://docs.aws.amazon.com/lambda/index.html](https://docs.aws.amazon.com/lambda/index.html)

## Function

A function is a resource that you can invoke to run your code in Lambda. A function has code to process the events that you pass into the function or that other AWS services send to the function.

## Triggers

- Invoke (AWS Console, SDK, cli)
- From Services
  - API Gateway
  - S3
  - SQS / SNS
  - ...
- Step Functions

## Events

Events are the data that you pass into the function. They can be sent by other AWS services or by your own code.

### API Gateway Event

````json
{
  "resource": "/",
  "path": "/",
  "httpMethod": "GET",
  "requestContext": {
    "resourcePath": "/",
    "httpMethod": "GET",
    "path": "/Prod/",
  },
  "headers": {
    "accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9",
    "accept-encoding": "gzip, deflate, br",
    "Host": "70ixmpl4fl.execute-api.us-east-2.amazonaws.com",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36",
    "X-Amzn-Trace-Id": "Root=1-5e66d96f-7491f09xmpl79d18acf3d050",
  },
  "multiValueHeaders": {
    "accept": [
      "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9"
    ],
    "accept-encoding": [
      "gzip, deflate, br"
    ],
  },
  "queryStringParameters": null,
  "multiValueQueryStringParameters": null,
  "pathParameters": null,
  "stageVariables": null,
  "body": null,
  "isBase64Encoded": false
}
````

### SQS Event

```json
{
  "Records": [
    {
      "messageId": "059f36b4-87a3-44ab-83d2-661975830a7d",
    	"receiptHandle": "AQEBwJnKyrHigUMZj6rYigCgxlaS3SLy0a...",
    	"body": "Test message.",
    	"attributes": {
    	  "ApproximateReceiveCount": "1",
    		"SentTimestamp": "1545082649183",
    		"SenderId": "AIDAIENQZJOLO23YVJ4VO",
    		"ApproximateFirstReceiveTimestamp": "1545082649185"
    	},
    	"messageAttributes": {},
    	"md5OfBody": "e4e68fb7bd0e697a0ae8f1bb342846b3",
    	"eventSource": "aws:sqs",
    	"eventSourceARN": "arn:aws:sqs:us-east-2:123456789012:my-queue",
    	"awsRegion": "us-east-2"
    },
    {
      "messageId": "2e1424d4-f796-459a-8184-9c92662be6da",
    	"receiptHandle": "AQEBzWwaftRI0KuVm4tP+/7q1rGgNqicHq...",
    	"body": "Test message.",
    	"attributes": {
    	  "ApproximateReceiveCount": "1",
    		"SentTimestamp": "1545082650636",
    		"SenderId": "AIDAIENQZJOLO23YVJ4VO",
    		"ApproximateFirstReceiveTimestamp": "1545082650649"
    	},
    	"messageAttributes": {},
    	"md5OfBody": "e4e68fb7bd0e697a0ae8f1bb342846b3",
    	"eventSource": "aws:sqs",
    	"eventSourceARN": "arn:aws:sqs:us-east-2:123456789012:my-queue",
    	"awsRegion": "us-east-2"
    }
 ]
}
````

## Code

Code is uploaded into `/var/task` directory

## Layers

[Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) let you develop your function's dependencies independently, and can reduce storage usage when you use the same layer with multiple functions.

## Execution

- Default Execution Time : 3 sec (max 900 sec)
- Default Memory Size : 128 MB (max 10240 MB)
- Default [Ephemeral Storage](https://docs.aws.amazon.com/lambda/latest/dg/API_EphemeralStorage.html) /tmp : 512 MB (max 10240 MB)
- [Memory & CPU](https://docs.aws.amazon.com/lambda/latest/operatorguide/computing-power.html) : Adding more memory proportionally increases the amount of CPU
- Lifetime : 45 mins
- Stateless
- Cold Start & Warmup

![figure-1](https://docs.aws.amazon.com/lambda/latest/operatorguide/images/perf-optimize-figure-1.png)

![figure-2](https://docs.aws.amazon.com/lambda/latest/operatorguide/images/perf-optimize-figure-2.png)

![figure-5](https://docs.aws.amazon.com/lambda/latest/operatorguide/images/perf-optimize-figure-5.png)

![figure-6](https://docs.aws.amazon.com/lambda/latest/operatorguide/images/perf-optimize-figure-6.png)

## Runtimes

[Lambda runtimes](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html)

## Images & Containers

On invocation, lambda service pop a new [container](https://hub.docker.com/search?q=aws+lambda) based on the configured runtime

Based images are available on [AWS ECR](https://gallery.ecr.aws/) (Elastic Container Registry)

- [AWS Lambda base images for custom runtimes](https://gallery.ecr.aws/lambda/provided)
- [NodeJS](https://gallery.ecr.aws/lambda/nodejs)

## Native Runtimes

- NodeJS : [aws-lambda-nodejs](https://hub.docker.com/r/amazon/aws-lambda-nodejs)
- Python : [aws-lambda-python](https://hub.docker.com/r/amazon/aws-lambda-python)
- Ruby : [aws-lambda-ruby](https://hub.docker.com/r/amazon/aws-lambda-ruby)
- Java : [aws-lambda-java](https://hub.docker.com/r/amazon/aws-lambda-java)
- Go : [aws-lambda-go](https://hub.docker.com/r/amazon/aws-lambda-go)
- .Net : [aws-lambda-dotnet](https://hub.docker.com/r/amazon/aws-lambda-dotnet)
- Default : [aws-lambda-provided](https://hub.docker.com/r/amazon/aws-lambda-provided)

## Runtimes Lifecycles

![Overview-Full-Sequence](https://docs.aws.amazon.com/lambda/latest/dg/images/Overview-Full-Sequence.png)

## Lambda runtime API

![api-concept-diagram](https://docs.aws.amazon.com/lambda/latest/dg/images/logs-api-concept-diagram.png)

## Swagger

```yaml
# This document describes the AWS Lambda Custom Runtime API using the OpenAPI 3.0 specification.
#
# A note on error reporting:
#
# Runtimes are free to define the format of errors that are reported to the runtime API, however,
# in order to integrate with other AWS services, runtimes must report all errors using the
# standard AWS Lambda error format:
#
# Content-Type: application/vnd.aws.lambda.error+json:
# {
#     "errorMessage": "...",
#     "errorType": "...",
#     "stackTrace": [],
# }
#
# See '#/components/schemas/ErrorRequest'.
#
# Lambda's default behavior is to use Lambda-Runtime-Function-Error-Type header value to construct an error response
# when error payload is not provided or can not be read.

openapi: 3.0.0
info:
  title: AWS Lambda Runtime API
  description: AWS Lambda Runtime API is an HTTP API for implementing custom runtimes
  version: 1.0.3

servers:
  - url: /2018-06-01

paths:
  /runtime/init/error:
    post:
      summary: >
        Non-recoverable initialization error. Runtime should exit after reporting
        the error. Error will be served in response to the first invoke.
      parameters:
        - in: header
          name: Lambda-Runtime-Function-Error-Type
          schema:
            type: string
      requestBody:
        content:
          "*/*":
            schema: {}
      responses:
        "202":
          description: Accepted
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/StatusResponse"
        "403":
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "500":
          description: >
            Container error. Non-recoverable state. Runtime should exit promptly.

  /runtime/invocation/next:
    get:
      summary: >
        Runtime makes this HTTP request when it is ready to receive and process a
        new invoke.
      responses:
        "200":
          description: >
            This is an iterator-style blocking API call. Response contains
            event JSON document, specific to the invoking service.
          headers:
            Lambda-Runtime-Aws-Request-Id:
              description: AWS request ID associated with the request.
              schema:
                type: string
            Lambda-Runtime-Trace-Id:
              description: X-Ray tracing header.
              schema:
                type: string
            Lambda-Runtime-Client-Context:
              description: >
                Information about the client application and device when invoked
                through the AWS Mobile SDK.
              schema:
                type: string
            Lambda-Runtime-Cognito-Identity:
              description: >
                Information about the Amazon Cognito identity provider when invoked
                through the AWS Mobile SDK.
              schema:
                type: string
            Lambda-Runtime-Deadline-Ms:
              description: >
                Function execution deadline counted in milliseconds since the Unix epoch.
              schema:
                type: string
            Lambda-Runtime-Invoked-Function-Arn:
              description: >
                The ARN requested. This can be different in each invoke that
                executes the same version.
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/EventResponse"
        "403":
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "500":
          description: >
            Container error. Non-recoverable state. Runtime should exit promptly.

  /runtime/invocation/{AwsRequestId}/response:
    post:
      summary: Runtime makes this request in order to submit a response.
      parameters:
        - in: path
          name: AwsRequestId
          schema:
            type: string
          required: true
      requestBody:
        content:
          "*/*":
            schema: {}
      responses:
        "202":
          description: Accepted
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/StatusResponse"
        "400":
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "403":
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "413":
          description: Payload Too Large
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "500":
          description: >
            Container error. Non-recoverable state. Runtime should exit promptly.

  /runtime/invocation/{AwsRequestId}/error:
    post:
      summary: >
        Runtime makes this request in order to submit an error response. It can
        be either a function error, or a runtime error. Error will be served in
        response to the invoke.
      parameters:
        - in: path
          name: AwsRequestId
          schema:
            type: string
          required: true
        - in: header
          name: Lambda-Runtime-Function-Error-Type
          schema:
            type: string
      requestBody:
        content:
          "*/*":
            schema: {}
      responses:
        "202":
          description: Accepted
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/StatusResponse"
        "400":
          description: Bad Request
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "403":
          description: Forbidden
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ErrorResponse"
        "500":
          description: >
            Container error. Non-recoverable state. Runtime should exit promptly.

components:
  schemas:
    StatusResponse:
      type: object
      properties:
        status:
          type: string

    ErrorResponse:
      type: object
      properties:
        errorMessage:
          type: string
        errorType:
          type: string

    ErrorRequest:
      type: object
      properties:
        errorMessage:
          type: string
        errorType:
          type: string
        stackTrace:
          type: array
          items:
            type: string

    EventResponse:
      type: object
```

## Build our own image

- [Execution Environment](https://docs.aws.amazon.com/lambda/latest/operatorguide/execution-environments.html)
- [Environment](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- [Runtimes API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html)
- [Extensions API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-extensions-api.html)
- [Custom Build](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html#runtimes-custom-build)

## Create a custom runtime (PHP)

Bref runtimes

- bref/php-xx
- bref/php-xx-fpm
- bref/php-xx-console