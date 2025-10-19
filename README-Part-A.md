<<<<<<< HEAD
# AWS in Action: API Gateway

The example in this repository reuses the example from chapter 10 in [Amazon Web Services in Action](http://bit.ly/amazon-web-services-in-action). You can find the code for the original example in the book's [code repository](https://github.com/AWSinAction/code/tree/master/chapter10).

Use [Swagger UI](http://petstore.swagger.io/?url=https://raw.githubusercontent.com/AWSinAction/apigateway/master/swagger.json) to have a look at the API definition.

## Setup

You have multiple options to setup the example:

1. Using the [Serverless Framework](http://serverless.com/)
2. Using CloudFormation
3. Using CloudFormation, [Swagger / OpenAPI Specification](https://openapis.org/specification) and the AWS CLI
4. Deprecated: ~~Using CloudFormation, [Swagger / OpenAPI Specification](https://openapis.org/specification) and the [Amazon API Gateway Importer](https://github.com/awslabs/aws-apigateway-importer)~~

### Using the Serverless Framework

clone this repository

```
$ git clone git@github.com:AWSinAction/apigateway.git
$ cd apigateway/
```

install the Serverless Framework

```
$ npm install -g serverless@1.0.0-rc.1
```

switch to the `serverless-framework` folder and install the dependencies

```
$ cd serverless-framework/
$ npm install --production
```

deploy the API

```
$ serverless deploy
[...]
Service Information
[...]
endpoints:
  GET - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/category/{category}/task
  PUT - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user/{userId}/task/{taskId}
  DELETE - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user/{userId}/task/{taskId}
  GET - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user/{userId}/task
  POST - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user/{userId}/task
  GET - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user/{userId}
  DELETE - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user/{userId}
  GET - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user
  POST - https://$ApiId.execute-api.us-east-1.amazonaws.com/dev/user
functions:
[...]
```

export the API endpoint from above in a environment variable to easily make calls the the API next.

```
export ApiGatewayEndpoint=" https://$ApiId.execute-api.us-east-1.amazonaws.com/dev"
```

and now [use the RESTful API](#use-the-restful-api).

### Using CloudFormation

clone this repository

```
$ git clone git@github.com:AWSinAction/apigateway.git
$ cd apigateway/
```

create the lambda code file (`lambda.zip`)

```
$ npm install --production
$ ./bundle.sh
```

create an S3 bucket in the US East (N. Virginia, `us-east-1`) region and upload the `lambda.zip` file (replace `$S3Bucket` with a S3 bucket name)

```
$ export AWS_DEFAULT_REGION=us-east-1
$ export S3Bucket=$(whoami)-apigateway
$ aws s3 mb s3://$S3Bucket
$ aws s3 cp lambda.zip s3://$S3Bucket/lambda.zip
```

create cloudformation stack (replace `$S3Bucket` with your S3 bucket name)

```
$ aws cloudformation create-stack --stack-name apigateway --template-body file://template_with_api.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,ParameterValue=$S3Bucket
```

wait until the stack is created (`CREATE_COMPLETE`)

```
$ aws cloudformation wait stack-create-complete --stack-name apigateway
```

get the `$ApiId`

```
$ aws cloudformation describe-stacks --stack-name apigateway --query Stacks[0].Outputs
```

set the `$ApiGatewayEndpoint` environment variable (replace `$ApiId`)

```
export ApiGatewayEndpoint="$ApiId.execute-api.us-east-1.amazonaws.com/v1"
```

and now [use the RESTful API](#use-the-restful-api).

### Using CloudFormation, Swagger / OpenAPI Specification and the AWS CLI

clone this repository

```
$ git clone git@github.com:AWSinAction/apigateway.git
$ cd apigateway/
```

create the lambda code file (`lambda.zip`)

```
$ npm install --production
$ ./bundle.sh
```

create an S3 bucket in the US East (N. Virginia, `us-east-1`) region and upload the `lambda.zip` file (replace `$S3Bucket` with a S3 bucket name)

```
export AWS_DEFAULT_REGION=us-east-1
export S3Bucket=$(whoami)-apigateway
$ aws s3 mb s3://$S3Bucket
$ aws s3 cp lambda.zip s3://$S3Bucket/lambda.zip
```

create cloudformation stack (replace `$S3Bucket` with your S3 bucket name)

```
$ aws cloudformation create-stack --stack-name apigateway --template-body file://template.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,ParameterValue=$S3Bucket
```

wait until the stack is created (`CREATE_COMPLETE`)

```
$ aws cloudformation wait stack-create-complete --stack-name apigateway
```

replace **all nine occurrences** of `$AWSRegion` in `swagger.json` with the region that you are creating your API and Lamdba in

```
$ sed -i '.bak' 's/$AWSRegion/us-east-1/g' swagger.json
```

get the `LambdaArn`

```
$ aws cloudformation describe-stacks --stack-name apigateway --query Stacks[0].Outputs
```

replace **all nine occurrences** of `$LambdaArn` in `swagger.json` with the ARN from the stack output above (e.g. `arn:aws:lambda:us-east-1:YYY:function:apigateway-Lambda-XXX`)

```
$ sed -i '.bak' 's/$LambdaArn/arn:aws:lambda:us-east-1:YYY:function:apigateway-Lambda-XXX/g' swagger.json
```

deploy the API Gateway

> make sure you have an up-to-date version (`aws --version`) of the AWS CLI >= 1.10.18. Learn more here: http://docs.aws.amazon.com/cli/latest/userguide/installing.html

```
$ aws apigateway import-rest-api --fail-on-warnings --body file://swagger.json
```

update the CloudFormation template to set the `ApiId` parameter (replace `$ApiId` with the `id` output from above)

```
$ aws cloudformation update-stack --stack-name apigateway --template-body file://template.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,UsePreviousValue=true ParameterKey=S3Key,UsePreviousValue=true ParameterKey=ApiId,ParameterValue=$ApiId
```

deploy to stage v1 (replace `$ApiId`)

```
$ aws apigateway create-deployment --rest-api-id $ApiId --stage-name v1
```

set the `$ApiGatewayEndpoint` environment variable (replace `$ApiId`)

```
export ApiGatewayEndpoint="$ApiId.execute-api.us-east-1.amazonaws.com/v1"
```

and now [use the RESTful API](#use-the-restful-api).

### Deprecated: ~~Using CloudFormation, Swagger / OpenAPI Specification and the Amazon API Gateway Importer~~

> This method is deprecated. Please choose one of the other methods mentioned earlier!

clone this repository

```
$ git clone git@github.com:AWSinAction/apigateway.git
$ cd apigateway/
```

create the lambda code file (`lambda.zip`)

```
$ npm install --production
$ ./bundle.sh
```

create an S3 bucket in the US East (N. Virginia, `us-east-1`) region and upload the `lambda.zip` file (replace `$S3Bucket` with a S3 bucket name)

```
export AWS_DEFAULT_REGION=us-east-1
export S3Bucket=$(whoami)-apigateway
$ aws s3 mb s3://$S3Bucket
$ aws s3 cp lambda.zip s3://$S3Bucket/lambda.zip
```

create cloudformation stack (replace `$S3Bucket` with your S3 bucket name)

```
$ aws cloudformation create-stack --stack-name apigateway --template-body file://template.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,ParameterValue=$S3Bucket
```

wait until the stack is created (`CREATE_COMPLETE`)

```
$ aws cloudformation wait stack-create-complete --stack-name apigateway
```

replace **all nine occurrences** of `$AWSRegion` in `swagger.json` with the region that you are creating your API and Lamdba in

```
$ sed -i '.bak' 's/$AWSRegion/us-east-1/g' swagger.json
```

get the `LambdaArn`

```
$ aws cloudformation describe-stacks --stack-name apigateway --query Stacks[0].Outputs
```

replace **all nine occurrences** of `$LambdaArn` in `swagger.json` with the ARN from the stack output above (e.g. `arn:aws:lambda:us-east-1:YYY:function:apigateway-Lambda-XXX`)

```
$ sed -i '.bak' 's/$LambdaArn/arn:aws:lambda:us-east-1:YYY:function:apigateway-Lambda-XXX/g' swagger.json
```

deploy the API Gateway

```
$ cd aws-apigateway-importer-master/
$ mvn assembly:assembly
$ ./aws-api-import.sh --create ../swagger.json
$ cd ..
```

using the CLI to see if it worked

```
$ aws apigateway get-rest-apis
```

update the CloudFormation template to set the `ApiId` parameter (replace `$ApiId` with the `id` output from above)

```
$ aws cloudformation update-stack --stack-name apigateway --template-body file://template.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,UsePreviousValue=true ParameterKey=S3Key,UsePreviousValue=true ParameterKey=ApiId,ParameterValue=$ApiId
```

deploy to stage (replace `$ApiId`)

```
$ cd aws-apigateway-importer-master/
$ ./aws-api-import.sh --update $ApiId --deploy stage ../swagger.json
$ cd ..
```

set the `$ApiGatewayEndpoint` environment variable (replace `$ApiId`)

```
export ApiGatewayEndpoint="$ApiId.execute-api.us-east-1.amazonaws.com/stage/v1"
```

and now [use the RESTful API](#use-the-restful-api).

## Use the RESTful API

the following examples assume that you replace `$ApiGatewayEndpoint` with `$ApiId.execute-api.us-east-1.amazonaws.com`

create a user

```
curl -vvv -X POST -d '{"email": "your@mail.com", "phone": "0123456789"}' -H "Content-Type: application/json" https://$ApiGatewayEndpoint/user
```

list users

```
curl -vvv -X GET https://$ApiGatewayEndpoint/user
```

create a task

```
curl -vvv -X POST -d '{"description": "test task"}' -H "Content-Type: application/json" https://$ApiGatewayEndpoint/user/$UserId/task
```

list tasks

```
curl -vvv -X GET https://$ApiGatewayEndpoint/user/$UserId/task
```

mark task as complete

```
curl -vvv -X PUT https://$ApiGatewayEndpoint/user/$UserId/task/$TaskId
```

delete task

```
curl -vvv -X DELETE https://$ApiGatewayEndpoint/user/$UserId/task/$TaskId
```

create a task with a category

```
curl -vvv -X POST -d '{"description": "test task", "category": "test"}' -H "Content-Type: application/json" https://$ApiGatewayEndpoint/user/$UserId/task
```

list tasks by category

```
curl -vvv -X GET https://$ApiGatewayEndpoint/category/$Category/task
```

## Teardown

### Using the Serverless Framework

remove the API

```
$ serverless remove
```

### Using CloudFormation

delete CloudFormation stack

```
$ aws cloudformation delete-stack --stack-name apigateway
```

delete S3 bucket (replace `$S3Bucket`)

```
$ aws s3 rb --force s3://$S3Bucket
```

### Using CloudFormation, Swagger / OpenAPI Specification and the AWS CLI

delete API Gateway (replace `$ApiId`)

```
$ aws apigateway delete-rest-api --rest-api-id $ApiId
```

delete CloudFormation stack

```
$ aws cloudformation delete-stack --stack-name apigateway
```

delete S3 bucket (replace `$S3Bucket`)

```
$ aws s3 rb --force s3://$S3Bucket
```


### Using CloudFormation, Swagger / OpenAPI Specification and the Amazon API Gateway Importer

delete API Gateway (replace `$ApiId`)

```
$ aws apigateway delete-rest-api --rest-api-id $ApiId
```

delete CloudFormation stack

```
$ aws cloudformation delete-stack --stack-name apigateway
```

delete S3 bucket (replace `$S3Bucket`)

```
$ aws s3 rb --force s3://$S3Bucket
```
=======
# CA1-Lab-CloudFormation-L00188348
1. Installation and Initial Configuration and Prerequisites
I installed theAWS CLIon my local machine (MacBook Pro).
I confirmed the installation with:

 aws --version // Reference: AWS CLI Official Docs


I configured my AWS credentials using:

	 aws configure
 AWS Access Key, Secret Key, region (us-east-1) and output format (json).


I tested with:

<img width="473" height="102" alt="Screenshot 2025-10-19 at 13 23 30" src="https://github.com/user-attachments/assets/a949b407-555c-4e41-bca1-6e13b49c057c" />

I created an environment variable for the S3 bucket:

 export S3Bucket=$(whoami)-apigateway
This createdromulo-apigatewayto store the Lambda code.


2. Cloning the Repository
I cloned the repository containing the API project:

 git clone git@github.com:AWSinAction/apigateway.git
cd apigateway
This folder contains the CloudFormation template(template_with_api.json), Lambda (lambda.js), and the function code (todo.js).
I confirmed the main files:
lambda.js→ main handler that calls the functions oftodo.js.
	todo.js→ User and task CRUD functions.
	template_with_api.json→ CloudFormation template to create Lambda, APIGateway e DynamoDB.

<img width="1064" height="78" alt="Screenshot 2025-10-19 at 13 26 49" src="https://github.com/user-attachments/assets/daa33748-5725-4337-982d-a24a16f3e4c7" />

3. Lambda Code Preparation
I installed Node.js dependencies (moment, uuid/node-uuid) and created the ZIP bundle:

 ./bundle.sh
Zip result:

 lambda.js
todo.js
config.json
I received notices thatnode_modules/momentandnode_modules/node-uuidwere not found, but the bundle was created correctly.
	I uploaded the ZIP to the S3 bucket:
	aws s3 cp lambda.zip s3://$S3Bucket/
I confirmed:

 aws s3 ls s3://romulo-apigateway/lambda.zip
Result:

 2025-10-14 17:09:12       1915 lambda.zip
4. Changes to the CloudFormation Template
Shelter template_with_api.jsonand I changed the names of the DynamoDB tables because there was a name conflict and I was stuck in a loop:
	Of todo-task → todo-task-romulo
	Of all-user → todo-user-romulo
I also adjusted the IAM Role permissions (LambdaRole) so that they point to the new tables:

 "Resource": [
    "arn:aws:dynamodb:...:table/todo-user-romulo*",
    "arn:aws:dynamodb:...:table/todo-task-romulo*"
]

5. Changes to the Codetodo.js
I've updated all functions to use the correct table names:
UserTable → "todo-user-romulo"
TaskTable → "todo-task-romulo"
I kept all the features:
CRUD for users:getUsers, postUser, getUser, deleteUser
CRUD for tasks:postTask, getTasks, putTask, deleteTask, getTasksByCategory
Pagination withLastEvaluatedKey
Data filters (due, overdue, futuredue, etc.)
Integration withAWS.DynamoDBLow-level API (S/N for attributes)


6. Creating the CloudFormation Stack
I deleted the previous stack if it existed:

 aws cloudformation delete-stack --stack-name MyApiStack
I created the stack with the updated template:

 aws cloudformation create-stack \
    --stack-name MyApiStack \
    --template-body file://template_with_api.json \
    --parameters ParameterKey=S3Bucket,ParameterValue=romulo-apigateway \
    --capabilities CAPABILITY_NAMED_IAM


I followed the stack status:

 aws cloudformation describe-stacks --stack-name MyApiStack

<img width="882" height="566" alt="Screenshot 2025-10-19 at 13 35 33" src="https://github.com/user-attachments/assets/d090e5ca-dff3-4e6d-8796-b1bf0edd23f6" />

I verified that the resources created correctly:


Bucket S3:romulo-apigateway/lambda.zip


DynamoDB Tables:todo-user-romuloandtodo-task-romulo


Lambda: configured withlambda.jsandtodo.js


API Gateway: ToDo API Gatewaywith all the resources and REST methods defined in the template


7. Completion of preparation
All stages ofprerequisitewere met: AWS CLI, S3 bucket, Lambda ZIP, CloudFormation template, DynamoDB tables created, CloudFormation stack created successfully.


Lambda Code (lambda.js + todo.js) ready to be tested.


API Gateway configured to call Lambda and access DynamoDB.


8. API testing and tuning
In my mac terminal:
aws apigateway get-rest-apis

<img width="1037" height="356" alt="Screenshot 2025-10-19 at 13 40 12" src="https://github.com/user-attachments/assets/af75d346-d30f-4ecc-9c06-07231e89e4a5" />

Then I tried to test the endpoint/user:
curl https://ulq0c01moi.execute-api.us-east-1.amazonaws.com/prod/user
and the return was an error: {"message": "Forbidden"}
Romulos-MacBook-Pro:apigateway romulo$ curl https://ulq0c01moi.execute-api.us-east-1.amazonaws.com/prod/user
{"message":"Forbidden"}Romulos-MacBook-Pro:apigateway romulo$ 
After much testing, believing it was a permissions issue or that I'd executed something incorrectly, I remembered that I'd faced some issues before because my MAC is old. So I started checking to see if I had all the requirements and realized I didn't have NODE. I ran:

<img width="835" height="276" alt="Screenshot 2025-10-19 at 13 47 03" src="https://github.com/user-attachments/assets/f538dbc4-cb0d-489c-87e8-90cf9129082e" />

and reloaded the shell:
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

After that, the packagelambda.ziphas been repackaged including the foldernode_modules.
I uploaded the new onelambda.zipto the S3 bucket and updated the Lambda function code using the command:
aws lambda update-function-code \

<img width="888" height="760" alt="Screenshot 2025-10-19 at 13 52 19" src="https://github.com/user-attachments/assets/fd1dc2b8-18b6-4b55-a64a-c1005b3c9416" />

The function was successfully redeployed and now correctly imports modules.momentanduuid.
Testes via API Gateway – Endpoint /user
I performed direct integration tests viacurlon the public endpoint exposed by API Gateway.
User creation:
curl -X POST https://ulq0c01moi.execute-api.us-east-1.amazonaws.com/v1/user \
  -H "Content-Type: application/json" \
  -d '{"email": "teste@exemplo.com", "phone": "123456789"}'
  
<img width="845" height="157" alt="Screenshot 2025-10-19 at 13 57 53" src="https://github.com/user-attachments/assets/593358ff-2120-44a2-b351-2d398974ad33" />

Testes via API Gateway – Endpoint /user/{uid}/task
Creating a task associated with the user:
curl -X POST https://ulq0c01moi.execute-api.us-east-1.amazonaws.com/v1/user/3573f66b-feac-459a-bad8-eeacb564146c/task \
  -H "Content-Type: application/json" \
  -d '{"category": "work", "description": "Prepare AWS report"}'
  
  <img width="998" height="115" alt="Screenshot 2025-10-19 at 14 10 07" src="https://github.com/user-attachments/assets/eb1dd5a6-8811-445d-bf5c-a4c7849b7827" />
Task consultation:
curl https://ulq0c01moi.execute-api.us-east-1.amazonaws.com/v1/user/3573f66b-feac-459a-bad8-eeacb564146c/task

<img width="815" height="61" alt="Screenshot 2025-10-19 at 14 14 26" src="https://github.com/user-attachments/assets/6be26bbd-7400-4818-83d6-f92d72732880" />

The data was correctly stored in DynamoDB and retrieved via API.

8. References and Tutorials Used
AWS CLI – Installation and Configuration: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html


AWS S3 CLI – File Upload: https://docs.aws.amazon.com/AmazonS3/latest/userguide/upload-objects.html


AWS Lambda Deployment com S3: https://docs.aws.amazon.com/lambda/latest/dg/nodejs-package.html


CloudFormation – Criar Stack: https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-stack.html


Repository base tutorial: https://cloudonaut.io/create-a-serverless-restful-api-with-api-gateway-cloudformation-lambda-and-dynamodb/

>>>>>>> 806b4606066a1a45a8170344fba5c47b33e3d794
