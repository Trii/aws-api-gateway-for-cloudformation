#API Gateway for CloudFormation
API Gateway for CloudFormation is a set of Custom Resources that allows you to manage your API Gateway setup
with CloudFormation. It is deployed with CloudFormation and run on AWS Lambda.

The project is inspired by [AWS Labs API Gateway Swagger Importer](https://github.com/awslabs/aws-apigateway-importer) so you will see a lot of familiar syntax in the setup.

![Build status](https://travis-ci.org/carlnordenfelt/aws-api-gateway-for-cloudformation.svg?branch=master)

##Contents
1. <a href="#a-note-on-terminology-before-we-begin">A note on terminology before we begin</a>
1. <a href="#installation">Installation</a>
    1. <a href="#uninstallation">Uninstallation</a>
1. <a href="#usage">Usage</a>
    1. <a href="#overview">Overview</a>
    1. <a href="#create-an-api">Create an API</a>
    1. <a href="#create-an-api-resource">Create an API Resource</a>
    1. <a href="#create-an-api-method">Create an API Method</a>
    1. <a href="#create-a-model">Create a Model</a>
    1. <a href="#create-a-domain-name">Create a Domain Name</a>
    1. <a href="#create-a-base-path-mapping">Create a Base Path Mapping</a>
1. <a href="#todo">TODO</a>

##A note on terminology before we begin
Throughout this document there are references to *resources* and *API Resources*.
It is very important to distinguish between the two:

* A *resource* is a CloudFormation term and can refer to any AWS resource.
* An *API Resource* is a specific type of AWS resource (Custom::ApiResource) which is called "Resource" in API Gateway.

Confusing, I know...

##Installation
##Prerequisites
* You'll need an [Amazon AWS Account](http://aws.amazon.com).
* You need an IAM user with access/secret key, see required permissions below.
* You have to install & configure the [AWS-CLI](http://docs.aws.amazon.com/cli/latest/userguide)

###Setup IAM permissions
To be able to install the Custom Resource library you require a set of permissions.
Configure your IAM user with the following policy and make sure that you have configured your aws-cli with access and secret key.

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "cloudformation:CreateStack",
                    "cloudformation:DescribeStacks",
                    "dynamodb:CreateTable",
                    "dynamodb:DescribeTable",
                    "iam:CreateRole",
                    "iam:CreatePolicy",
                    "iam:AttachRolePolicy",
                    "iam:GetRole",
                    "iam:PassRole",
                    "lambda:CreateFunction",
                    "lambda:UpdateFunctionCode",
                    "lambda:GetFunctionConfiguration"
                ],
                "Resource": [
                    "*"
                ]
            }
        ]
    }

###Run the installation script
Run the following command from you shell:

    make install

The installation takes a couple of minutes after which you are ready to start using the new CloudFormation resources.
The script will output the Custom Resource Lambda function ARN.
Save this value, it is the value of the ServiceToken parameter that each Custom Resource requires in your CloudFormation templates. 

####Manual installation

The installation scripts does not work under Windows and may fail under Linux. Until this has been fixed you can still do the setup manually by following the steps below. 

1. Clone the repository. 
1. Run now install --production
1. Create a zip file containing /lib, /node_modules and package.json.
1. Login to the AWS Console and go to CloudFormation
1. Create a new stack using the template file: _scripts/install/ApiGatewayCloudFormation.template
1. Once the stack is created, note the LambdaArn under the Outputs tab, this is the service token for your templates.
1. Go to Lambda, find the function and update it with the zip package you created earlier. 
1. All done!

####Uninstallation
If you want to uninstall the setup you simply run make teardown.
After you've run this command you can run make install again to get a new environment.

**Note:** if you reinstall the setup you have to update the ServiceToken in your CloudFormation templates. 

#Usage   

##Overview
This setup allows you to manage the majority of the API Gateway releated resource. Below you'll find (hopefully) exhaistive documentation on how to use each resource type.

One thing that is not currently supported are API deployments. There is a bit of a catch-22 thing happening with the SDK and deployments where I can't create a stage without a deployment and I can create a deployment without a stage. I have this on the TODO list but for now you'll have to mange API deployment outside of this project.

Also note that some resources may be flagged as experimental which means that they haven't been tested thouroghly.

That said, on to what you can do.

##Create an API

Creates a new Api Gateway REST API
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createRestApi-property

**Note:** The Custom Resource does not support API cloning. This is intentional.

###Parameters
**name:**
Name of the REST API.

* Required: *yes*
* Type: String
* Update: No interruption

**description:**
Name of the REST API. 
An optional description of your REST API.

* Required: no
* Type: String
* Update: No interruption, partial. You may add/change this property but not remove it. 

###Outputs
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createRestApi-property
In addition the parent resource id is also returned in the response:

    Fn::GetAtt: ["MyRestApi", "parentResourceId"]

###CloudFormation example
    "MyRestApi" : {
        "Type" : "Custom::RestApi",
        "Properties" : {
            "ServiceToken": "{Lambda_Function_ARN}",
            "name": "My API",
            "description": "This is my API"
        }
    }

##Create an API Resource

Creates a new API Resource in an existing API
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createResource-property

**Note:** If you delete an API Resource from your CloudFormation template, all child resources are deleted by API Gateway.
This may create certain data inconsistencies between the actual API and what is believed to be setup from the Custom Resource perspective.
I therefor recommend that if you remove an API Resource from a tempalte, remove all child resources at the same time to ensure a consistent data model.

###Parameters
**restApiId:**
Reference to the REST API in which you want to create this API Resource.

* Required: *yes*
* Type: String
* Update: Not supported

**parentId** 
Id of the parent API Resource under which you want to place this API Resource.

* Required: *yes*
* Type: String
* Update: No interruption, partial. You may add/change this property but not remove it. 

**pathPart** 
The path suffix of this API Resource, appended to the full path of the parent API Resource.

* Required: *yes*
* Type: String
* Update: No interruption, partial. You may add/change this property but not remove it. 

**corsConfiguration**
If you supply cors configuration this API Resource will enable CORS requests.
For more information on CORS, please refer to http://www.w3.org/TR/cors/

* Required: no
* Type: Object
* Update: No interruption

**corsConfiguration.allowMethods**
A list of HTTP methods that allow CORS requests under this API Resource.
 
* Required: *yes*
* Type: String array
* Update: No interruption

**corsConfiguration.allowHeaders**
List of headers that the server allows the user-agent to send in requests.
If this property is not set it is defaulted to the headers that Amazon recommends: Content-Type,X-Amz-Date,Authorization,X-Api-Key
If it is set it will override the default headers and not include them. See *corsConfiguration.allowDefaultHeaders* for further details.
 
* Required: no
* Type: String array
* Update: No interruption

**corsConfiguration.allowDefaultHeaders**
If you set *corsConfiguration.allowHeaders* and still want to include the default set of headers you can set this property
to true and the default headers will be appended to the headers you specified in *corsConfiguration.allowHeaders*

* Required: no
* Type: Boolean
* Update: No interruption

**corsConfiguration.allowOrigin**
If you supply cors configuration this API Resource will enable CORS requests.

* Required: no, default is *
* Type: Object
* Update: No interruption

**corsConfiguration.exposeHeaders**
A list of headers that are exposed to the client in the response, if present.

* Required: no, default is none
* Type: String array
* Update: No interruption

**corsConfiguration.maxAge**
Max age in seconds that a pre-flight check should be cached on the client.
If not supplied, the time that pre-flight requests are stored is at the discretion of the user agent.

* Required: no
* Type: Integer
* Update: No interruption

**corsConfiguration.allowCredentials**
Sets the Access-Control-Allow-Credentials header to true if this configuration is set to true.

* Required: no, default value is false
* Type: Boolean
* Update: No interruption

###Outputs
See http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createResource-property

###CloudFormation example
    "MyApiResource" : {
        "Type" : "Custom::ApiResource",
        "Properties" : {
            "ServiceToken": "{Lambda_Function_ARN}",
            "restApiId": "xyz123",
            "parentId": "abc456",
            "pathPart": "test",
            "corsConfiguration":
                "allowMethods" ["GET", "POST"],
                "allowHeaders": ["x-my-header", "some-other-header"],
                "allowDefaultHeaders": true,
                "allowOrigin": "http://example.com"
                "exposeHeaders": ["some-header", "x-another-header"],
                "maxAge": 1800,
                "allowCredentials": true
             }
        }
    }

##Create an API Method

Creates a new Api Gateway Method including response, request and integration.
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#putMethod-property

###Parameters
**restApiId:**
Reference id to the Rest API.

* Required: *yes*
* Type: String
* Update: Not supported

**resourceId** 
Id of the API Resource on which the API Method should be applied.

* Required: *yes*
* Type: String
* Update: No interruption

**method**
Basic method configuration object.

* Required: *yes*
* Type: Object
* Update: Not available

**method.httpMethod**
The HTTP verb that this method adheres to.

* Required: *yes*
* Type: String
* Update: No interruption

**method.authorizationType**
API Method authorization type

* Required: no, default is NONE
* Type: String
* Update: No interruption

**method.apiKeyRequired**
Set to true if this API Method requires an API Key.

* Required: no, default is false
* Type: Boolean
* Update: No interruption

**method.requestModels**
Specifies the Model resources used for the request's content type. 
Request models are represented as a key/value map, with a content type as the key and a Model name as the value.

* Required: no
* Type: Map<String{content-type},String{model-name}>
* Update: No interruption

**method.parameters**
Represents requests parameters that are sent with the backend request. 
Request parameters are represented as a string array of parameter destinations. 
The destination must match the pattern {location}.{name}, where location is either querystring, 
path, or header. name must be a valid, unique parameter name.

* Required: no
* Type: String Array
* Update: No interruption

**integration**
Backend integration configuration
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#putIntegration-property

* Required: *yes*
* Type: Object, *required*
* Update: Not available

**integration.type**
Backend type

* Required: *yes*
* Type: String
* Update: No interruption

**integration.credentials**
AWS credentials used to invoke the (AWS) backend.

* Required: no
* Type: String
* Update: No interruption

**integration.cacheNamespace**
Integration input cache namespace

* Required: no
* Type: String
* Update: No interruption

**integration.cacheKeyParameters**
Integration input cache keys

* Required: no
* Type: String array
* Update: No interruption

**integration.httpMethod**
HTTP method of the backend integration request.

* Required: conditional, must be set if *integration.type* is not set to MOCK.
* Type: String
* Update: No Interruption

**integration.requestTemplates**
Specifies the templates used to transform the method request body. 
Request templates are represented as a key/value map, with a content-type as the key and a template as the value.

For simple mapping the template can be expressed as a JSON object.
More complex templates can be expressed as a string.

* Required: no
* Type: Map[String{content-type},String{template}] or Map[String{content-type},Object{template}] 
* Update: No interruption

**integration.requestParameters**: 
Represents request parameters that are sent with the backend request. 
Request parameters are represented as a key/value map, with a destination as the key and a source as the value. 
A source must match an existing method request parameter, or a static value. 
Static values must be enclosed with single quotes, and be pre-encoded based on their destination in the request. 
The destination must match the pattern integration.request.{location}.{name}, where location is either querystring, 
path, or header. name must be a valid, unique parameter name.

* Required: no
* Type: Map<String{destination},String{source}>
* Update: No interruption

**integration.uri**
URI if the backend service.

* Required: conditional, must be set if *integration.type* is not set to MOCK.
* Type: String
* Update: No interruption

**responses** 
Configurations for both the IntegrationResponses and the MethodResponses.
The key is the selection pattern used to map the repsonse to a status code.
There should be one selection pattern with the value "default" which acts as the default response.
The value is a response configuration object.

http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#putIntegrationResponse-property
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#putMethodResponse-property

* Required: no
* Type: Map{String,Object}
* Update: Not available 

**response.statusCode**
The HTTP Status Code that this response configuration maps to
Must be unique within the scope on each API Method definition.

* Required: *yes*
* Type: Integer
* Update: No interruptions

**response.headers** 
Map of headers where the key is the destination header name and value is the source, or static value of the header.
Static values are specified using enclosing single quotes, and backend response headers can be read 
using the pattern integration.response.header.{name}.
CORS headers should not be specified as they are added automatically.

* Required: no,
* Type Map{String,String}
* Update: No interruptions

**response.responseTemplates** 
Specifies the templates used to transform the integration response body. 
Response templates are represented as a key/value map, with a content-type as the key and a template as the value.
The template can be represented as a JSON string or a JSON object.
A template represented by an empty string is the equivalent of output pass-through.

**Note:** If no responseTemplates are provided a default pass-through template is created for application/json.

* Required: no
* Type: Map{String,String} or Map{String,Object}
* Update: No interruptions

**response.responseModels** 
Specifies the Model resources used for the response's content type. 
Response models are represented as a key/value map, with a content type as the key and a Model name as the value.

* Required: no
* Type: Map<String{content-type},String{model-name}>
* Update: No interruption

###CloudFormation example:
    "MyApiMethod" : {
        "Type" : "Custom::ApiGateway_ApiMethod",
        "Properties" : {
            "ServiceToken": "{Lambda_Function_ARN}",
            "restApiId": "q1w2e3r4t5y6",
            "resourceId": "Resource Id",
            "method": {
                "authorizationType": "NONE",
                "httpMethod": "GET",
                "apiKeyRequired": "true",
                "requestModels": {
                    "application/json": "MyModel",
                },
                "parameters": [
                    "querystring.sortBy",
                    "header.x-my-header",
                    "path.entityType"
                ]
            },
            "integration": {
                "type": "AWS",
                "credentials": "arn:aws:iam::0123456789:role/APIGateway_Invoke_Lambda_20150102",
                "cacheKeyParameters": [
                    "sortBy"
                ],
                "cacheNamespace": "MyCacheNamespaceKey",
                "httpMethod": "POST",
                "requestTemplates": {
                    "application/json: ""
                },
                "requestParameters": {
                    "someKey: "STRING"
                }
                "uri": "arn:aws:apigateway:eu-west-1:lambda:path/2015-03-31/functions/arn:aws:lambda:eu-west-1:0123456789:function:MyLambdaBackend-LambdaFunction-YZX123XYZ123/invocations"
            },
            "responses": {
                "default": {
                    "statusCode": "200",
                    "responseModels": {
                        "application/json": "MyResponseModelName",
                    },
                    "headers": {
                        "X-Custom-Header": "x-my-header"
                    }
                },
                ".*NotFound.*": {
                    "statusCode": "404",
                    "responseModels": {
                        "application/json": "MyResponseModelName",
                    }
                }
            }
        }
    }

Outputs:
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#putMethod-property

##Create a Model

Creates a new Api Gateway Model
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createModel-property

###Parameters
**restApiId:**
Reference id to the Rest API.

* Required: *yes*
* Type: String
* Update: Not supported

**name:**
Name of the model

* Required: *yes*
* Type: String
* Update: Not supported 

**contentType:**
The content-type for the model.

* Required: no, default is application/json
* Type: String
* Update: Not supported

**description:**
Description of the model.

* Required: no, default is an empty string
* Type: String
* Update: No interruption

**schema:**
The model schema. This can be represented either a JSON object or a valid JSON string. 

* Required: no, default is an empty object
* Type: String or object
* Update: No interruption

###Outputs
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createModel-property

###CloudFormation example
    "MyApiModel" : {
        "Type" : "Custom::ApiModel",
        "Properties" : {
            "ServiceToken": "{Lambda_Function_ARN}",
            "restApiId": "xyz123",
            "name": "myModel",
            "contentType": "application/json",
            "description": "This is my model",
            "schema": "..."
        }
    }

##Create a Domain Name

**This resource is experimental**

Creates a new Api Gateway Domain Name
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createDomainName-property

###Parameters
**domainName:**
The name of the DomainName resource.

* Required: *yes*
* Type: String
* Update: Not supported

**certificateName:**
The name of the certificate.

* Required: *yes*
* Type: String
* Update: Not supported

**certificateBody:**
The body of the server certificate provided by your certificate authority.

* Required: *yes*
* Type: String
* Update: Not supported

**certificatePrivateKey:**
Your certificate's private key.

* Required: *yes*
* Type: String
* Update: Not supported

**certificateChain:**
The intermediate certificates and optionally the root certificate, one after the other without any blank lines. 
If you include the root certificate, your certificate chain must start with intermediate certificates and end with 
the root certificate. Use the intermediate certificates that were provided by your certificate authority. 
Do not include any intermediaries that are not in the chain of trust path.

* Required: *yes*
* Type: String
* Update: Not supported

###Outputs
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createDomainName-property

###CloudFormation example
    "MyApiDomainName" : {
        "Type" : "Custom::ApiDomainName",
        "Properties" : {
            "ServiceToken": "{Lambda_Function_ARN}",
            "domainName": "example.com",
            "certificateName": "testCertificate",
            "certificateBody": "...",
            "certificatePrivateKey": "...",
            "certificateChain": "..."
        }
    }

##Create a Base Path Mapping

**This resource is experimental**

Creates a new Api Gateway Base Path Mapping
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createBasePathMapping-property

###Parameters
**domainName:**
The domain name of the BasePathMapping resource to create.

* Required: *yes*
* Type: String
* Update: Not supported

**restApiId:**
Reference to the REST API in which you want to create this API Base Path Mapping.

* Required: *yes*
* Type: String
* Update: Not supported

**basePath:**
The base path name that callers of the API must provide as part of the URL after the domain name. 
This value must be unique for all of the mappings across a single API. 
Exclude this if you do not want callers to specify a base path name after the domain name.

* Required: no
* Type: String
* Update: Not supported

**stage:**
The name of the API stage that you want to use for this mapping. 
Exclude this if you do not want callers to explicitly specify the stage name after any base path name.

* Required: no
* Type: String
* Update: No interruption

###Outputs
http://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/APIGateway.html#createBasePathMapping-property

###CloudFormation example
    "MyApiBasePathMapping" : {
        "Type" : "Custom::ApiBasePathMapping",
        "Properties" : {
            "ServiceToken": "{Lambda_Function_ARN}",
            "restApiId": "xyz123",
            "domainName": "example.com",
            "basePath": "test",
            "stage": "beta"
        }
    }

#TODO

* Enable deployment management
* Change deploy script so that it does not require a local file with reference to the Lambda ARN, thus enabling multi-region setups from one installation.
* Create stable release packages with release notes etc.
