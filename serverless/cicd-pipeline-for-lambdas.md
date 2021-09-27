# Setting up a CI/CD Pipeline for a Java Lambda using AWS CodePipeline, CodeBuild, SAM, and CloudFormation

1. Create a GitHub Repository to host your Lambda code.

2. Generate some boilerplate code that sets up your Lamba's logic (simple response, no other service integrations at this point)

3. Add a template file for AWS SAM that describes how your Lambda should be configured
Example:
```
AWSTemplateFormatVersion: '2010-09-09'
Transform: 'AWS::Serverless-2016-10-31'
Description: A simple AWS Lambda for searching book records within a DynamoDB table.
Resources:
  <NAME_FOR_YOUR_RESOURCE>:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: <RELATIVE_PATH_TO_YOUR_JAR>
      Handler: <FULLY_QUALIFIED_PATH_TO_YOUR_HANDLER>
      Runtime: java8.al2
      Description: <DESCRIPTION_FOR_YOUR_RESOURCE>
      MemorySize: 256
      Timeout: 30
      Tracing: Active
      Policies:
      - arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:policy/<YOUR_POLICY>
```

4. Create an S3 bucket that will hold the generated CloudFormation template that AWS SAM creates (bucket can stay private)

5. Create an AWS CodeBuild project that will be used to build our function logic into a JAR and prepare a CloudFormation template
Example buildspec: 
```
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto8
  build:
    commands:
      - echo Build started on `date`
      - mvn package
      - sam package
        --template-file sam-template.yml
        --s3-bucket <YOUR_BUCKET>
        --output-template-file packaged-template.yml

artifacts:
  files:
    - packaged-template.yml
```	

6. Attach a policy to the CodeBuild service role that allows for it to communicate with the S3 bucket you made in Step 4.
Example policy:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucketMultipartUploads",
                "s3:ListBucketVersions",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::*/*",
                "arn:aws:s3:::<YOUR_BUCKET>"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:ListStorageLensConfigurations",
                "s3:ListAccessPointsForObjectLambda",
                "s3:ListAllMyBuckets",
                "s3:ListAccessPoints",
                "s3:ListJobs",
                "s3:ListMultiRegionAccessPoints"
            ],
            "Resource": "*"
        }
    ]
}
```

7. Create a CodePipeline that includes a Source stage (pulls from the repo you  made in Step 1) and a Build stage that uses the CodeBuild project you created in Step 5), skip the Deploy stage for now.

8. Let the pipeline run and confirm it builds successfully. Also check to see that an object was placed in the S3 bucket you made in Step 4.

9. Create a new IAM role that will be leveraged by CloudFormation to setup and deploy your Lambda function. Attach the following policy to the new role:
Policy:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "lambda:CreateFunction",
                "lambda:GetFunction",
                "lambda:DeleteFunction",
                "lambda:UpdateFunctionCode",
                "lambda:UpdateFunctionConfiguration",
                "lambda:ListTags",
                "lambda:TagResource",
                "lambda:UntagResource",
                "cloudformation:CreateChangeSet",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:PutRolePolicy",
                "iam:AttachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PassRole",
                "s3:GetObject"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

10. Edit the pipeline and add a stage called "Deploy", add an Action Group called "Deploy" with the following configurations:
- Action Provider: CloudFormation
- Input Artifacts: `BuildArtifact`
- Action mode: Create or update a stack
- Stack name: `<YOUR_STACK_NAME>` (doesn't have to be an existing one)
- Template:
  - Artifact name: `BuildArtifact`
  - File name: `packaged-template.yml`
  - Capabilities: `CAPABILITY_IAM` and `CAPABILITY_AUTO_EXPAND`

11. Save your changes and release change to trigger the pipeline.

12. Savor your victory!
