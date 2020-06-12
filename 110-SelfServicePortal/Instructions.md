# Instructions for the 110 Wordpress Self Service
All scripting runs from the 110 directory
## Setup
### Create S3 Buckets
```bash
aws cloudformation create-stack --template-body file://Business/provisioningbuckets.yaml --stack-name whibuckets
```
That gave me two output links:
```bash
aws cloudformation describe-stacks --stack-name whibuckets --output yaml --query Stacks[*].Outputs[*]
```

```yaml
- - Description: TemplateURL for use in support.html
    OutputKey: resbucketurl
    OutputValue: http://whibuckets-resbucket-1n126tfxx7fea.s3.amazonaws.com/wp-110-Linux1-distro.yaml
  - Description: URL for Self Service Portal
    OutputKey: selfservebucketURL
    OutputValue: http://whibuckets-selfservebucket-of3tzclmylm1.s3-website-us-east-1.amazonaws.com
  - Description: S3 location to copy wp-110-Linux1-distro.yaml to
    OutputKey: resbucket
    OutputValue: s3://whibuckets-resbucket-1n126tfxx7fea
```

### Configure Google Authentication
 * Go to https://console.developers.google.com/projectselector/apis/library?pli=1
 * Create a new project
 * Navigate to APIs & Services -> OAuth consent Screen
   * Pick External
   * Add in an Application name
   * Put the selfservebucketURL into the Authorized Domains box
 * Click '+ Create Credentials'
   * Use OAuth client ID
   * Choose the 'Web Application' Application type
     * Enter selfservebucketURL into the Authorized JavaScript Origins 
	 * Enter selfservebucketURL into the Authorized redirect URIs
 * Save the credentials
   * Client ID: 816410849714-h28vm7l44b2h1gahem0fhlm3rttolj2g.apps.googleusercontent.com
   * Secret: YnEuB2C1rjdCpKdj1l3S8NW8
   
### Create IAM role to manage stacks
```bash
aws cloudformation create-stack --template-body file://Business/stackrole.yaml --stack-name whi-stack-role --capabilities CAPABILITY_NAMED_IAM
```
Gather output:
```bash
aws cloudformation describe-stacks --stack-name whi-stack-role --output yaml
```

```yaml
Stacks:
- Capabilities:
  - CAPABILITY_NAMED_IAM
  CreationTime: '2020-06-12T17:48:06.589000+00:00'
  DisableRollback: false
  DriftInformation:
    StackDriftStatus: NOT_CHECKED
  EnableTerminationProtection: false
  NotificationARNs: []
  Outputs:
  - Description: Stack Role arn
    OutputKey: stackrole
    OutputValue: arn:aws:iam::406319049568:role/whi-stack-role-iamrole-11LQKHQ0PFOXP
  RollbackConfiguration: {}
  StackId: arn:aws:cloudformation:us-east-1:406319049568:stack/whi-stack-role/df2d3eb0-acd4-11ea-ba49-0ed4cff9b69d
  StackName: whi-stack-role
  StackStatus: CREATE_COMPLETE
  Tags: []
```
### Create Customer Role
This is the role that will be used by the customer after they've authenticated via Google.

We'll also create a customer user with read-only access to AWS. The only reason for allowing this access is so that they can see what's going on in the AWS account on their behalf.

```bash
aws cloudformation create-stack --template-body file://Customer/customeriamandrole.yaml --stack-name whi-customer --capabilities CAPABILITY_NAMED_IAM
```

Gather the output:
```bash
aws cloudformation describe-stacks --stack-name whi-customer --output yaml
```

```yaml
Stacks:
- Capabilities:
  - CAPABILITY_NAMED_IAM
  CreationTime: '2020-06-12T17:55:41.763000+00:00'
  DisableRollback: false
  DriftInformation:
    StackDriftStatus: NOT_CHECKED
  EnableTerminationProtection: false
  NotificationARNs: []
  Outputs:
  - Description: Customer Access Key ID
    OutputKey: customerakid
    OutputValue: AKIAV5GUAVNQMVYLBKRT
  - Description: Customer Role arn
    OutputKey: customerrole
    OutputValue: arn:aws:iam::406319049568:role/whi-customer-iamrole-RSB4WC6XLL0U
  - Description: Customer Secret Access Key
    OutputKey: customersecretkey
    OutputValue: L35OthVzEhstTM81nRbyg7BkbCc2SyUdvgQKMVyu
  RollbackConfiguration: {}
  StackId: arn:aws:cloudformation:us-east-1:406319049568:stack/whi-customer/e5012350-acd5-11ea-a349-126bf0867249
  StackName: whi-customer
  StackStatus: CREATE_COMPLETE
  Tags: []
```

### Create support team login
We will be using a manual user to create customer stacks - this should be fully automated, responding to the customer generated event (probably via a lambda function?) after they've paid for the service. However this is the user that will be used to login.

```bash
aws cloudformation create-stack --template-body file://Business/supportiam.yaml --stack-name whi-support --capabilities CAPABILITY_NAMED_IAM
```

```bash
aws cloudformation describe-stacks --stack-name whi-support --output yaml
```

```yaml
aws cloudformation describe-stacks --stack-name whi-support --output yaml
Stacks:
- Capabilities:
  - CAPABILITY_NAMED_IAM
  CreationTime: '2020-06-12T18:00:44.217000+00:00'
  DisableRollback: false
  DriftInformation:
    StackDriftStatus: NOT_CHECKED
  EnableTerminationProtection: false
  NotificationARNs: []
  Outputs:
  - Description: Support Secret Access Key
    OutputKey: adminsecretkey
    OutputValue: S1av6ovFN36d6SD81pci7GNs7+K1TE9NVzT/u4Hu
  - Description: Support Access Key ID
    OutputKey: adminakid
    OutputValue: AKIAV5GUAVNQP4VPEJRT
  RollbackConfiguration: {}
  StackId: arn:aws:cloudformation:us-east-1:406319049568:stack/whi-support/a2bff790-acd6-11ea-9d98-1246411399d1
  StackName: whi-support
  StackStatus: CREATE_COMPLETE
  Tags: []
```

### Set up cognito
* select the cognito service
* select Manage Identity Pools
* Choose 'WHI Identity Pool' for the pool name
* Select Authentication Providers
* Select Google+
* Enter Google client id
* Select 'create pool'
* Select 'cancel'
* In the top banner, select 'click here to fix it'
* Select the Authenticated Role drop down, and pick the whicustomer-iam-role
* click 'save changes'

### Configure Web Pages
#### Support users web page
Support users will be using `support.html` web page. Instead of authenticating them we're just going to hard code their credentials into the web page.

We're also going to add the region, the arn that will be used to deploy the wordpress template as well as the TemplateURL for that template (the TemplateURL was output as part of the provisioningbuckets stack). 

Copy that file to the selfserve bucket:

```bash
aws s3 cp Business/support.html s3://whibuckets-selfservebucket-of3tzclmylm1
```
#### Self server portal web page
Now do similar for the `self-serve.html` web page.
There are three things to change:
 1. Cognito pool id
 2. Cognigo region id
 3. Google client id

Get the cognito identity pool id thus:
```bash
aws cognito-identity list-identity-pools --output text --max-results 1 --query IdentityPools[*].IdentityPoolId
```

Copy the changed file to the self serve portal:

```bash
aws s3 cp Customer/self-serve.html s3://whibuckets-selfservebucket-of3tzclmylm1
```



## Execution
Open up two tabs, one as the customer, one as the support engineer.

They both use the self-serve url, but with the `self-serve.html` or `support.html` paths, respectively

As the customer, login using your Google account. You'll see your ID, along with an empty list of provisioned resources.

As the support engineer, you can list the currently running stacks, which are all those used to setup this demo

As the support engineer, create a stack, using a client name format of 'WP<profileid>STACK<N>' where '<N>' is the unique id for that customer's stack.


