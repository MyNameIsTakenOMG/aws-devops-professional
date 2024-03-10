# aws-devops-professional
## Table of Contents
 - [SDLC automation](#sdlc-automation)
 - [Configuration Management and IaC](#configuration-management-and-iac)
 - [Resilient cloud solutions](#resilient-cloud-solutions)
 - [Monitoring and Logging](#monitoring-and-logging)
 - [Incident and Event Response](#incident-and-event-response)
 - [Security and Compliance](#security-and-compliance)
 - [Other Services](#other-services)

## SDLC Automation
 - CI/CD: having our code in a repo and deploy it onto AWS `automatically, the right way, making sure it's tested before being deployed, with possibility to go into different stages(dev, test, staging, prod), with manual approval where needed`.
   - CI: dev push code to the repo, a test/build server checks the code and gives the feedback to the dev.
   - CD: ensure software can be released reliably when needed and deployment happens ofter and quickly
   - tech stack: (code: codeCommit/github/...) -> (build & test: codebuild/jenkins/...) -> (deploy: codeDeploy) -> (provision: EC2/on-prem/lambda/ECS/...)
     - use `Elastic Beanstalk` to cover `deploy` and `provision` stages
     - use `codePipeline` to manager the whole process
 - CodeCommit: version control/ using GIT/ Private repo/ no size limit/ fully managed, HA/ integrate with various CI tools/ Security: ssh key & https(authentication); IAM policies(authorization); encryption(automatically with MKS at rest, in transit, https or ssh); using IAM role and STS (cross-account access) 
 - CodeCommit - advanced:
   - `monitoring with eventbridge(near real-time)`
   - `migrate git repo to codeCommit`
   - cross-region replication: use `eventbridge` to listen to events, then invoke ECS task to replicate to another repo in another region. Use case: lower latency for global devs, backups,...
   - branch security: using IAM policies to restrict users to push or merge code to a specific branch. `note: resource policy not supported yet`
   - pull request approval rules: specify a pool of users to approve PR and the number of approvals. specify IAM principal ARN(users, roles,groups). then setup `Approval rule templates` 
 - CodePipeline: visual workflow to orchestrate your CI/CD. `source`,`build`,`test`,`deploy`,`invoke`. Each stage can have sequential actions and/or parallel actions. Manual approval can be defined at any stage. `Artifact`: each stage will output artifacts and put them into S3 bucket, then will be put into next stage. `Troubleshooting`: use cloudwatch events for failed pipelines or cancelled stages. if a stages failed, it can be seen in the console. if pipeline cannot perform an action, check the IAM role. Also cloudtrail can be used to audit aws api calls.
 - CodePipeline - extra:
   - events vs webhooks vs polling
   - manual approval --> two permissions: `get pipeline`, `put approval result`
 - CodePipeline - cloudformation integration: `cloudformation deploy action` can be used to deploy aws resources. to deploy resources across accounts or regions, using `stacksets`. Using `CREATE_UPDATE` mode for existing stacks, and `DELETE_ONLY` to delete an existing stack. `workflow`: codebuild-->cfn(create_update)-->codetest-->cfn(delete_only)-->cfn deploy prod infra(create_update)
   - action modes: create/replace/execute a change set; create/update/delete/replace a stack
   - template parameter override
 - CodePipeline - advanced:
   - best practices: one codepipeline, one codedeploy, parallel deploy to multiple deployment groups; parallel actions used in a stage using `RunOrder` (similar to github action without `steps`); deploy to a `pre-prod` before deploying to prod
   - with EventBridge: detect and react to any failures in any stage (such as invoke lambda or send SNS)
   - Invoke actions: lambda & step functions
   - multi-region: actions in the pipeline can be in different regions; s3 artifact stores must be provisioned in each region(must have read/write); codepipeline will handle copying artifact across regions automatically. (multiple templates may need to be created for multiple regions)
 - CodeBuild:
   - source: codeCommit, s3, bitbucket, github
   - build instructions: `buildspec.yaml` or insert manually in console
   - output logs: can be stored in s3 & cloudwatch logs
   - monitor using cloudwatch metrics
   - eventbridge to detect failed builds and trigger notifications
   - cloudwatch alarms to notify if you need `thresholds` for failures
   - build projects can be defined within codepipeline or codebuild
   - `buildspec.yml`:
     - located at the root dir
     - env: `variables`,`parameter-store`,`secrets-manager`
     - phases: `install`,`pre_build`,`build`(build commands),`post_build`
     - artifacts: upload to s3(using KMS)
     - cache: files to cache(usually deps) to s3 for future build
   - codebuild - local build: codebuild agent for deep troubleshooting beyond logs
   - codebuild - inside VPC: by default, codebuild runs outside the VPC. you can specify: VPC id, subnet id, security group id, then your build can access resources in your vpc. use case: integration tests, data query, internal load balancers...
 - CodeBuild - advanced:
   - env variables: default env variables, custom env variables(static--defined at build time, dynamic--parameter store, secrets manager)
   - security: codebuild service role
   - build badges: support codecommit, github, bitbucket. available at branch level
   - triggers: eventbridge, lambda, github(webhook)
   - validate pull requests
   - test reports: various tests with any test framework. in `buildspec.yml`, add a `report group` in the `reports` section
 - CodeDeploy: deploy new versions to ec2, on-prem, lambda, ecs; automated rollback capability in case of failures or trigger cloudwatch alarm; gradual deployment control; a file `appspec.yml` configuring the deployment
   - ec2/on-prem: perform in-place or blue/green deployments; must run codedeploy agent on the target instances; define deployment speed: allatonce, halfattime, oneatatime, custom.
   - in-place deployment: update a certain part at a time
   - blue/green deployment: create a new identical group but with new version, then update
   - codedeploy agent: must on ec2 as pre-requisites, using ssm to install and update automatically. must have permissions to access s3 to get deployment bundles.
   - lambda: help automate traffic shift for lambda aliases; feature integrated with SAM. linear: grow traffic every N min until 100%. canary: try X percent then 100%. AllAtOnce
   - ecs: only support blue/green deployment. linear, canary, allatonce
 - CodeDeploy - EC2 deep dive: use ec2 tags or asg to identify instances you want to deploy to. Deployment Hooks: certain scripts run by codedeploy on each ec2 instances. process(using load balancer): block traffic, app stops, install , restart app, validate service, allow traffic.
   - deployment hooks examples: beforeinstall(such as decrypting files, creating backup), afterinstall(such as configure app), applicationstart, validateservice, beforeallowtraffic(such as perform health checks before registering it to load balancer)
   - blue/green deployment(must have a load balancer): manually mode: provision blue and green and identify by tags. automatic mode: new asg is provisioned
   - blue/green instance termination: blueinstanceTerminationOption(whether delete blue after deployment), action terminate(specify wait time), action keep alive(instances keep running but deregisterd from ELB and deployment group)
   - deployment hooks: some for v1, others for v2
   - deployment configurations: specify number of instances remain available during the deployment, using pre-defined configs: allatonce, halfatatime, oneatatime or custom
   - trigger: send events to SNS topic
 - CodeDeploy - ECS deep dive: automatically handle new ecs task definition to ecs service. only support blue/green, ecs task definition and container image must be ready, the task definition and load balancer info are specified in `appspec.yml`. no codedeploy agent required
   - deployment to ecs: linear, canary, all at once. can define a second ELB test listener to test the green before traffic is rebalanced.
   - deployment hooks: all lambda functions. such as `afterAllowTestTraffic`: perform health check and trigger a rollback if it is failed
 - CodeDeploy - lambda deep dive: a lambda alias pointing to v1, then create a v2 and specify the version info in the `appspec.yml`, then codedeploy updates the version by making lambda alias pointing to v2. no codedeploy agent required.
   - only blue/green. linear, canary, all at once
   - hooks
 - CodeDeploy - rollbacks & troubleshooting:
   - rollback: redeploy the previous version (automatically when cloudwatch alarm threshold breach, or manually)
   - disable rollbacks
   - when rollback, a new deployment will be created(not restore the version)
   - troubleshooting: exception`InvalidSignatureException`: when the date and time on ec2 instance is not set correctly, they might not match the signature date of your deployment request, which codedeploy rejects.
   - when deployment or all lifecycle events are skipped(ec2/on-prem) with errors: `too many individual instances failed deployment`, `too few healthy instances for deployment`, `some instances are experiencing problems`. Reasons: no codedeploy agent, service role, codedeploy agent with http proxy, date and time mismatch between codedeploy and agent.
   - when an asg is performing scale-out operation, a new instance with v1 will be created, but by default, codedeploy will perform a follow-on deployment to update the version.
   - when failed `allowTraffic` in blue/green with no error, check out ELB health check and correct it.
 - CodeArtifact: software packages or dependencies. store and retrieve them is called artifact management. work with common package management tools, such as maven, gradle, npm, pip. devs and codebuild can retrieve packages from codeartifact
   - eventbridge integration
   - resource policy used for cross-account premissions
 - CodeArtifact - upstream repositories & domains:
   - upstream repos: a codeartifact repo can have other codeartifact repos as upstream repos, so that a single repo endpoint is created.
   - external connection: a codeartifact repo and an external repo. 1-to-1
   - retention: upstream repo change will affect downstream, but we can delete it and update the package in the downstream. all intermediate repos do not keep the package, only the ones connect to the client and the external repo
   - domain: deduplicate storage by making a shared storage, thus we have fast copying, easy sharing across repos and teams. apply resource-based policy 
 - CodeGuru:
   - ML-powered service for code reviews and app performance recommendations: reviewer: static code analysis(development). profiler: recommendations about app performance during runtime(production)
 - CodeGure - extra:
   - codeguru reviewer secrets detector: identify secrets in your code, and suggest remediation with secrets manager.
   - codeguru profiler -extra: `function decorator @with_lambda_profiler`, or enable it through aws console
 - EC2 image builder: used to automate the creation of VM or container images(create, maintain, validate, and test ec2 ami). can be on a schedule. free service and can publish ami to multiple regions and multiple accounts.
 - EC2 image builder - extra: sharing using Resource access manager(RAM) to share images, recipes and components across aws accounts or through aws organization. tracking latest ami (using parameter store to store the latest ami version or id)
 - AWS amplify: elastic beanstalk for web and mobile apps
 - AWS amplify - extra: CD: connect to codecommit and have one deployment per branch(dev, prod), connect your app to a custom domain via route53

## Configuration Management and IaC
 - Cloudformation - overview: declarative way to provision aws infras
   - benefits:
     - Infrastructure as code
     - cost: resources within the stack have an identifier so that we can see how much does the stack costs you; estimate the costs of resources using cloudformation template; saving strategy: in dev, we can delete and recreate resources at certain time period.
     - productivity: declarative programming
     - seperation of concern: such as vpc stack, network stack, app stack
     - no need to re-invent the wheel: checkout existing templates, and docs
   - workflow: create and upload a template to s3 bucket, then cloudformation will create the stack, to update the stack, we have to upload a new version. when deleting a stack, all artifacts created by cloudformation will be deleted.
   - deploy cloudformation template: manual way & automated way
 - Cloudformation - create/delete stack
 - cloudformation:
   - resources: mandatory; the resource types identifiers are of the form: `service-provider::service-name::data-type-name`. **note:** for creating a dynamic number of resources, we can use cloudformation Macros and Transform. and using cloudformation resources to provision custom resources.
   - parameters: provides a way to insert inputs for your template. use cases: reuse the template or some inputs cannot be determined ahead of time. when to use parameters: if the certain config will likely change in the future, then make one. for the types of parameters: such as allowedValues or noEcho. to refer to a parameter, use the intrinsic function `!Ref <parameter name>` or `Fn::Ref`
     - Pseudo parameters: Aws offers a bunch of pseudo parameters by default which can be used at any time. Such as `AWS::AccountId`,`AWS::Region`,`AWS::StackId`,`AWS::StackName`,`AWS::NotificationARNs`,`AWS::NoValue` 
   - mappings: fixed variables(all possible values are known in advance), use the intrinsic function `Fn::FindInMap`(!FindInMap [<map-name>, <path>,... ]). use parameter when values are user-specific.
   - outputs & exports: the best way to perform collaboration cross stacks, using `Export: <name for the outputs> in the Outputs section`, and use the intrinsic function `Fn::ImportValue` to refer to the outputs, `!ImportValue <name of the outputs>`
   - conditions: used to control of the creation of resources or outputs based on a condition. to create a condition: `<condition name>: !Equals [!Ref envType, prod]`(and, equals, if, not, or...). to use a condition, `add a condition: <condition name> to the resource`.
   - intrinsic functions: Ref, Base64, GetAtt, FindInMap, ImportValue, Condition functions, ...
   - rollbacks:  when stack creation fails: by default, rolls back, or to disable rollback and troubleshoot. when update fails: by default, rolls back, able to see in the log of what happened. when rollback fails, fix resources manually then issue `continueUpdateRollback` api call from console/cli 
   - service role: IAM roles that allows cloudformation to create/update/delete resources on your behalf even if you don't have permission to work with the resources in the stack. the user must have a `iam::PassRole` permissions
   - capabilities: `CAPABILITY_NAMED_IAM` and `CAPABILITY_IAM`: iam-related resources; `CAPABILITY_AUTO_EXPAND`: needed when template include Macros or nested stacks to perform dynamic transformations. `InsufficientCapabiiltiesException`: if the capabilities haven't been acknowledged when deploying a template.
   - deletion policy: by default, deletionPolicy=Delete. **note:** s3 bucket won't be deleted if it's not empty.
     - deletionPolicy - retain: such as dynamodb table
     - deletionPolicy - Snapshot: create a snapshot before deleting the resource
   - stack policy: during stack update, all update action are allowed on all resources. a stack policy defines update actions allowed on certain resources during the stack updates (`Allow`/`Deny`)
   - termination protection:  applied on the entire stack (differ from deletion policy)
   - custom resources: used to define resources not supported by cloudformation, or on-prem, 3rd-party resources, or having custom scripts to run through create/update/delete through lambda functions(such as empty an s3 bucket before deleting it). `AWS::CloudFormation::CustomResource` or `Custom::MyCustomResourceTypeName`(recommended). backed by lambda or SNS topic.
     - define a custom resource: `serviceToken`: where cloudformation sends requests to, such as lambda ARN or SNS ARN(must be in the same region). input data parameters.
   - dynamic references: refer to the external values stored in SSM parameter store and secrets manager. Supports: ssm/ssm-secure/secretsmanager, `{{resolve:<service-name>:<reference-key>:<version>}}`
     - option1 - `ManageMasterUserPassword: true`: create admin secret implicitly for rds, aurora
     - option2 - `dynamic reference`: 1. create secret 2. reference secret in rds instance. 3 secretRDSAttachment: link secret to the db instance for rotation.
   - user data: a script to be executed during the launch of ec2 instance for the first time. **note:** the important thing to pass is the entire script through the function `Fn::Base64`. Good to know, the use data script log is in the `/var/log/cloud-init-output.log`
   - cfn-init: `cloudformation helper scripts: python scripts that comes on AMIs or installed using yum on non-amazon amis`. `under metadata section, aws::cloudformation::init(config: packages, files, commands, services)`
   - cfn-signal & wait condition: run `cfn-signal` after `cfn-init`. we need waitCondition: block the template until there's a signal from `cfn-signal`. we attach a `<sample wait condition>.CreationPolicy`, we can define a `Count > 1` if we need more signals.
   - cfn-signal failures: wait condition didn't receive the required number of signals from resource, such as ec2 instance:
     - ensure the AMI has helper scripts installed, or download them to the instance.
     - verify the `cfn-init`, `cfn-signal` run successfully, viewing the logs at `/var/log/cloud-init.log`,`/var/log/cfn-init.log`.
     - make sure to disable the `rollback on failure`, otherwise, all logs and the instance will be delted
     - make sure the instance has the access to the internet if it's in a vpc or even a private subnet 
   - nested stacks: allows you to isolate repeated patterns/common parts. cross stack vs nested stacks
   - depends on: `DependsOn`, applied automatically when using `!Ref` or `!GetAtt`
   - troubleshooting: `delete_failed`: such as delete s3 bucket when it's not empty, or use custom resources with lambda functions to automate some actions like emptying s3 bucket; security groups cannot be deleted until all ec2 instances in the group are gone; or check the deletionpolicy(if it's retain-->skep deltion). `update_rollback_failed`: can be caused by resources changed outside of cloudformation, insufficient permissions, asg that doesn't receive enough signals... Manually fix the error, then `continueUpdateRollback`. `stackset--troubleshooting`: insufficient permission in a target account for creating resources specified in your template, or trying to create a global resource but not unique, or admin account has not a trusted relationship with target accounts, or reached the limit or quota in the target account(too many resources) 
   - changesets: it won't tell if the update will be successful, but provide a preview of what will be changed before applying them. for nested stacks, you will see the changes across all stacks
   - cfn-hup: can be used to tell your ec2 instance to look for metadata changes every 15 min and apply the metadata config again, it replies on `cfn-hup` config: `/etc/cfn/cfn-hup.conf` and `/etc/cfn/hooks.d/cfn-auto-reloader.conf`
   - drift: cloudformation drift detect drift on entire stack or individual resource. or perform drift detection on stackset. any changes made outside of cloudformation is considered drifted. (drift detection feature has to be enabled manually)
 - stacksets - warning: using admin account to create/update/delete stackset, once update or delete stackset, it will applied into all accounts or regions/ or all accounts of AWS organizations.
   - permission models: `self-managed`: create iam roles for admin and target and build a trust relationship. `service-managed`: utilizing AWS organization(enable all features and trusted access). deploy stack to new account in organization automatically. can delegate stacksets admin to memeber account. again, trusted access in organization must be enable before delegating admin.
 - cloudformation - stacksets (create / update / delete)
 - service-catalog: admin users create a portfolio(collection of products--templates) with iam permissions, then users with proper iam permissions will launch certain template. help ensure consistency, governance, compliance. integration with `self-service portals` such as ServiceNow.
 - servie-catalog - extra:
    - stack set constraints: accounts, regions, permissions
   - launch constraints: iam role assigned to a template(product), example: end user has only access to the catalog, others permissions required are attached to the launch constaint iam role. iam role mush have permissions: cloudformation(full access), aws services in the template, s3 bucket which contains the cloudformation template(read access)
   - CD pipeline(syncing with codeCommit): codeCommit(mapping.yml-->product<->template)-->lambda(checkout template)-->update/create service catalog(according product)
 - Elastic Beanstalk
   - overview: application, application version, environment
   - HA environment: 
   - deployment modes: All at once / Rolling / Rolling with additional batches( old app is still available while spinning up new instances) -- additional cost / immutable (spins up new instances in a new asg, deploys version to these instances, then swaps all the instance when everything is healthy) -- high cost / (blue/green): create a new env and switch over when ready by utilizing Route53 to redirect a portion of traffic to the new env, then swap urls when done with env test / traffic splitting(canary testing) - send a small % of traffic to new deployment (a temporary asg of new version with a small portion of traffic, then migrate them to the main asg and terminate old version, otherwise automatically rollback)
   - extra: web server vs worker env(long-term task); notification via eventbridge
 - serverless application model(SAM)
   - overview: a framework for developing and deploying serverless apps. can use codedeploy to deploy lambda functions and help run lambda, api gateway, dynamodb locally.
     - recipe:
     - workflow: app code + sam template --> app code + cfn template --> sam package or zip cfn package to s3 bucket --> deploy to cfn to build stack
     - cli debug: locally build, test, debug serverless apps. providing a lambda -like execution env locally. sam + aws toolkit --> step through and debug your code (an IDE plugin which allows to test lambda functions using aws sam)
   - with CodeDeploy: natively use codedeploy; traffic shifting feature; pre and post traffic hooks; easy & automated rollback using cloudwatch alarms
     - sam and codedeploy:
       - autoPublishAlias: detect changes; create and publish a updated version; point the alias to the new version.
       - deploymentPreference: canary / linear / allAtOnce
       - alarms: alarms that trigger a rollback
       - hooks: pre and post traffic shifting lambda functions to test your deployment
 - cloud development kit (CDK): using programming languages to define infras `constructs`, which will be transformed into cloudformation template. thus, you can deploy app code and infra code together: great for lambda function and docker containers in ecs/eks.
   - cdk + sam: use sam cli to test your cdk apps
 - step functions: model your workflow as state machine(start workflow with SDK call, API gateway, eventBridge).
   - task states: invoke one aws service, or run an activity
   - states: choice state / fail or success state / pass state / wait state / map state / parallel state
 - AppConfig: could have config outside of your app; deploy dynamic config changes to your app independently(no need to restart the app). feature flags, app tuning, allow/block listing... gradually deploy the config changes and rollback if issues occur. validate config changes before deployment using: json schema(syntactic check) or lambda function - run code to perform validation (semantic check)
 - system manager (SSM): help manage ec2 and on-prem systems at scale; patching automation for enhanced compliance; work for windows and linux; integrate with cloudwatch and aws config; easily detect problems and get insights of your infra.
   - features: Resource groups, shared resources--documents, change management--automation,maintenance windows, application management--parameter store, node management--inventory,session manager, run command, state manager, patch manager. need a ssm agent installed on the target machine(such as Amazon linux 2 ami), check the ssm agent installation as well as iam role which allows ssm actions
 - AWS tags & SSM resource groups: add key-value tags to aws resources, which used for resources grouping, automation, cost allocation... better to have too many tags than too few. / create, view or manage logical group of resources thanks to tags, allows to create logical groups of resources such as apps, different layers of app stack, prod vs dev envs. regional service. work with ec2, s3, dynamodb, lambda, etc.
 - SSM documents(center of SSM) & SSM run command:
   - documents: define parameters, actions, many existing documents. these documents can be used in other SSM features.
   - run commands: execute a document, across multi-instances, rate control/error control, integrated with iam and cloudtrail, no need for ssh, output can be shown in the console, s3 bucket or cloudwatch logs, send notifs to SNS abou the command status, can be invoked by eventbridge.
 - SSM automations
   - overview: simplify common maintenance and deployment tasks of ec2 instances and other aws resources. `automation runbook`: ssm documents of type automation, pre-defined runbooks or custom runbooks. can be triggered by aws console, SDK, cli, eventbridge, maintenance windows, aws config remediation. 
   - use case
 - SSM parameter store: secure storage for config and secrets/ version tracking/ eventbridge/ cloudformation/ iam/ kms/ serverless,scalable,durable,easy SDK./ store hierarchy./ parameter policies(advanced tier): set TTL to a parameter, assign multiple policies at a time.
 - SSM patch manager and maintenance windows:
   - patch manager: automates patching process, os(linux, macos, windows) updates, app updates, security updates... support both ec2 and on-prem, patch on-demand or scheduled using `maintenance windows`, scan instances and create patch compliance report and send to s3 bucket. `patch baseline`: what should or should not be installed. `patch group`: associate a set of instances with a specific patch baseline. instances should be defined with the tag key `Patch Group`. one instance only attached to one group. one group only registered with one baseline. `pre-defined patch baseline`: `AWS-RunPatchBaseline(ssm document)`, `custom patch baseline`
   - maintenance windows: defines a schedule for when to perform actions on your instances. `schedule`, `duration`, `set of registered instances`, `set of registered tasks`
 - SSM session manager
   - overview: allows to start a secure shell on your ec2 and on-prem. no need for ssh access, bastion hosts or ssh key. support multi-os, logs can be sent to s3 bucket or cloudwatch logs. cloudtrail can be used to intercept `StartSession` events. `IAM permissions`: control access. optionally, restrict commands a user can run.
   - with VPC endpoints: to connect to ec2 instances in a private subnet without internet access: vpc interface endpoint(for ssm), vpc interface endpoint(for ssm session manager), vpc interface endpoint(optional kms), vpc interface endpoint(optional cloudwatch logs), vpc gateway endpoint(optional - s3 bucket) **note**: for s3 gateway endpoint, update route tables
 - ~SSM cleanup~
 - SSM default host management configuration (DHMC): when enabled, ec2 instance will be config as managed instance without the use of `ec2 instance profile`. `Instance identity role`: a type of iam role with no permissions beyond identifying the instance to aws service(such as ssm). ec2 instances must have `IMDSv2 enabled` and `SSM Agent installed`. session manager, patch manager and inventory enabled automatically. must be enabled per region. ssm agent up to date automatically.
 - SSM hybrid environments: setup ssm to manage on-prem servers, IoT devices, edge devices and virtual machines(for hybrid managed nodes, use the prefix 'mi-'). workflow: hybrid activation --> activation code and Id --> install ssm agent --> registered with activation code and id(could use API gateway, lambda and ssm to automate the process)
 - SSM with IoT greengrass: IoT Core runs in the cloud and Greengrass runs at the edge (usually). IoT Core is a cloud service but Greengrass is an edge runtime. to manage iot greengrass core devices using ssm, install ssm agent on core devices and add permissions to the `token exchange role`(IAM role for the IoT core device) to communicate with ssm, support all ssm features. use cases: update and maintain os and software across a fleet of greengrass core devices.
 - SSM compliance: scan managed nodes for patch compliance and config inconsistencies. display current data about `patched in patch manager`, `associations in the state manager`. can sync data  to s3 bucket using `resource data sync` and analyze using athena and quicksight. can collect and aggregate data from multiple accounts and regions. can send compliance data to `security hub`. 
 - SSM opsCenter: allows to view, investigate, and remediate issues in one place(no need to go to different aws services). security issues(security hub), performance issues(dynamodb throttle), failures(asg failed launch instance),... reduce meantime to resolve issues. support both ec2 instances and on-prem nodes. `OpsItems`: issues or interruptions; event, resource, aws config changes, cloudtrail logs, eventbridge... `provide recommended runbooks to resolve the issue` 
 - AWS opsWorks
   - get started (1 & 2)
   - lifecycle events
   - cloudwatch events integration
   - summary
## Resilient cloud solutions
 - Lambda
   - versions and aliases
   - environment variables
   - concurrency
   - file systems mounting
   - cross-account file systems mounting
 - API Gateway
   - overview
   - stages and deployment
   - Open API
   - caching
   - canary deployment
   - monitoring, logging and tracing
 - ECS:
   - overview
   - auto scaling
   - solution architectures
   - logging
 - ECR
 - ECR - extra
 - EKS
 - EKS - logging
 - Amazon Kinesis
 - kinesis data streams
   - overview
   - consumers scaling
 - kinesis data firehose
 - kinesis data analytics
   - overview
   - using ML
 - Route53
   - overview
   - routing policies:
     - weighted
     - latency
     - failover
 - RDS read replicas vs multi-AZ
 - Aurora - extra
 - elasticCache
   - overview
   - redis cluster modes
 - dynamodb
   - overview
   - advanced features
 - AWS DMS
   - overview
   - monitoring
 - S3 - replication
 - AWS storage gateway
   - overview
   - file gateway cache refresh
 - Auto scaling groups
   - scaling policies
   - lifecycle hooks
   - event notifications
   - termination policies
   - warm pools
 - application auto scaling
 - ELB
   - ALB rules deep dive
   - extra
 - NAT gateway
 - multi-AZ architectures
 - blue-green architectures
 - multi-region architectures
 - disaster recovery
## Monitoring and Logging
 - cloudwatch metrics
 - cloudwatch custom metrics
 - cloudwatch anomaly detection
 - amazon lookout for metrics
 - cloudwatch-logs
 - cloudwatch-logs - live tail
 - cloudwath-logs - metric filters
 - all kinds of logs
 - cloudwatch agent & cloudwatch logs agent
 - cloudwatch alarms
 - cloudwatch synthetics
 - amazon Athena
## Incident and Event Response
 - eventBridge
   - overview
   - content filtering
   - input transformation
 - S3 - event notifications
 - S3 - object integrity
 - AWS health dashboard
   - overview
   - events & notifications
 - EC2 instance status checks
 - cloudtrail
   - overview
   - eventBridge integration
 - SQS - dead letter queues
 - SNS - redrive policy
 - AWS X-Ray
 - AWS X-Ray with Beanstalk
 - AWS Distro for openTelemetry
## Security and Compliance
 - AWS config
   - overview
   - configurations recorder and aggregator
   - conformance packs
   - organizational rules
 - AWS organizations
   - overview
   - service control policy (SCP)
 - AWS control tower
   - overview
   - landing zones
   - account factory & migrating accounts
   - customizations for AWS control tower (CFCT)
   - config integation
   - account factory for terraform
 - IAM identity center
   - overview
   - extra
 - AWS web application firewall (WAF)
 - AWS firewall manager
   - overview
   - policies
 - Amazon guardduty
   - overview
   - advanced
   - cloudformation integration
 - amazon detective
 - amazon inspector
   - overview
   - EC2 setup
 - EC2 instance migration using AMIs
 - AWS trusted advisor
   - overview
   - architectures
 - AWS secrets manager
## Other Services
 - AWS tag editor
 - AWS quicksight
 - AWS glue
