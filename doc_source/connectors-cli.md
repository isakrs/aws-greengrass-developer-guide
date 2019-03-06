# Getting Started with Greengrass Connectors \(CLI\)<a name="connectors-cli"></a>

This feature is available for AWS IoT Greengrass Core v1\.7 only\.

This tutorial shows how to use the AWS CLI to work with connectors\.

## <a name="w4aac27c39b8"></a>

Use connectors to accelerate your development life cycle\. Connectors are prebuilt, reusable modules that can make it easier to interact with services, protocols, and resources\. They can help you deploy business logic to Greengrass devices more quickly\. For more information, see [Integrate with Services and Protocols Using Greengrass Connectors](connectors.md)\.

In this tutorial, you configure and deploy the [ Twilio Notifications](twilio-notifications-connector.md) connector\. The connector receives Twilio message information as input data, and then triggers a Twilio text message\. The data flow is shown in following diagram\.

![\[Data flow from Lambda function to Twilio Notifications connector to Twilio.\]](http://docs.aws.amazon.com/greengrass/latest/developerguide/images/connectors/twilio-solution.png)

After you configure the connector, you create a Lambda function and a subscription\.
+ The function evaluates simulated data from a temperature sensor\. It conditionally publishes the Twilio message information to an MQTT topic\. This is the topic that the connector subscribes to\.
+ The subscription allows the function to publish to the topic and the connector to receive data from the topic\.

The Twilio Notifications connector requires a Twilio auth token to interact with the Twilio API\. The token is a text type secret created in AWS Secrets Manager and referenced from a group resource\. This enables AWS IoT Greengrass to create a local copy of the secret on the Greengrass core, where it is encrypted and made available to the connector\. For more information, see [Deploy Secrets to the AWS IoT Greengrass Core](secrets.md)\.

The tutorial contains the following high\-level steps:

1. [Create a Secrets Manager Secret](#connectors-cli-create-secret)

1. [Create a Resource Definition and Version](#connectors-cli-create-resource-definition)

1. [Create a Connector Definition and Version](#connectors-cli-create-connector-definition)

1. [Create a Lambda Function Deployment Package](#connectors-cli-create-deployment-package)

1. [Create a Lambda Function](#connectors-cli-create-function)

1. [Create a Function Definition and Version](#connectors-cli-create-function-definition)

1. [Create a Subscription Definition and Version](#connectors-cli-create-subscription-definition)

1. [Create a Group Version](#connectors-cli-create-group-version)

1. [Create a Deployment](#connectors-cli-create-deployment)

The tutorial should take about 30 minutes to complete\.

**Using the AWS IoT Greengrass API**

It's helpful to understand the following patterns when you work with Greengrass groups and group components \(for example, the connectors, functions, and resources in the group\)\.
+ At the top of the hierarchy, a component has a *definition* object that is a container for *version* objects\. In turn, a version is a container for the connectors, functions, or other component types\.
+ When you deploy to the Greengrass core, you deploy a specific group version\. A group version can contain one version of each type of component\. A core is required, but the others are included as needed\.
+ Versions are immutable, so you must create new versions when you want to make changes\. 

**Tip**  
If you receive an error when you run an AWS CLI command, add the `--debug` parameter and then rerun the command to get more information about the error\.

The AWS IoT Greengrass API lets you create multiple definitions for a component type\. For example, you can create a `FunctionDefinition` object every time that you create a `FunctionDefinitionVersion`, or you can add new versions to an existing definition\. This flexibility allows you to customize your version management system\.

## Prerequisites<a name="connectors-cli-prerequisites"></a>

To complete this tutorial, you need:

### <a name="w4aac27c39c26b6"></a>
+ A Greengrass group and a Greengrass core \(v1\.7\)\. To learn how to create a Greengrass group and core, see [Getting Started with AWS IoT Greengrass](gg-gs.md)\. The Getting Started tutorial also includes steps for installing the AWS IoT Greengrass core software\.
+  AWS IoT Greengrass must be configured to support local secrets, as described in [Secrets Requirements](secrets.md#secrets-reqs)\.
**Note**  
This includes allowing access to your Secrets Manager secrets\. If you're using the default Greengrass service role, Greengrass has permission to get the values of secrets with names that start with *greengrass\-*\.
+ A Twilio account SID, auth token, and Twilio\-enabled phone number\. After you create a Twilio project, these values are available on the project dashboard\.
**Note**  
You can use a Twilio trial account\. If you're using a trial account, you must add non\-Twilio recipient phone numbers to a list of verified phone numbers\. For more information, see [ How to Work with your Free Twilio Trial Account](https://www.twilio.com/docs/usage/tutorials/how-to-use-your-free-trial-account)\.
+ AWS CLI installed and configured on your computer\. For more information, see [Installing the AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/installing.html) and [Configuring the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) in the *AWS Command Line Interface User Guide*\.

   

  The examples in this tutorial are written for Linux and other Unix\-based systems\. If you're using Windows, see [Specifying Parameter Values for the AWS Command Line Interface](https://docs.aws.amazon.com/cli/latest/userguide/cli-using-param.html) to learn about differences in syntax\.

  If the command contains a JSON string, the tutorial provides an example that has the JSON on a single line\. On some systems, it might be easier to edit and run commands using this format\.

## Step 1: Create a Secrets Manager Secret<a name="connectors-cli-create-secret"></a>

In this step, you use the AWS Secrets Manager API to create a secret for your Twilio auth token\.

1. First, create the secret\.
   + Replace *twilio\-auth\-token* with your Twilio auth token\.

   ```
   aws secretsmanager create-secret --name greengrass-TwilioAuthToken --secret-string twilio-auth-token
   ```
**Note**  
By default, the Greengrass service role allows AWS IoT Greengrass to get the value of secrets with names that start with *greengrass\-*\. For more information, see [secrets requirements](secrets.md#secrets-reqs)\.

1. Copy the `ARN` of the secret from the output\. You use this to create the secret resource and to configure the Twilio Notifications connector\.

## Step 2: Create a Resource Definition and Version<a name="connectors-cli-create-resource-definition"></a>

In this step, you use the AWS IoT Greengrass API to create a secret resource for your Secrets Manager secret\.

1. Create a resource definition that includes an initial version\.
   + Replace *secret\-arn* with the `ARN` of the secret that you copied in the previous step\.

    

------
#### [ JSON Expanded ]

   ```
   aws greengrass create-resource-definition --name MyGreengrassResources --initial-version '{
       "Resources": [
           {
               "Id": "TwilioAuthToken",
               "Name": "MyTwilioAuthToken",
               "ResourceDataContainer": {
                   "SecretsManagerSecretResourceData": {
                       "ARN": "secret-arn"
                   }
               }
           }
       ]
   }'
   ```

------
#### [ JSON Single\-line ]

   ```
   aws greengrass create-resource-definition \
   --name MyGreengrassResources \
   --initial-version '{"Resources": [{"Id": "TwilioAuthToken", "Name": "MyTwilioAuthToken", "ResourceDataContainer": {"SecretsManagerSecretResourceData": {"ARN": "secret-arn"}}}]}'
   ```

------

1. Copy the `LatestVersionArn` of the resource definition from the output\. You use this value to add the resource definition version to the group version that you deploy to the core\.

## Step 3: Create a Connector Definition and Version<a name="connectors-cli-create-connector-definition"></a>

In this step, you configure parameters for the Twilio Notifications connector\.

1. Create a connector definition with an initial version\.
   + Replace *account\-sid* with your Twilio account SID\.
   + Replace *secret\-arn* with the `ARN` of your Secrets Manager secret\. The connector uses this to get the value of the local secret\.
   + Replace *phone\-number* with your Twilio\-enabled phone number\. Twilio uses this to initiate the text message\. This can be overridden in the input message payload\. Use the following format: `+19999999999`\.

    

------
#### [ JSON Expanded ]

   ```
   aws greengrass create-connector-definition --name MyGreengrassConnectors --initial-version '{
       "Connectors": [
           {
               "Id": "MyTwilioNotificationsConnector",
               "ConnectorArn": "arn:aws:greengrass:region::/connectors/TwilioNotifications/versions/1",
               "Parameters": {
                   "TWILIO_ACCOUNT_SID": "account-sid",
                   "TwilioAuthTokenSecretArn": "secret-arn",
                   "TwilioAuthTokenSecretArn-ResourceId": "TwilioAuthToken",
                   "DefaultFromPhoneNumber": "phone-number"
               }
           }
       ]
   }'
   ```

------
#### [ JSON Single\-line ]

   ```
   aws greengrass create-connector-definition \
   --name MyGreengrassConnectors \
   --initial-version '{"Connectors": [{"Id": "MyTwilioNotificationsConnector", "ConnectorArn": "arn:aws:greengrass:region::/connectors/TwilioNotifications/versions/1", "Parameters": {"TWILIO_ACCOUNT_SID": "account-sid", "TwilioAuthTokenSecretArn": "secret-arn", "TwilioAuthTokenSecretArn-ResourceId": "TwilioAuthToken", "DefaultFromPhoneNumber": "phone-number"}}]}'
   ```

------
**Note**  
`TwilioAuthToken` is the ID that you used in the previous step to create the secret resource\.

1. Copy the `LatestVersionArn` of the connector definition from the output\. You use this value to add the connector definition version to the group version that you deploy to the core\.

## Step 4: Create a Lambda Function Deployment Package<a name="connectors-cli-create-deployment-package"></a>

### <a name="w4aac27c39c34b4"></a>

To create a Lambda function, you must first create a Lambda function *deployment package* that contains the function code and dependencies\. Greengrass Lambda functions require the [AWS IoT Greengrass Core SDK](lambda-functions.md#lambda-sdks-core) for tasks such as communicating with MQTT messages in the core environment and accessing local secrets\. This tutorial creates a Python function, so you use the Python version of the SDK in the deployment package\.

1. <a name="download-ggc-sdk"></a>Download the AWS IoT Greengrass Core SDK Python 2\.7 version 1\.3\.0\. You can download the SDK from the **Software** page in the AWS IoT Core console or from the [AWS IoT Greengrass Core SDK](what-is-gg.md#gg-core-sdk-download) downloads\. This procedure uses the console\.

   1. In the [AWS IoT Core console](https://console.aws.amazon.com//iotv2/home), choose **Software**\.  
![\[The left pane of the AWS IoT Core console with Software highlighted.\]](http://docs.aws.amazon.com/greengrass/latest/developerguide/images/console-software.png)

   1. Under **SDKs**, for **AWS IoT Greengrass Core SDK**, choose **Configure download**\.  
![\[The SDKs section with Configure download highlighted.\]](http://docs.aws.amazon.com/greengrass/latest/developerguide/images/console-software-ggc-sdk.png)

   1. Choose **Python 2\.7 version 1\.3\.0**, and then choose **Download Greengrass Core SDK**\.  
![\[The AWS IoT Greengrass Core SDK page with Python 2.7 version 1.3.0 and Download Greengrass Core SDK highlighted.\]](http://docs.aws.amazon.com/greengrass/latest/developerguide/images/gg-get-started-016.png)

1. <a name="untar-sdk"></a>Unpack the `greengrass-core-python-sdk-1.3.0.tar.gz` file\.
**Note**  
For information about how to do this on different platforms, see [this step](create-lambda.md) in the Getting Started section\. For example, you might use the following `tar` command:  

   ```
   tar -xzf greengrass-core-python-sdk-1.3.0.tar.gz
   ```

1. <a name="open-sdk-folder"></a>Open the extracted aws\_greengrass\_core\_sdk/sdk directory\.

   ```
   cd aws_greengrass_core_sdk
   cd sdk
   ```

1. <a name="unzip-sdk"></a>Unzip `python_sdk_1_3_0.zip`\.

1. Save the following Python code function in a local file named `temp_monitor.py`\.

   ```
   from __future__ import print_function
   import greengrasssdk
   import json
   import random
   
   client = greengrasssdk.client('iot-data')
   
   # publish to the Twilio Notifications connector through the twilio/txt topic
   def function_handler(event, context):
       temp = event['temperature']
       
       # check the temperature
       # if greater than 30C, send a notification
       if temp > 30:
           data = build_request(event)
           client.publish(topic='twilio/txt', payload=json.dumps(data))
           print('published:' + str(data))
           
       print('temperature:' + str(temp))
       return
   
   # build the Twilio request from the input data
   def build_request(event):
       to_name = event['to_name']
       to_number = event['to_number']
       temp_report = 'temperature:' + str(event['temperature'])
   
       return {
           "request": {
               "recipient": {
                   "name": to_name,
                   "phone_number": to_number,
                   "message": temp_report
               }
           },
           "id": "request_" + str(random.randint(1,101))
       }
   ```

1. Zip the following items into a file named `temp_monitor_python.zip`\. When creating the ZIP file, include only the code and dependencies, not the containing folder\.
   + **temp\_monitor\.py**\. App logic\.
   + **greengrasssdk**\. Required library for Python Greengrass Lambda functions that publish MQTT messages\.

   This is your Lambda function deployment package\.

## Step 5: Create a Lambda Function<a name="connectors-cli-create-function"></a>

Now, create a Lambda function that uses the deployment package\.

1. First, create an IAM role so you can pass in the ARN when you create the function\.
**Note**  
AWS IoT Greengrass doesn't use this role because permissions for your Greengrass Lambda functions are specified in the Greengrass group role\. For this tutorial, you create an empty role, or alternatively, you can use an existing execution role\.

    

------
#### [ JSON Expanded ]

   ```
   aws iam create-role --role-name Lambda_empty --assume-role-policy '{
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Principal": {
                   "Service": "lambda.amazonaws.com"
               },
              "Action": "sts:AssumeRole"
           }
       ]
   }'
   ```

------
#### [ JSON Single\-line ]

   ```
   aws iam create-role --role-name Lambda_empty --assume-role-policy '{"Version": "2012-10-17", "Statement": [{"Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"},"Action": "sts:AssumeRole"}]}'
   ```

------

1. Copy the `Arn` of the output\. 

1. Use the AWS Lambda API to create the TempMonitor function\. The following command assumes that the zip file is in the current directory\.
   + Replace *role\-arn* with the `Arn` that you copied\.

   ```
   aws lambda create-function \
   --function-name TempMonitor \
   --zip-file fileb://temp_monitor_python.zip \
   --role role-arn \
   --handler temp_monitor.function_handler \
   --runtime python2.7
   ```

1. Publish a version of the function\.

   ```
   aws lambda publish-version --function-name TempMonitor --description 'First version'
   ```

1. Create an alias for the published version\.

   Greengrass groups can reference a Lambda function by alias \(recommended\) or by version\. Using an alias makes it easier to manage code updates because you don't have to change your subscription table or group definition when the function code is updated\. Instead, you just point the alias to the new function version\.
**Note**  
AWS IoT Greengrass doesn't support Lambda aliases for **$LATEST** versions\.

   ```
   aws lambda create-alias --function-name TempMonitor --name GG_TempMonitor --function-version 1
   ```

1. Copy the `AliasArn` from the output\. You use this value when you configure the function for AWS IoT Greengrass and when you create a subscription\.

Now you're ready to configure the function for AWS IoT Greengrass\.

## Step 6: Create a Function Definition and Version<a name="connectors-cli-create-function-definition"></a>

To use a Lambda function on an AWS IoT Greengrass core, you create a function definition version that references the Lambda function by alias and defines the group\-level configuration\. For more information, see [Controlling Execution of Greengrass Lambda Functions by Using Group\-Specific Configuration](lambda-group-config.md)\.

1. Create a function definition that includes an initial version\.
   + Replace *alias\-arn* with the `AliasArn` that you copied when you created the alias\.

    

------
#### [ JSON Expanded ]

   ```
   aws greengrass create-function-definition --name MyGreengrassFunctions --initial-version '{
       "Functions": [
           {
               "Id": "TempMonitorFunction",
               "FunctionArn": "alias-arn",
               "FunctionConfiguration": {
                   "Executable": "temp_monitor.function_handler",
                   "MemorySize": 16000,
                   "Timeout": 5
               }
           }
       ]
   }'
   ```

------
#### [ JSON Single\-line ]

   ```
   aws greengrass create-function-definition \
   --name MyGreengrassFunctions \
   --initial-version '{"Functions": [{"Id": "TempMonitorFunction", "FunctionArn": "alias-arn", "FunctionConfiguration": {"Executable": "temp_monitor.function_handler", "MemorySize": 16000,"Timeout": 5}}]}'
   ```

------

1. Copy the `LatestVersionArn` from the output\. You use this value to add the function definition version to the group version that you deploy to the core\.

1. Copy the `Id` from the output\. You use this value later when you update the function\.

## Step 7: Create a Subscription Definition and Version<a name="connectors-cli-create-subscription-definition"></a>

<a name="connectors-how-to-add-subscriptions-p1"></a>In this step, you add a subscription that enables the Lambda function to send input data to the connector\. The connector defines the MQTT topics that it subscribes to, so this subscription uses one of the topics\. This is the same topic that the example function publishes to\.

<a name="connectors-how-to-add-subscriptions-p2"></a>For this tutorial, you also create subscriptions that allow the function to receive simulated temperature readings from AWS IoT and allow AWS IoT to receive status information from the connector\.

1. Create a subscription definition that contains an initial version that includes the subscriptions\.
   + Replace *alias\-arn* with the `AliasArn` that you copied when you created the alias for the function\. Use this ARN for both subscriptions that use it\.

    

------
#### [ JSON Expanded ]

   ```
   aws greengrass create-subscription-definition --initial-version '{
       "Subscriptions": [
           {
               "Id": "TriggerNotification",
               "Source": "alias-arn",
               "Subject": "twilio/txt",
               "Target": "arn:aws:greengrass:region::/connectors/TwilioNotifications/versions/1"
           },        
           {
               "Id": "TemperatureInput",
               "Source": "cloud",
               "Subject": "temperature/input",
               "Target": "alias-arn"
           },
           {
               "Id": "OutputStatus",
               "Source": "arn:aws:greengrass:region::/connectors/TwilioNotifications/versions/1",
               "Subject": "twilio/message/status",
               "Target": "cloud"
           }
       ]
   }'
   ```

------
#### [ JSON Single\-line ]

   ```
   aws greengrass create-subscription-definition \
   --initial-version '{"Subscriptions": [{"Id": "TriggerNotification", "Source": "alias-arn", "Subject": "twilio/txt", "Target": "arn:aws:greengrass:region::/connectors/TwilioNotifications/versions/1"},{"Id": "TemperatureInput", "Source": "cloud", "Subject": "temperature/input", "Target": "alias-arn"},{"Id": "OutputStatus", "Source": "arn:aws:greengrass:region::/connectors/TwilioNotifications/versions/1", "Subject": "twilio/message/status", "Target": "cloud"}]}'
   ```

------

1. Copy the `LatestVersionArn` from the output\. You use this value to add the subscription definition version to the group version that you deploy to the core\.

## Step 8: Create a Group Version<a name="connectors-cli-create-group-version"></a>

Now, you're ready to create a group version that contains all of the items that you want to deploy\. You do this by creating a group version that references the target version of each component type\.

First, get the group ID and the ARN of the core definition version\. These values are required to create the group version\.

1. Get the ID of the group:

   1. List your groups\.

      ```
      aws greengrass list-groups
      ```

   1. Copy the `Id` of the target group from the output\. You use this to get the core definition version and when you deploy the group\.

   1. Copy the `LatestVersion` of the group version from the output\. You use this to get the core definition version\.

1. Get the ARN of the core definition version:

   1. Get the group version\. For this step, we assume that the latest group version includes a core definition version\.
      + Replace *group\-id* with the `Id` that you copied for the group\.
      + Replace *group\-version\-id* with the `LatestVersion` that you copied for the group\.

      ```
      aws greengrass get-group-version \
      --group-id group-id \
      --group-version-id group-version-id
      ```

   1. Copy the `CoreDefinitionVersionArn` from the output\.

1. Create a group version\.
   + Replace *group\-id* with the `Id` that you copied for the group\.
   + Replace *core\-definition\-version\-arn* with the `CoreDefinitionVersionArn` that you copied for the core definition version\.
   + Replace *resource\-definition\-version\-arn* with the `LatestVersionArn` that you copied for the resource definition\.
   + Replace *connector\-definition\-version\-arn* with the `LatestVersionArn` that you copied for the connector definition\.
   + Replace *function\-definition\-version\-arn* with the `LatestVersionArn` that you copied for the function definition\.
   + Replace *subscription\-definition\-version\-arn* with the `LatestVersionArn` that you copied for the subscription definition\.

   ```
   aws greengrass create-group-version \
   --group-id group-id \
   --core-definition-version-arn core-definition-version-arn \
   --resource-definition-version-arn resource-definition-version-arn \
   --connector-definition-version-arn connector-definition-version-arn \
   --function-definition-version-arn function-definition-version-arn \
   --subscription-definition-version-arn subscription-definition-version-arn
   ```

1. Copy the value of `Version` from the output\. This is the ID of the group version\. You use this value to deploy the group version\.

## Step 9: Create a Deployment<a name="connectors-cli-create-deployment"></a>

Deploy the group to the core device\.

1. In a core device terminal, make sure that the AWS IoT Greengrass daemon is running\.

   1. To check whether the daemon is running:

      ```
      ps aux | grep -E 'greengrass.*daemon'
      ```

      If the output contains a `root` entry for `/greengrass/ggc/packages/1.7.1/bin/daemon`, then the daemon is running\.

   1. To start the daemon:

      ```
      cd /greengrass/ggc/core/
      sudo ./greengrassd start
      ```

1. Create a deployment\.
   + Replace *group\-id* with the `Id` that you copied for the group\.
   + Replace *group\-version\-id* with the `Version` that you copied for the group version\.

   ```
   aws greengrass create-deployment \
   --deployment-type NewDeployment \
   --group-id group-id \
   --group-version-id group-version-id
   ```

1. Copy the `DeploymentId` from the output\.

1. Get the deployment status\.
   + Replace *group\-id* with the `Id` that you copied for the group\.
   + Replace *deployment\-id* with the `DeploymentId` that you copied for the deployment\.

   ```
   aws greengrass get-deployment-status \
   --group-id group-id \
   --deployment-id deployment-id
   ```

   If the status is `Success`, the deployment was successful\. For troubleshooting help, see [Troubleshooting AWS IoT Greengrass](gg-troubleshooting.md)\.

## Test the Solution<a name="connectors-cli-test-solution"></a>

### <a name="w4aac27c39c46b4"></a>

1. <a name="choose-test-page"></a>On the AWS IoT Core console home page, choose **Test**\.  
![\[The left pane in the AWS IoT Core console with Test highlighted.\]](http://docs.aws.amazon.com/greengrass/latest/developerguide/images/console-test.png)

1. For **Subscriptions**, use the following values, and then choose **Subscribe to topic**\. The Twilio Notifications connector publishes status information to this topic\.    
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/greengrass/latest/developerguide/connectors-cli.html)

1. For **Publish**, use the following values, and then choose **Publish to topic** to invoke the function\.    
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/greengrass/latest/developerguide/connectors-cli.html)

   If successful, the recipient receives the text message and the console displays the `success` status from the [output data](twilio-notifications-connector.md#twilio-notifications-connector-data-output)\.

   Now, change the `temperature` in the input message to **29** and publish\. Because this is less than 30, the TempMonitor function doesn't trigger a Twilio message\.

## See Also<a name="connectors-cli-see-also"></a>
+ [Integrate with Services and Protocols Using Greengrass Connectors](connectors.md)
+  [AWS\-Provided Greengrass Connectors](connectors-list.md)
+ AWS Secrets Manager commands in the [https://docs.aws.amazon.com/cli/latest/reference/secretsmanager](https://docs.aws.amazon.com/cli/latest/reference/secretsmanager)
+ IAM commands in the [https://docs.aws.amazon.com/cli/latest/reference/iam](https://docs.aws.amazon.com/cli/latest/reference/iam)
+ AWS Lambda commands in the [https://docs.aws.amazon.com/cli/latest/reference/lambda](https://docs.aws.amazon.com/cli/latest/reference/lambda)
+ AWS IoT Greengrass commands in the [https://docs.aws.amazon.com/cli/latest/reference/greengrass](https://docs.aws.amazon.com/cli/latest/reference/greengrass)