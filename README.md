# Advanced CloudFormation Pattern: Conditionalize like a PRO

AWS CloudFormation allows for the definition of *Condition* statements that evaluate to true or false at deploy time. 

Stackery's 1-button **"use existing resource"** option and automated management of **environment specific** configuration data in AWS Systems Parameter Store makes it easy to create conditional logic that results in the utilization of existing resources under certain circumstances OR the provisioning of new resources in other circumstances.

This readme and the template.yaml provide an example for us to learn from.

### Why is this important? Cloud-side development is about service composition

Rather than raw servers, cloud-native applications are composed of a collection of managed cloud services. Localhost has become a poor representation of the production environment, as it is impossible to replicate all of the functionality of AWS locally on a laptop.

Balancing simplicity with power is the name of the game. 

The Stackery approach to defining *Condition* statements combined with environment specific configuration parameters results in a powerful pattern for controlling the circumstances under which resources are provisioned depending on the environement the stack is being deployed into, such as a **test environment** versus a **production environment.**


### Example Use Case

An existing stack deployed to the "prod" environment includes an SNS publish/subscribe topic related to that USER create, update, and delete messages. 

Chris is tasked with creating a new service called MicroService25 that needs to:

  a) subscribe to a newly provisioned "chris" SNS topic when deployed to the "chris" environment

  b) subscribe to an existing "prod" SNS topic when deployed to prod 

### Condition Functions

*Conditions* statements defined by the developer make use of [condition functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html).

The condition functions Stackery utilizes are such as Fn::If, Fn::Equals, and Fn::Not

##### Fn::If is the same as !If
Returns one value if the specified condition evaluates to true and another value if the specified condition evaluates to false. 

##### Fn::Equals is the same as !Equals
Returns true if two values are equal or false if they aren't.

##### Fn::Not is the same as !Not
Returns true for a condition that evaluates to false or returns false for a condition that evaluates to true.
 

### Stackery Environments & Environment Parameters that support this example

Every Stackery environment includes a configuration store that allows for configuration values to be stored in JSON format. The values are automatically persisted in (AWS Systems Manager > Parameter store)[https://console.aws.amazon.com/systems-manager/parameters?region=us-east-1]

#### Confg value stored in "chris" environment

{
  "sns\_crud\_topic_arn": "false"
}

Gets persisted in parameter store as: 
  
  Name: chris/sns\_crud\_topic_arn

  Value: false


#### Confg value stored in "prod" environment

{
  "sns_crud_topic_arn": "arn:aws:sns:us-east-1:394493744367:stackery-186910833539367-topic60FA4D08",
}

Gets persisted in parameter store as: 

  Name: prod/sns\_crud\_topic_arn

  Value: false


### Now for some Fun! Let's breakdown the Infrastructure as Code that supports this example

The template can be found in github at this URL: https://github.com/stackery/demo-conditional-examples/blob/master/template.yaml

**On lines 4, 11, and 36 you'll see 3 resources defined in this template:**

  a) UserServicesTopic 
       Type:       AWS::SNS::Topic
       Condition:  UserServicesTopicCreateNewResource

+ Because this resource is setup with a condition, it's only created if the condition is true.

  b) MicroService25
       Type: AWS::Serverless::Function
       Events:
          Topic:
          Type: SNS
          Properties:
            Topic: !If
              - UserServicesTopicUseExistingResource
              - !Ref UserServicesTopicExistingResource
              - !Ref UserServicesTopic


  c) UserServicesTopicExistingResource
       Type: Custom::StackeryExistingResource
       Condition: UserServicesTopicUseExistingResource

+ Because this resource is setup with a condition, it's only created if the condition is true.

**On line 50 you'll see the following parameter that instructs CloudFormation to grab a value from the AWS Parameter Store at deploy time:

  a) EnvConfigsnscrudtopicarnAsString:
       Type: AWS::SSM::Parameter::Value<String>
       Default: /<EnvironmentName>/sns\_crud\_topic_arn

##Spoiler Alert! Here's the where the heart of the logic lies!

Let's hone in on each of the 2 Conditions defined in lines 53 -58

a) UserServicesTopicCreateNewResource: !Equals
     - 'false'
     - !Ref EnvConfigsnscrudtopicarnAsString

+ During a deploy to the "chris" environment, this function returns **true** because the value we saved in the in the config store ("false") is equal to the other value provided ('false').

+ During a deploy to the "prod" environment, this function returns **false** because the value we saved in the config store ("arn:aws:sns:us-east-1:394493744367:stackery-186910833539367-topic60FA4D08") is NOT equal to the other value provided ('false').
  
b) UserServicesTopicUseExistingResource: !Not
     - Condition: UserServicesTopicCreateNewResource

+ During a deploy to the "chris" environment, this function returns **false** because UserServicesTopicCreateNewResource returns ('true') in this environment. 

+ During a deploy to the "prod" environment, this function returns **true** because UserServicesTopicCreateNewResource returns ('false') in this environment..
  

#### Fn::If is the same as !If
Returns one value if the specified condition evaluates to true and another value if the specified condition evaluates to false. 

#### Fn::Equals is the same as !Equals
Returns true if two values are equal or false if they aren't.

#### Fn::Not is the same as !Not
Returns true for a condition that evaluates to false or returns false for a condition that evaluates to true.



