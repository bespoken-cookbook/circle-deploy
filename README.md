# How It Works
This tool creates or updates a Fargate service, along with the pieces that it relies on (ELB Rules, Target Group, Health Check, etc.).

It assumes that an image has been pushed to Docker Hub - it takes the image name and creates a Fargate service around it.

The tool supports the following actions:
* create: Creates a service
* delete: Deletes a service
* service: Creates or updates a service (depending on if it already exists)
* update: Updates a service

The syntax for the script is:  
`fargate <action> <options>`

Example (creates or updates the service `my-service` with the specified parameters):
```
fargate service \
  --command "node lib/service.js" \
  --containerPort 3000 \
  --cpu 1024 \
  --env key=value \
  --image bespoken/my-service-image \
  --memory 2048 \
  --name my-service
```

For `create`, the script will:  
1) Register a task definition
2) Create an ELB target group, with health check
3) Create an ELB rule for the target group - assigned to <SERVICE_NAME>.bespoken.io
4) Create a service

For `update`, the script will:  
1) Register a task definition
2) Update the service

For `delete`, the script will:  
1) Delete ELB rule associated with the service
2) Delete the target group associated with the service
2) Update the desiredCount to 0 for the service (necessary before deletion)
4) Delete the service

For `service`, the script will:
1) Create the service if it does not exist
1A) Update the service if it already exists

For `schedule`, the script will (TO BE IMPLEMENTED):
1) Register a task definition
2) Create a Cloud Watch Event with cron configuration
3) Associate the CloudWatch Event with the task definition
More information here:  
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/scheduled_tasks_cli_tutorial.html

# Setup
Install the package:  
`npm install fargate-helper -g`

Set these environment variables - for the AWS SDK that this relies on:
* AWS_ACCESS_KEY_ID
* AWS_SECRET_ACCESS_KEY
* AWS_DEFAULT_REGION - defaults to `us-east-1` if not set

# Build-Time Configuration
The parameters for configuring the service come from three places:
1) Command-line
2) Environment variables
3) AWS Secrets - the "fargate-helper" key

## Required Parameters
These parameters must be manually configured for the deployment to run:  
* command: The command to run for the Docker service
* containerPort: The port the service should run on
* cpu: The CPU allocated for the service, where 1024 is equal to a full CPU
* image: The DockerHub image to use for this service
* memory: The amount of memory allocated for the service
* name: The name of the service

## Other Important Parameters
Though not required, these are useful parameters for more advanced cases:
* env: Key-value pair that is passed to the TaskDefition/container runtime
* envFile: The relative path to a file that contains environment settings - set inside the TaskDefinition/container runtime
* hostname: The fully-qualified hostname for the service - i.e., "service.bespoken.tools". When the default <SERVICE>.bespoken.io is not appropriate.
* logGroup: The CloudWatch Log Group to use - defaults to `fargate-cluster`
* passEnv: "true" or "false" - defaults to true. If set to false, will not automatically set pass thru environment variables in the build environment to the container environment
* taskDefinition: A file to use as the baseline for the taskDefinition - if not specified, just uses the default that is included in the code

## Command-Line Configuration
For the command-line, parameters are passed in with the format:  
`--name value`

For parameters with spaces, they should be passed in as so:  
`--name "my value"`

**NOTE** The memory and CPU must be valid for Fargate. The supported configurations are here:  
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-cpu-memory-error.html

## Environment Variables
Parameters can also be set via environment variable. If the parameter is not specified on the command-line, but is found in the environment, that will be used.

For example:
```
export cpu=1024
fargate create --name serviceName
```

In this case, the `cpu` will be set to `1024` for the service.

## AWS Secret Configuration
We store certain key default parameters, such as DockerHub credentials, in defaults in the AWS Secret Manager.  

These are values that are generally universal for the account - these should not need to changed or overridden.

They can be found under the name "fargate-helper". Values we store there are:
* accountId: The AWS account ID
* albArn: The ARN for the ALB being used for this account
* cluster: The fargate cluster name
* dockerHubSecretArn: The name of the AWS Secret that stores our docker credentials
* listenerArn: The ARN for the ALB listener being used for this account
* roleArn: Used for taskRoleArn and executionRoleArn
* securityGroup: The VPC security group to use
* subnets: The list of subnets to use
* vpcId: The VPC ID used by this configuration - specified when creating the target group on the ALB

# Runtime Configuration
Environment variables can also be set inside the running container.

If `--passEnv` is set to true, we take all the environment variables currently set and pass them to the container in the taskDefinition, under environment.

Environment variables in the container can also be set by specifying on the command-line:  
`fargate create --env KEY=VALUE`

This will set the environment variable `key` to `value` inside the container.

Additionally, the `--envFile` parameter can be used - this takes as an argument a relative path to a file in key value form. All values in this file will be set in the container (via the Environment section on the TaskDefinition).

# Load Balancer Configuration
By default, we create a target group with rules on our ELB when first creating the service.

The target group will be configured:
* With a health-check that pings the service every thirty seconds, via a GET on /
* With a rule that directs traffic for `<SERVICE_NAME>.bespoken.io` to the service

For more complex ELB configurations, we recommend they be done manually, and then only update be called (which does not modify the ELB configuration).

# Examples

## Delete
```
fargate delete --name my-service
```

## Real-world
To see a sample project that uses this, check out the Utterance Tester:  
https://github.com/bespoken/UtteranceTester

In particular, here is the Circle CI configuration:  
https://github.com/bespoken/UtteranceTester/blob/master/.circleci/config.yml#L32
