---
layout: post
author: Keiran
title: Using AWS Lambda for Home Assistant notifications and automations
---

## Introduction
[Home Assistant](https://www.home-assistant.io/) has extensive support for a wide range of cloud providers and services. One of those is [AWS Lambda](https://www.home-assistant.io/integrations/aws/) - [AWS's Function as a service platform](https://aws.amazon.com/lambda/).

In this example, I'll walk through the process of defining a lambda as a notification target in home assistant that can then be triggered when certain things occur in the home. This assumes you have a base level of knowledge of AWS and Home Assistant. 

* [The Code for this example will soon be published here](https://notes.keiran.io)
* This document assumes you will be be using the ap-southeast-2 (Sydney) region, so you'll need to change it up if yours is different.

## Setting up and configuring your Lambda and AWS Resources

1.  Create an AWS Lambda for Home Assistant.

    In this case, it will be a Python 3 Lambda called hass_lambda_integration that takes a JSON encoded event and logs the payload to Cloudwatch Logs.


2.  Create an IAM Policy for the Lambda

    The IAM Policy for the Lambda governs the access it has into the AWS account and will vary based on the task you want the function to perform.

    In this example, the lambda will be given basic access to execute and store its logs in it's cloudwatch log group only.

    Ensure you adjust the IAM policy to meet your needs, and ensure you align to a least privledge approach at all times.


3.  Create an additional IAM Policy that allows the function to be invoked (executed) by Home Assistant
    This policy defines what action home assistant can do with the Lambda, in this example code, we only allow the Lambda to be invoked.


4.  Create an IAM User for Home Assistant
    We need a way for Home Assistant to access the function and AWS API's so, we'll create a user account called hass_user.

    With this user, we do two things:

      a. Associate the IAM Policy from step 3 with the user, granting access to the hass_user to invoke the hass_lambda_integration lambda

      b. Create a set of API keys for it that will be placed in the Home Assistant configuration, allowing it to invoke the Lambdas and pass data to them.

5.  We are now ready to configure home assistant, in the next step you will need:

    * The API Keys for the hass_user created above
    * The ARN of the Lambda function created above

## Setting up Home Assistant to trigger your Lambda

1.  In your Home Assistant Configuration file (configuration.yaml), add the following block of code and restart your Home Assistant Install

    ```
    aws:
      notify:
        - service: lambda
          region_name: ap-southeast-2
          name: hass_lambda_integrationn
          aws_access_key_id: <Redacted>
          aws_secret_access_key: <Redacted>
    ```

    When this is completed, under the developer section you will find you now have a new notify service called ```notify.hass_lambda_integration```

2.  Calling your new home assistant service and invoking your AWS function

    The integration works by calling ```notify.hass_lambda_integration``` with suitable Service Data, Home Assistant will then invoke the function, passing the data onto the function.
    
    The most basic invocation requires the following data value as it is a notification service
      * title - The message title
      * message - The message body of the message
      * target - The ARN of the Lambda function (Note i've obscred my account name)

    In YAML format, this looks below: 

    ```
    ---
    title: Hello from Home Assistant - This is the title
    message: Hello from Home Assistant - This is the message 
    target: arn:aws:lambda:ap-southeast-2:1234567890:function:hass_lambda_integration
    ```
    
    And the Lambda function receives this as the following event (as logged to cloudwatch logs from my sample function)

    ```
    [INFO]	2020-07-29T10:45:18.169Z	d23aac70-39a4-4256-84ed-25a187b93da8	The event was : 
    {
        "message": "Hello from Home Assistant - This is the message",
        "title": "Hello from Home Assistant - This is the title",
        "target": [
            "arn:aws:lambda:ap-southeast-2:1234567890:function:hass_lambda_integration"
        ]
    }
    ```

    If you would like to pass more data to the function, you can pass an optional data block of key value pairs from templates and other logic inside your Home Assistant install as per the below example.

    ```
    ---
    title: Hello from Home Assistant - This is the title
    message: Hello from Home Assistant - This is the message 
    target: arn:aws:lambda:ap-southeast-2:1234567890:function:hass_lambda_integration
    data:
      key1: value1
      key2: value2
      key3: value3
    ```

    As per the above, my example Lambda will log the following, showing you the format of the data you have access to.

    ```
    [INFO]	2020-07-29T10:48:34.788Z	78ead725-440c-4002-b165-8991965c3ce3	The event was : 
    {
        "message": "Hello from Home Assistant - This is the message",
        "title": "Hello from Home Assistant - This is the title",
        "target": [
            "arn:aws:lambda:ap-southeast-2:1234567890:function:hass_lambda_integration"
        ],
        "data": {
            "key1": "value1",
            "key2": "value2",
            "key3": "value3"
        }
    }
    ```

## So .. what can I do with this ?

This is quite a powerful integration, putting the capabilities of AWS at your fingertips from home assistant.

Some of the things I've been playing with are:

  * Leveraging Home Assistants presence detection automation trigger Lambdas when I leave / arrive my home to stop and start my AWS EC2 Instances and other lab resources or kick off a set of code build jobs so you have a fresh nightly build of software when you get to the office
  * Sending events into your own private AWS based notification system such as internal chat products, etc

I hope you find this helpful :)