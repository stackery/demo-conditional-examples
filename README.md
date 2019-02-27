# Advanced CloudFormation Pattern: Conditionalize like a PRO

AWS CloudFormation allows for the definition of __condition__ statements that evaluate to true or false at deploy time. 

Stackery's one-click __Use Existing Resource__ option and automated hierarchical management of **environment specific** configuration data makes it easy to create conditional logic that results in the utilization of existing resources under certain circumstances OR the provisioning of new resources in other circumstances.

This readme and the template.yaml provide an example for us to learn from.

### Why is this important? Simplicity is a prerequisite of reliability

Rather than raw servers, cloud-native applications are composed of a collection of managed cloud services. Localhost has become a poor representation of the production environment, as it is impossible to replicate all of the functionality of AWS locally on a laptop. As such, infrastructure-as-code (IaC) has become the best practice for configuring consistent application stacks across various environments. 

Without condition statements, many turn to creating a version of the template for each deploy target or including external config files during a step of the release process. While these techniques work, they introduce un-intended complexity, especially as the number of microservices and team members scale. 

The Stackery approach to defining __condition__ statements combined with environment specific configuration parameters results in a powerful pattern for controlling the circumstances under which resources are provisioned depending on the environement the stack is being deployed into, such as a __test environment__ versus a __production environment__.

A single IAC template per application stack is simpler and less brittle then multiple files.

The benefits of the pattern shown below are ease of understanding, ease of change, ease of debugging, and flexibility.

### Example Use Case

An existing stack deployed to the "prod" environment includes an SNS publish/subscribe topic related to that USER create, update, and delete messages. 

Chris is tasked with creating a new service called MicroService25 that needs to:

  a) subscribe to a newly provisioned "chris" SNS topic when deployed to the "chris" environment

  b) subscribe to an existing "prod" SNS topic when deployed to prod 


## Condition Functions

Condition statements defined by the developer make use of [condition functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html).

The condition functions Stackery utilizes are such as Fn::If, Fn::Equals, and Fn::Not

## Stackery Environments Parameters

Every Stackery environment includes a configuration store that allows for configuration values to be stored in JSON format. The values are automatically persisted in (AWS Systems Manager > Parameter store)[https://console.aws.amazon.com/systems-manager/parameters?region=us-east-1]

__Chris Environment__
```yaml
{
  "sns_crud_topic_arn": "false"
}
```

__Prod Environment__
```yaml
{
  "sns_crud_topic_arn": "arn:aws:sns:us-east-1:394493744367:stackery-186910833539367-topic60FA4D08"
}
```

## Key Contents of the IAC Template (automatically created by Stackery)

### Resource Section: 

1.  __UserServicesTopic__ (Line 4)

    ```yaml
      UserServicesTopic:
        Type: AWS::SNS::Topic
        Properties:
          TopicName: !Sub ${AWS::StackName}-UserServicesTopic
        Metadata:
          StackeryName: UserServicesTopic
        Condition: UserServicesTopicCreateNewResource
    ```
    Because this resource is setup with a condition, it's only created if the condition is true.

    During a deployment to the __Chris__ Environment, UserServicesTopicCreateNewResource will evaluate to true (see below) which means that this condition will result to true and therefore this resource will be provisioned.
    
    During a deployment to the __Prod__ Environment, UserServicesTopicCreateNewResource will evaluate to false (see below) which means that this condition will result to false and therefore this resource will NOT be provisioned.

    
2. __MicroService25__ (Line 11)

    ```yaml
      MicroService25:
        Type: AWS::Serverless::Function
        Properties:
          ...
          Events:
            Topic:
              Type: SNS
              Properties:
                Topic: !If
                  - UserServicesTopicUseExistingResource
                  - !Ref UserServicesTopicExistingResource
                  - !Ref UserServicesTopic
        Metadata:
          StackeryName: MicroService25
    ```


3. __UserServicesTopicExistingResource__ (Line 36)

    ```yaml
      UserServicesTopicExistingResource:
        Type: Custom::StackeryExistingResource
        Properties:
          ServiceToken: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:stackery-agent-commander
          Type: topic
          Data: !Ref EnvConfigsnscrudtopicarnAsString
        Condition: UserServicesTopicUseExistingResource
    ```
    Because this resource is setup with a condition, it's only created if the condition is true.
    
    During a deployment to the __Chris__ Environment, UserServicesTopicUseExistingResource will evaluate to false (see below) which means that this condition will result to true and therefore this resource will NOT be provisioned.
    
    During a deployment to the __Prod__ Environment, UserServicesTopicUseExistingResource will evaluate to true (see below) which means that this condition will result to true and therefore this resource will be provisioned.


### Parameters Section:

1. __EnvConfigsnscrudtopicarnAsString__ (Line 50)

    ```yaml
    EnvConfigsnscrudtopicarnAsString:
      Type: AWS::SSM::Parameter::Value<String>
      Default: /<EnvironmentName>/sns_crud_topic_arn
    ```
    This parameter instructs CloudFormation to check for, and use a value, stored in Stackery Environment Parameters in the environment that is being deployed to, all at deploy time

## Spoiler Alert! Here's the where the heart of the logic lies!

### Conditions Section:

1. __UserServicesTopicCreateNewResource__ (Lines 57 to 61)
    ```yaml
    Conditions:
      UserServicesTopicCreateNewResource: !Equals
        - 'false'
        - !Ref EnvConfigsnscrudtopicarnAsString
      UserServicesTopicUseExistingResource: !Not
        - Condition: UserServicesTopicCreateNewResource
    ```
    During a deployment to the __Chris__ Environment

    * `UserServicesTopicCreateNewResource` returns **true** because the value we saved in the in the config store ("false") is equal to the other value provided ('false').

    * `UserServicesTopicUseExistingResource` returns **false** because UserServicesTopicCreateNewResource returns ('true') in this environment.

    During a deployment to the __Prod__ Environment

    * `UserServicesTopicCreateNewResource` returns **false** because the value we saved in the config store ("arn:aws:sns:us-east-1:394493744367:stackery-186910833539367-topic60FA4D08") is NOT equal to the other value provided ('false').

    * `UserServicesTopicCreateNewResource` returns **true** because UserServicesTopicCreateNewResource returns ('false') in this environment..


## Full Template

The template can be found in github at this URL: https://github.com/stackery/demo-conditional-examples/blob/master/template.yaml

```yaml
AWSTemplateFormatVersion: 2010-09-09
Transform: AWS::Serverless-2016-10-31
Resources:
  UserServicesTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub ${AWS::StackName}-UserServicesTopic
    Metadata:
      StackeryName: UserServicesTopic
    Condition: UserServicesTopicCreateNewResource
  MicroService25:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub ${AWS::StackName}-MicroService25
      Description: !Sub
        - Stack ${StackTagName} Environment ${EnvironmentTagName} Function ${ResourceName}
        - ResourceName: MicroService25
      CodeUri: src/MicroService25
      Handler: index.handler
      Runtime: nodejs8.10
      MemorySize: 3008
      Timeout: 30
      Tracing: Active
      Policies:
        - AWSXrayWriteOnlyAccess
      Events:
        Topic:
          Type: SNS
          Properties:
            Topic: !If
              - UserServicesTopicUseExistingResource
              - !Ref UserServicesTopicExistingResource
              - !Ref UserServicesTopic
    Metadata:
      StackeryName: MicroService25
  UserServicesTopicExistingResource:
    Type: Custom::StackeryExistingResource
    Properties:
      ServiceToken: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:stackery-agent-commander
      Type: topic
      Data: !Ref EnvConfigsnscrudtopicarnAsString
    Condition: UserServicesTopicUseExistingResource
Parameters:
  StackTagName:
    Type: String
    Description: Stack Name (injected by Stackery at deployment time)
  EnvironmentTagName:
    Type: String
    Description: Environment Name (injected by Stackery at deployment time)
  EnvConfigsnscrudtopicarnAsString:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /<EnvironmentName>/sns_crud_topic_arn
Conditions:
  UserServicesTopicCreateNewResource: !Equals
    - 'false'
    - !Ref EnvConfigsnscrudtopicarnAsString
  UserServicesTopicUseExistingResource: !Not
    - Condition: UserServicesTopicCreateNewResource
Metadata:
  EnvConfigParameters:
    EnvConfigsnscrudtopicarnAsString: sns_crud_topic_arn

```
