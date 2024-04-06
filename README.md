# aws-devops-professional
## Table of Contents
 - [SDLC automation](#sdlc-automation)
 - [Configuration Management and IaC](#configuration-management-and-iac)
 - [Resilient cloud solutions](#resilient-cloud-solutions)
 - [Monitoring and Logging](#monitoring-and-logging)
 - [Incident and Event Response](#incident-and-event-response)
 - [Security and Compliance](#security-and-compliance)
 - [Other Services](#other-services)
 - [Filling the gap](#filling-the-gap)

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
   - get started (1 & 2): `opswork stacks`(exam required), `opswork for chef automate`, `opswork for puppet enterprise`. `opswork stacks`: a configuration management service that helps you build and operate highly dynamic apps and propagate changes instantly. a stack is a set of layers, instances and related aws resources whose configuration you want to manage together / `time-based instances` & `load-based instances` / Apps / deployments / monitoring / resources / permissions / tags
   - lifecycle events(most important): each layer has a set of five lifecycle events(`setup`(includes `deploy`),`configure`,`deploy`,`undeploy`,`shutdown`(happen when shutting down an instance but before it's terminated completely)), each of which has an associated set of recipes that are specific to the layer. for `configure` events, it occures on all instances when one instance enters or leaves online state; attach or detach Elastic IP to/from an instance; attach or detach ELB to/from a layer. **note**: `configure` happens on all instances, others are instance-specific
   - cloudwatch events integration: (opswork has auto-healing feature, to listen to that change happening, create a rule on eventbridge)
   - summary
## Resilient cloud solutions
 - Lambda
   - versions and aliases: `$LATEST`, versions(code + configuration, has their own ARN) are immutable, and each version can be accessed as well as the `$LATEST`. Aliases are pointers to different versions, which also are mutable and enable `canary` deployment. they have their own ARN and cannot reference other aliases.
   - environment variables: key-value pair, lambda has its own env variables as well, can be encrypted by kms, lambda service key or CMK
   - concurrency: `The default concurrency limit across all functions per region in a given account is 1,000`. can set `reserved concurrency` at the function level. each invocation will trigger a `throttle`(synchronous--> throttleError 429; asynchronous--> retry automatically and then go to DLQ), open a support ticket if you need a higher limit.
     - concurrency and asynchronous invocations: throttleErrors(429) or system errors(500), events will be returned to the queue and try to run the function again up to 6 hrs. the retry intercal increases exponentially from 1 sec to 5 min at maximal.
     - cold starts & provisioned concurrency: first request served by new instances has higher latency than the rest. Using provisioned concurrency to reserve a certain amount of concurrency for the function so that cold start never happen. Application auto scaling can manage concurrency(schedule or target utilization)
     - reserved and provisioined concurrency: reserved-->This represents the maximum number of concurrent instances allocated to your function. When a function has reserved concurrency, no other function can use that concurrency. Configuring reserved concurrency for a function incurs no additional charges. provisioned-->This is the number of pre-initialized execution environments allocated to your function. These execution environments are ready to respond immediately to incoming function requests. Configuring provisioned concurrency incurs additional charges to your AWS account.
   - file systems mounting: lambda function can access EFS file system if they run in a VPC. config lamdba to mount EFS file system to local directory during initialization. must leverage EFS access points. limits: EFS has connection limits and connection burst limit.
   - cross-account file systems mounting: lambda in VPC A needs several permissions -- VPC peering -- EFS file system in VPC B + access point  needs setup a EFS policy to allow account A.
 - API Gateway
   - overview: fully-managed, support websocket, api versioning, different env, authentication & authorization, api keys and handle request throttling, swagger/open api import to quickly define apis, transform and validate requests and responses, generate sdk and api specification, cache api responses.
     - integrations: lambda function, HTTP(internal HTTP API on-prem, ALB), aws service
     - endpoint types: edge-optimized(default, for global clients), regional, private(only be accessed in your VPC).
     - security: user authentication--> IAM roles(internal apps), cognito(external users), custom authorizer; custom domain name https through ACM(aws certificate manager)--> for edge-optimized, certificate must be in `us-east-1`, for regional, certificate must be in the api gateway region; must setup CNAME or A-alias record in route53.
   - stages and deployment: when making changes to api gateway, you have to deploy it to `stages`(as many as you want, like dev, test, prod), each stage has its own config and can roll back to a history deployment.
     - stage variables: like env variables for api gateway, can be used in lambda function ARN, http endpoint, parameter mapping template. use cases: config http endpoints your stages talk to; pass config parameters to lambda through mapping templates. stage variables to passed to the `context` object in the lambda. format: `${stageVariables.variableName}`
     - stage variables & lambda aliases
   - Open API: using API definition as code to define rest apis. import exsiting openapi3.0 spec to api gateway(method,method request,integration request,method response,aws extensions for API gateway and setup every single option), can export current api as openapi spec. using openapi we can generate sdk for our apps.
     - rest api --request validation -- openapi (setup request validation by importing openapi definitions file): `params-only` or `all validator on the post/validation method`  
   - caching: default TTL(300 secs), can be defined `per stage`, possible to override cache settings `per method`, can be encrypted, capacity(0.5GB--237GB), expensive-->use it in prod.
     - cache invalidation: flush entire cache instantly. `header:Cache-Control:max-age=0`(with proper IAM authorization). any client could invalidate the cache if no invalidateCache policy imposed or not choosing `require authorization checkbox in the console`
   - canary deployment: possible to enable canary deployments for any stage(usually prod). metrics and logs separate(for better monitoring). possible to override stage variables. this is blue/green deployment with lambda and api gateway
   - monitoring, logging and tracing:
     - cloudwatch logs: contain request/response body, enabled at stage level, and can override settings on a per api basis
     - x-ray: enable to trace to get some extra info about the requests in the api gateway. x-ray api gateway and lambda gives the full picture.
     - cloudwatch metrics: metrics are stage-specific, possible to enable detailed metrics. `CacheHitCount` & `CacheMissCount`: efficiency of cache. `Count`: total number of requests in a given period. `integrationLatency`: time between api gateway relay a request to the backend and api gateway receives the response from the backend. `latency`: time between api gateway receives a request from client and it returns a response to the client (includes integration latency and other API gateway overhead). 4xx client error, 5xx server error.
     - api gateway throttling: account limit, soft limit (10000 rps), error 429 too many requests. can set stage limit & method limits to improve performance. or define `usage plans` to throttle per customer
     - api gateway errors: 4xx: 400, 403, 429; 5xx: 502(bad gateway, incompatible output from a lambda proxy integration backend or out-of-order invocations due to heavy loads), 503(service unavailable), 504(integration time-out 29 secs)
 - ECS:
   - overview: `ec2 launch type`, launch docker containers=launch `ecs task`, you must provision and maintain the infra(ec2 instances), each ec2 instance must have an ecs agent, aws takes care of containers. `fargate launch type`: no need for managing infra, is serverless, just need to create task definitions. aws run ecs tasks based on cpu/ram you need. to scale, just increase the number of tasks
     - IAM roles: `ec2 instance profile`(only for ec2 launch type): used by ecs agent to talk to ecs, ecr, cloudwatch logs, secrets manager or ssm parameter store. `ecs task role`: defined in the `task definition`, allow each task to have a specific role to talk to other services.
     - load balancer integrations: ALB suits for most cases, NLB for high throughput use cases.
     - data volumes(EFS): works for both ec2 and fargate launch types. used for persistent multi-az shared storage for your containers. **note**: s3 cannot be used as file system.
   - auto scaling: `ecs service auto scaling`: `aws application auto scaling`, three metrics: ecs service average cpu utilization, memory utilization, ALB request count per target. tracking strategies: target tracking(cloudwatch metric), step scaling(cloudwatch alarm), scheduled scaling(predictable changes). ecs service auto scaling!= ec2 auto scaling. `ec2 launch type -- auto scaling ec2 instances`: 1 auto scaling group(based on CPU usage), 2 ecs cluster capacity provider: paired with asg, used to scale infra for ecs tasks automatically.
   - solution architectures: invoked by eventbridge, eventbridge schedule, SQS, intercept `stopped tasks` using eventbridge
   - logging: logging with `awslogs` driver, need to turn on `awslogs` log driver, and config `logConfiguration` parameters in the task definition. `fargate launch type`: task execution role must have required permissions. `ec2 type`: use `cloudwatch unified agent and ecs container agent`, enable `ECS_AVAILABLE_LOGGING_DRIVERS` in `/etc/ecs/ecs.config`. `logging with sidecar container`: used for collecting logs and send to cloudwatch logs
 - ECR: images storages service, access controlled by iam, support vulerability scan,versioning...
 - ECR - extra: lifecycle policies, uniform pipeline(pull images regardless programming languages)
 - EKS: support ec2 and fargate launch types. use case: if k8s already used on-prem or in other cloud. `node types`: `managed node group`, `self-managed nodes`, `aws fargate`. `data volumes`: EBS, EFS, FSx for lustre, FSx for netapp ontap
 - EKS - logging: send logs to cloudwatch logs(control plane logs: API server, audit, authenticator, controller manager, scheduler. can select which ones to send to cloudwatch). `nodes & container logs`: use `cloudwatch agent` to send metrics, use `fluent bit` or `fluentd` to send logs to cloudwatch logs. container logs path: `/var/log/containers`, use `cloudwatch container insights` to get monitoring solution for nodes, pods, tasks and services.
 - Amazon Kinesis: collect, process,and analyze streaming data in real-time.
 - kinesis data streams
   - overview: up to 1 yr retention, can replay, cannot be deleted, ordering(partition key), producers: SDK, kpl, kinesis agent, consumers: kcl, SDK, lambda, firehose, data analytics.
     - provisioned mode: each shard 1mb/s in, 2mb/s out
     - on-demand mode: default 4mb/s in , automatically scale based on throughput peak observed during last 30 days
     - security: iam policies, https, kms, or cmk, vpc endpoints for kinesis to access vpc, monitor api calls using cloudtrail
   - consumers scaling: `GetRecords.IteratorAgeMilliseconds`(cloudwatch metric): the difference between current time and the time when last record was written via `GetRecords` call. `IteratorAgeMilliseconds` > 0, then we are not processing the records fast enough
 - kinesis data firehose: fully-managed , destinations: aws redshift, s3, opensearch; 3rd party; custom http endpoint. near real-time: write data in batch, buffer: 1mb minimal, interval(0 -- 900 secs). support many data format, transformation, or custom transformation using lambda, can send failed or all data to backup s3 bucket
 - kinesis data analytics
   - overview: `sql applications`--> real-time analytics on kds & kdf using SQL, add reference data from s3 to enrich streaming data. fully-managed. output: kds & kdf. use cases: time-series analytics, real-time dashboard, real-time metrics. `for apache flink`(was mks managed streaming kafka)--> use flink(java,scala or sql) to process and analyze streaming data. flink does not read from firehose (use kinesis analytics instead)
   - using ML: `RANDOM_CUT_FOREST`: sql function used for anomaly detection, using recent history to compute model. `HOTSPOTS`: locate and return info about relatively dense regions in your data.
 - Route53
   - overview: fully-managed, authoritative, health check, only service with 100% sla. `records`: type: A/AAAA/CNAME, domain name, value, routing policy, TTL. `hosted zones`(container for records): public hosted zones, private hosted zones(how to route traffic within one or more vpcs). $0.5 per month per hosted zone 
   - routing policies:
     - weighted: control the % of the requests that go to each specific resource. weights don't need to sum up to 100, can be associated with health check, dns records must have the same name and type. assign weight 0 to a record to stop sending traffic. if all records are weight 0, then all records will be returned equally.
     - latency: direct traffic to the least latent for the clients. latency is based on traffic between users and aws regions, can be associated with health check.
     - failover: active-passive
 - RDS read replicas vs multi-AZ:
   - `read replicas for read scalability`: up to 15, replication is async, can be promoted to their own DB having their own lifecycle, apps must update the connection string. `use cases`: create a replica for reporting app or data analytics. only for read operations. `network cost`: same region is free, cross-region is not free.
   - multi-az: disaster recovery. sync replication. one dns name -- automatically failover. no manual intervention, not for scaling. read replica can be setup as multi-az for disaster recovery.
   - from single az to multi az: zero downtime(no need to stop db). click on 'modify' button, and a snapshot is taken, a new db is restored from the snapshot, then synchronization is established between two db instances.
 - Aurora - extra: `auto scaling`: writer endpoint and reader endpoint. `global aurora`: `aurora cross region read replicas` or `aurora global database`(recommended): 1 primary region(read/write), and up to 5 secondary regions(read-only), each of which can has up to 16 read replicas. promoting another region(for disaster recovery) has a RTO of < 1 min. typical cross-region replication takes less than 1 sec
   - unplanned failover: aurora endpoint stored in ssm parameter store, and use a lambda to health check global aurora, then send a notif to admin if it's failed, then admin will promote another region and update the endpoint in the ssm parameter store.
   - global application: each region has a local aurora db and a aurora global shared datasets.
 - elasticCache
   - overview: serverless service, support redis or Memcached, help load off read intensive workloads of db, or make your app stateless. `use case`: db cache, must have an invalidation strategy to make sure only the most current data is used in there. user session store.
     - redis vs Memcached: redis--> data durability, memcached--> pure data caching, multi-threaded architecture
   - redis cluster modes:
     - cluster mode disabled: one shard, all nodes have all the data, one primary node, up to 5 replicas, asynchronous replication, multi-az enabled by default for failover. horizontal scale out/in by adding/removing read replicas. vertical scale up/down the node type. elasticache will internally create a new node group and replicate data to the new group.
     - cluster mode enabled: data is partitioned across shards(helpful to scale write), each shard is the same as `cluster mode disabled`, multi-az capability. up to 500 nodes per cluster(consists of a bunch of shards)
       - auto scaling: increase/decrease the desired shards or replicas. support both target tracking and scheduled scaling policies, only for `cluster mode enabled`. the cloudwatch metric: `elasticachePrimaryEngineCPUUtilization`, and update apps to use the cluster `configuration endpoint`
       - redis connection endpoints: `standalone node`: one endpoint for read and write. `cluster mode disabled cluster`: primary endpoint for all write, reader endpoint for evenly split read across all read replicas, node endpoint for read. `cluster mode enabled cluster`: configuration endpoint for all read/write, node endpoint for read
 - dynamodb
   - overview: full-managed, highly available with replication across multiple az. no maintenance or patching, always available. standard & infrequent access(IA) table class. `basics`: made of tables, each table has a primary key and has an infinite numer of items. each item size is 400kb at maximal. support scalar types, document types, set types. can repidly evolve schemas. `read/write capacity modes`: provisioned mode(default) with auto-scaling feature. on-demand mode: auto scale up/down, more expensive, good for unpredictable workloads
   - advanced features
     - accelerator: fully-managed, highly available, seamless in-memory cache for dynamodb, help solve read congestion, microseconds latency for cached data, 5 min TTL default, no app logic modification. for elasticache, it's good for storing aggregation result.
     - stream processing: ordered stream of item-level modifications in a table. use cases: react to real-time changes, data analytics, cross-region replication... `dynamodb streams`(24 hrs, limited consumers) and `kinesis data streams`(1 yr retention, more consumers,)
     - global tables: two-way replications, active-active replication(read/write in any region), must enable dynamodb stream first
     - TTL: delete items after an expiry timestamp. use cases: reduce stored data by only keeping current items, adhere to regulatory obligations, web session handling...
     - backups for disaster recovery:
       - continuous backups using point-in-time recovery(PITR): optionally enabled for last 35 days, or recover to any time within the backup window, it will crete a new table.
       - on-demand backups: full backups for long-term retention until deleted explicitly, no affect on performance or latency, can be config and managed by aws backup, it will create a new table
     - integration with s3: export to s3(enable PITR)--> works for last 35 days, no affect on read capacity, can perform data analysis, or ETL on top of s3 data, format: json or ion. import from s3--> csv, dynamodb json or ion format, no consume on write capacity, create a new table, any error will be sent to cloudwatch logs
 - AWS DMS
   - overview: quickly and securely migrate databases to aws, resilient, self healing, no affect on source database. support homogeneous migrations and heterogeneous migrations. continuous data replication using CDC(change data capture). an ec2 instance must be running DMS.
     - sources: on-prem dbs, azure sql db, aws rds including aurora, s3, documentdb
     - targets: on-prem dbs, aws rds, redshift, opensearch , kinesis data stream, kafka, nepture, documentdb,redis
     - aws schema conversion tool(sct): convert db schema from one engine to another
     - multi-az deployment: standby replica(synchronously)
   - monitoring:
     - replication task monitoring: task status(task status bar), table state
     - cloudwatch metrics: host metrics, replication task metrics, table metrics 
 - S3 - replication: cross-region replication(crr, compliance, replication across accounts, low latency access), same-region replication(srr, log aggregation, live replication between prod and test accounts). copying is asynchronous, must have iam permissions
 - AWS storage gateway
   - overview: hybrid cloud, unlike EFS/NFS, s3 is a proprietary storage technology, to expose s3 data on-prem, we need aws storage gateway(allow on-prem to use aws cloud s3 data). use cases: disaster recovery, backup&restore, tiered storage. types: file, volume, tape
   - file gateway cache refresh: `RefreshCache api`--> file gateway will automatically update the file cache when user write files to file gateway which will be synced with s3 bucket. or call `RefreshCache` api to refresh the cache for file gateway if a user upload file directly to s3. `Automating cache refresh`--> automatically refresh file gateway periodically
 - Auto scaling groups
   - scaling policies:
     - dynamic scaling: `target tracking scaling`,`simple/step scaling`(with cloudwatch)
     - scheduled scaling: for known usage patterns
     - predictive scaling: continuously forecast load and schedule scaling ahead
     - good metrics to scale on: `CPUUtilization`, `RequestCountPerTarget`, `Average Network In/Out`, `Any custom metric`
     - scaling cooldown: cooldown period(default 300 sec) after a scaling activity, during which asg will allow for metrics to stablize. advice: use ready-to-use ami to reduce configuration time in order to be serving request fasters and reduce the cooldown period.
   - lifecycle hooks: you can perform some actions before an ec2 instance is launched or terminated, can be integrated with eventbridge, sns, sqs
   - event notifications: sns notifications: 4 events for asg: ec2_instance_launch, ec2_instance_launch_error, ec2_instance_terminate, ec2_instance_terminate_error. eventbridge: can have more events on different conditions(successful, failed,cancelled...) 
   - termination policies: `default termination policy`: select az with more instances, terminate one with the oldest launch template or launch configuration, or terminate one with the same launch template that is closest to the next billing hour. `aoolcationStrategy`, `OldestLaunchTemplate`,`OldestLaunchConfiguration`,`ClosestToNextInstanceHour`,`NewestInstance`,`OldestInstance`. **note**: can use one or more policies, just define the evaluation order. also can define custom termination policy backed by lambda function
   - warm pools: scale-out latency strategy by maintaining a pool of pre-initialized instances(running--not cost saving, stopped, hibernated). warm pool not contribute to asg metrics that affect scaling policies. warm pool size: `minimum warm pool size`,`max prepared capacity`,`or max prepared capacity`
     - instance reuse policy: by default, asg will terminate instance when scale in, then launch a new instance in warm pool. while instance reuse policy allow to return instances to the warm pool when scale in
     - warm pool--lifecycle hooks(warmed:pending:wait event)
 - application auto scaling: monitor your apps and automatically adjusts capacity to maintain steady, predictable performance at lowest cost. setup scaling for multiple resources across multiple services from a single place. point to your app and select the service and resource you want to scale(no need alarms and scaling actions for each service). search for resources/services using cloudformation stack,tags,or ec2 asg. build `scaling plans` to automatically add/remove capacity from your resources in real-time as demand changes. support target tracking , step, and scheduled scaling policies.
 - ELB
   - ALB rules deep dive: each rule has a target, support actions(forward,redirect,fixed-reponse), conditions: host-header,request method,path pattern,source IP, http header, query string. `target group weighting`: specify weight for each target group, example:blue/green deployment. distribute the traffic for your apps.
   - extra: `dualStack networking`: allow clients talk with elb using ipv4 and ipv6, support ALB and NLB, can have mixed ipv4 and ipv6 targets in seperate target groups. **note**: az must be added/enabled for instances to receive traffic. `privatelink integration`: for overlapping ip addresses in two vpcs, instead of using vpc peering, we create vpc interface endpoint on one end(vpc1), and connect to NLB at the other end(vpc2)
 - NAT gateway: aws-managed NAT, az-specific, cannot be used by ec2 instances in the same subnet, require an internet gateway, no security group required
   - high availability: resilient in a single az, must create multiple NAT gateways in multiple az.
 - multi-AZ architectures: implicit(s3 except onezone-IA, dynamodb, all aws managed services), explicit(EFS,ELB,ASG,beanstalk,Rds,elasticache,aurora,opensearch,jenkins)
 - blue-green architectures: for an ALB, define two target groups(blue,green), and connect them to the same listener, then switch traffic at once or use weighted TG. or if we have one ALB for one target group, then we can use route53 switch traffic, but that depends on clients ttl cache. if using api gateway, we can use `prod stage canary`, or more granular, we can use lambda alias, no changes to api gateway or client
 - multi-region architectures: with route53: health check-->automated dns failover. monitor an endpoint, monitor other health checks, or monitor cloudwatch alarms. health checks are integrated with cloudwatch metrics.
 - disaster recovery:
   - types: `on-prem -> on-prem`(traditional but expensive), `on-prem -> cloud`(hybrid), `cloud region 1 -> cloud region 2`(fast, inexpensive). `RPO`(recovery point objective),`RTO`(recovery time objective)
   - strategies:
     - backup and restore(High RPO,RTO)
     - pilot light(a small version of app is always running, useful for the critical core, such as db)
     - warm standby(full system is up and running, but at minimum size)
     - hot site approach(very low RTO, very expensive, full production scale is running on aws and on-prem, route53-active active)
   - tips:
     - backup: ebs snapshot, rds automated backup/snapshot,... regular push to s3 and other storage classes. from on-prem using snowball or storage gateway.
     - high availability: route53 to direct from region to region, rds, elasticache, efs multi-az, s3. site-to-site vpn as a recovery from direct connect
     - replication: rds replication, aurora + global db, replication from on-prem db to rds, storage gateway
     - automation: cloudformation, elastic beanstalk to rebuild new env, recover ec2 if cloudwatch alarm triggered, lambda for customized automation.
     - chaos: ex. netflix randomly terminateing ec2 instances
## Monitoring and Logging
 - cloudwatch metrics: metric is a variable to monitor, which belongs to namespaces, a dimension is an attribute of a metric. up to 30 dimensions per metric, metric has timestamps. cloudwatch dashboard of metrics, can create custom metrics.
   - metric streams: near real-time delivery, targets: firehose, or 3rd party services. or option to filter metrics to only stream a subset of them. 
 - cloudwatch custom metrics: use api `PutMetricData` to send your own custom metrics to cloudwatch, can also use dimensions. metric resolution `storageResolution` api parameter: standard: 1 min, hight resolution: 1/5/10/30 sec high cost. **important**: accepts metric data points two weeks in the past and two hours in the future(make sure ec2 instance configured correctly) 
 - cloudwatch anomaly detection: continuously analyze metrics to determine normal baselines and surface anomalies using ML algorithm. a model will be created based on the past data, and show you the values out of the normal range, allow you to create alarms based on metric's expected value(instead of static threshold), can also exclude specified time periods or events from being trained.
 - amazon lookout for metrics: more complete, automatically detect anomalies and identify root cases using ML without manual intervention. can integrate with different aws services (not just cloudwatch)and 3rd party saas apps through appFlow
 - cloudwatch-logs: `log groups`,`log stream`,`log expiration policies`, by default encrypted, or can use kms, logs can be sent to s3, kinesis data streams, kinesis data firehose, lambda, opensearch
   - source: SDK, cloudwatch unified agent, cloudwatch log agent, ecs, lambda, vpc flow logs, api gateway, cloudtrail, route53 dns log
   - insights: cloudwatch logs insights: search and analyze log data, built-in query language, can query multiple log groups in different aws accounts, not real-time
   - s3 export: logs data needs up to 12 hrs to become available for export, api call `CreateExportTask`, not real-time, use logs subscriptions instead.
   - logs subscriptions: get real-time log events with subscription filter, then send logs to kinesis data streams, firehose or lambda.
   - cloudwatch log aggregation multi-account & multi-region
   - cross-account subscription
 - cloudwatch-logs - live tail
 - cloudwath-logs - metric filters: cloudwatch logs can use filter expression, the filter dont retroactively filter data, only publish the metric data points for events that happen after the filter was created. able to specify up to 3 dimensions for the metric filter(optional)
 - all kinds of logs:
   - application logs: produced by app code, written to a local file system. on ec2, use cloudwatch agent to send logs. for lambda, ecs or fargate, or elastic beanstalk, direct integration with cloudwatch logs
   - operating system logs(event logs, system logs): produced by ec2 or on-prem, informing system behavior, using cloudwatch agent to send to cloudwatch logs
   - access logs: list of all the requests for individual files that people have requested from website. usually load balancers, proxies, web servers,etc. aws provides some access logs
   - aws managed logs: ELB access logs-->s3, cloudtrail logs--> s3 and cloudwatch logs, vpc flow logs --> s3 and cloudwatch logs, route53 access logs --> cloudwatch logs, s3 access logs --> s3, cloudfront access logs --> s3
 - cloudwatch agent & cloudwatch logs agent: install cloudwatch logs agent on ec2 or on-prem to send logs to cloudwatch logs. `cloudwatch logs agent` & `unified agent`(newer version, can collect additional system-level metrics such as ram, processes,etc), use ssm parameter store to config centrally
   - unified agent -- metrics: CPU, disk metrics, ram, netstat, processes, swap space
 - cloudwatch alarms: various options(%,min,max...). state: ok, insuffient_data, alarm. period: length of time to evaluate the metric.
   - targets: actions on ec2 instances. ec2 auto scaling. amazon sns
   - composite alarms: monitor multiple states of alarms, using `and` and `or` conditions, help to reduce `alarm noise`
   - ec2 instance recovery: status check(instance status, system status). recovery: same private, public, elastic Ip, metadata, placement group
   - good to know: can create alarms based on metrics filters. to test alarms and notifications, set the alarm state to `Alarm` using cli
 - cloudwatch synthetics canary: a configurable script that monitor your apis, URLs, websites... reproduce what your customers do to find issues before customes are impacted. check the availability and latency of your endpoints and can store the load time data and screenshots of the UI. written in nodejs or python. can access to a chrome browser, can run once or on a regular schedule.
   - blueprint: `heartbeat monitor`,`api canary`,`broken link checker`,`visual monitoring`,`canary recorder`,`gui workflow builder`  
 - amazon Athena: serverless query service to analyze data in s3 using standard SQL language, support csv,json,orc,arvo, and parquet. commonly used with aws quicksight for dashboard. use case: BI/analytics/analyze& query vpc flow logs, elb logs, cloudtrail logs, etc....
   - performance improvement: use `columnar data` for cost-savings(less scan), apache parquet or orc is recommended. use `aws Glue` to transform data to parquet or orc. `compress data` for smaller retrieves. `partition datasets` in s3 for easy querying(only query certain amount of data). `use larger files`(> 128mb) to minimize overhead
   - federated query: can run sql query across sql or non-sql dbs or custom data sources(aws, on-prem). use `data source connectors` that run on aws lambda to run federated queries(such as cloudwatch logs, dynamodb, rds,...), then store the results back in s3.
## Incident and Event Response
 - eventBridge
   - overview: schedule--cron jobs; event pattern--event rules to react to a service doing something; trigger lambda functions, send sqs/sns messages... can optionally filter events. can have default event bus, partner event bus, and custom event bus. a reource-based policies can be applied to event buses. can archive events(all/filter for indefinitely or a period), able to replay archived events for debugging. can infer the event schema, and schema registry allows you to generate code for your app, schema can be versioned. eventbridge uses resource-based policy to manage access. use case: aggregate all events from aws organization in a single aws account or region.
   - content filtering
   - input transformation
 - S3 - event notifications: deliver events in secs or mins. for sns, sqs, lambda, we define resource policy for access management. all s3 events will be sent to eventbridge which has more than 18 destinations of aws services: advanced filtering, multiple destinations, eventbridge capabilities
 - S3 - object integrity: s3 uses `checksum` to validate the integrity of uploaded objects. `using MD5`: user calculated MD5 and add a http header: content-md5 to s3, and s3 will calculate md5 itself and then compare the two. `using md5 & etag`: s3 has an etag for object which is equal to md5 if using sse-s3, then `get object metadata`, compare the etag with local version.
 - AWS health dashboard
   - overview: `service history`(show all regions, all services health, historical info for each day, has a RSS feed to subscribe to), `your account`(provides alerts and remediation guidance when aws is facing events that may impact you, giving personal view into performance and availability of aws services you are using. shows relevant and timely info to help manage events in progress and provides proactive notif to help plan for scheduled activities. can aggregate data from entire aws organization. global service)
   - events & notifications: integrate with eventbridge to react to aws health events in your account(possible for account events--affected resources in your account and public events--regional availability of a service). use cases: send notifs, take corrective action, capture event info, remediation...
 - EC2 instance status checks: `system status checks`(any software/hardware issues on the physical host, check aws health dashboard for any critical maintenance. resolution: stop and start a new instance--instance migrated to a new host), `instance status checks`(software/network config, resolution: reboot the instance or change instance config)
   - status checks -- cloudwatch metrics & recovery:
     - metrics: `statusCheckFailed_system`,`statusCheckFailed_instance`,`statusCheckFailed`(for both)
     - option1: cloudwatch alarm-- recover ec2 instance with the same private/public IP, EIP, metadata, and placement group.
     - option2: asg-- set `min/max/desired` to 1 to recover an instance which will launch a new instance(ips will be different)
 - cloudtrail
   - overview: get a history of events/api calls in your aws account. can put logs from cloudtrail to cloudwatch logs or s3. a trail can be applied to single region or all regions(default).(if a resource is deleted, investigate the cloudtrail first)
     - management events(default): operations happened on resources, can separate `read events` and `write events`
     - data events: by default data events are not logged(high volume operations)
     - cloudtrail insights events(not free): has to enable to detect unusual activity in your account. analyze normal management events to create a baseline, then continuously analyzes write events to detect unusual patterns, the insights events appear in the cloudtrail console, sent to s3 and eventbridge
     - cloudtrail events retention: 90 days in cloudtrail, then log them to s3 and use athena to analyze. 
   - eventBridge integration: cloudtrail events will appear in the eventbridge
 - SQS - dead letter queues: we can set `MaximumReceives` threshold, then message will go to dead letter queue if exceeded. dlq type should be the same with the source queue(fifo--fifo,standard--standard). make sure to process messages in the dlq befire expiry date(good to set retention). `redrive to source`: manual inspection and debugging, once fixed, redrive the message from dlq to the source queue in batches without writing custom code.
 - SNS - redrive policy: for messages not delivered successfully, we set a dlq(sqs, or sqs fifo), and create a `redrive policy`. the dlq is attached to sns subscription-level(not topic level)
 - AWS X-Ray: visual analysis of our apps(distributed system). tracing requests across your microservices. integration with ec2(x-ray agent),ecs(x-ray agent or docker container),lambda, beanstalk(automated agent installed),api gateway(helpful to debug, like 504). need iam permissions to x-ray. 
 - AWS X-Ray with Beanstalk: beanstalk include x-ray daemon, just to enable `x-ray`. make sure instance profile has iam permissions for x-ray daemon to function correctly. make sure app code has x-ray sdk. **note**: x-ray daemon not provided for multi-container
 - AWS Distro for openTelemetry: vendor-agnostic, open-source, production-ready aws-supported distributed tracing framework. collect distributed traces and metrics from your apps, and collect metadata from your aws resources and services. `auto-instrumentation agents` to collect traces without changing your code. can send to x-ray, cloudwatch, prometheus... migrate from x-ray to openTelemetry if for standard apis or send traces to multiple destinatioins simultaneously.
## Security and Compliance
 - AWS config
   - overview: help audit and record compliance of your aws resources. can receive alerts for any changes. per-region service. can be aggregated across regions and accounts. possible to storing config data in s3
     - rules: managed rules or custom rules. can be evaluated or triggered for any change or at regualr intervals. rules dont prevent actions from happening. no free tier.
     - resource: view compliance, config, cloudtrail api calls of resources over time.
     - remediations: can trigger ssm automation document to remediate non_compliant issues.
     - notifications: use eventbridge to trigger notifs when non_compliant. use sns to send notif when config or conpliance changes
   - configurations recorder and aggregator: `recorder`: store configurations of aws resources as `configuration item` which is a point-in-time view of all attributes of an aws resource. created whenever a change detected. the recorder only record the resource types you specify. must created before aws config can track(when using aws cli or console to enable aws config, created automatically). `aggregator`: created in one central aggregator account. aggregator rules, resources, etc... across multiple accounts and regions. if using aws organization, no need for individual authorization. rules are created in each individual source aws account. can deploy rules using cloudformation stacksets to multiple targets
   - conformance packs: collection of aws config rules and remediation actions. in yaml format, similar to cloudformation. deploy to an aws account and regions or across an aws organization. pre-built sample or custom packs. can include `custom config rules` backed by lambda to evaluate  the compliance. can use `ssm parameter store`. can designate a delegated admin to deploy conformance packs to your aws organization(a member account). can be integrated into cicd
   - organizational rules: conformance rules-->accounts and organization. organizational rules-->only for aws organization, and managed at organizational level, one rule at a time vs many rule at a time(conformance packs)
 - AWS organizations
   - overview: `OrganizationAccountAccessRole` has full permissions in the member account to the management account, used to perform admin tasks in the member account(e.g. create iam users), can be assumed by iam users in the management account. automatically added to all new member accounts created in the aws organization, but must created this role manually if to invite an existing member account.
     - multi account strategies: create accounts based on envs, or departments, or regulatory restrictions(using SCP) / multi account vs one account multi vpc / using tagging standards for billing purposes / enable cloudtrail on all accounts to send logs to central s3 account / send cloudwatch logs to central logging account / create an account for security.
     - feature modes: `consolidated billing features`: single payment method for all accounts, pricing benefits from aggregated usage(volume discount for ec2, s3...). `all eatures`(default): include consolidated billing, scp. invited accounts must approve enabling all features, able to apply scp to prevent member accounts from leaving the org. cannot switch back to `consolidated billing feature only`
     - reserved instances: `consolidated billing feature` treat all accounts as one account, all accounts can benefit from reserved instances purchased by any other account, the payer account(management account) can turn off the sharing of reserved instance discount and saving plans discount. and to share the reserved instance or saving plan discount with an account, both accounts must have sharing turned on.
     - moving account between orgs: first remove member account, then invite it from another org, then accept the invitation.  
   - service control policy (SCP): allow or deny iam actions. applied to OU or Account level. does not apply to management account. scp is applied to all users or roles in the account, including root user. does not affect service-linked roles which used for other aws service to talk with aws org. scp must have explicit `allow` to allow some actions
 - AWS control tower
   - overview: runs on top of aws organization. setup and govern a secure and compliant multi-account aws env based on best practice. automate setup envs, policy management using guardrails, detect policy violations and remediate them, monitor compliance through a dashboard.
     - account factory: automate account provisioning and deployments using aws service catalog, enables you to create pre-approved baseline and config options for aws accounts in your org.
     - detect and remediate policy violations: `guardrail`: preventive(using scp), detective(using aws config)
     - guardrails levels: `mandatory`(enforced by aws control tower), `strongly recommended`(based on aws best practice ,optional),`elective`(commonly used by enterprise, optional)
   - landing zones: automatically provision secure, compliant, multi-account env based on aws best practices. it consists of: aws organization, account factory, organizational units, service control policies(scp), iam identity center, guardrails(rules and policies), aws config(monitor and assess resources compliance with guardrails)
   - account factory & migrating accounts
     - account factory customization(AFC): automatically customize resources in new and existing accounts created through account factory. `custom blueprint`: cloudformation template with resources and configs used to customize accounts. defined in the form of service catalog product. recommend to create a `hub account` to store all custom blueprints. only one blueprint can be deployed to the account. each time a new account created, an event will be sent to eventbridge.
     - migrate an aws account to control tower: first move the account to the unregistered OU, then create iam role(awsControlTowerExecution), then create conformance packs, then evaluate compliance results of config conformance packs, then remove config delivery & config recorder created during evaluation process, then move into target OU and setup control tower successfully.
   - customizations for AWS control tower (CFCT): gitops-style customization framework created by aws, helps add customization to your landing zone using custom cloudformation templates and scps. automatically deploy resources to new aws accounts created using account factory.**note**: cfct is different from afc(account factory customization), blueprint
   - aws config integration: using aws config to implement `detective guardrails`, automatically enabled aws config in enabled regions. aws config configuration history and snapshots sent to s3 bucket in a centralized log archive account. control tower uses cloudformation stacksets to create resources like config aggregator,...
     - `aws config conformance packs`: a set of config compliance rules & remediations. for example, when a new account created in control tower, an event will be sent to eventbridge which can be used to trigger a lambda to create stacksets and deploy conformance packs into the account.
   - account factory for terraform(AFT): help provision and customize aws accounts in control tower through terraform using a deployment pipeline. create `account request terraform file` to trigger aft workflow for account provisioning. `built-in feature options`(disabled by default): `aws cloudtrail data events`,`aws enterprise support plan`,`delete the aws default vpc`. terraform module maintained by aws. works with terraform open-source, terraform enterprise, and terraform cloud.
 - IAM identity center
   - overview: successor to aws single sign-on: one login for all aws accounts in aws organization. identity providers: built-in identity store in iam identity center, 3rd party: active directory, okta...
     - permission sets assigned to users and groups
     - similar to other apps, such as salesforce, slack, micro 365, or some custom apps
     - attribute-based access control(ABAC): fine-grained permissions based on users' attributes. define permission once, then change the value of attributes of users.
   - extra: `external identity providers`: saml2.0, you must create users and groups in iam identity center that are identical to the users and groups in the external identity providers, because saml2.0 cannot query the idP to learn about users and groups. thus we can use `SCIM`(system for cross-domain identity management): automatic provisioning(synchronization) of users identities from an external idp into iam identity center. must be supported by external idp, and perfect complement to using smal2.0
   - attribute-based access control: fine-grained permissions based on users' attributes stored in iam identity center identity store. user attributes can be used in permission sets and resource-based policy. user attributes are mapped from idp as key-value pairs
   - multi-factor authentication: every time they sign-in(always on), or only when their sign-in context changes(context-aware)
 - AWS web application firewall (WAF): protect web apps from common web exploits (layer 7). on ALB(localized rules), on Api gateway(rules running at the regional or edge level), on cloudfront(rules globally on the edge locations), on Appsync(protect your graphql apis). not for ddos protection(using aws shield). define web ACL(access control list): include ip addresses, http headers, body, or url strings, protect common attack--sql injection, xss. size constraints, geo match, rate-based rule, rule actions: count | allow|block|captcha
   - managed rules: over 190 managed rule, ready-to-use rules. `baseline rule groups`: general protection from common threats, `use-case specific rule groups`,`ip reputation rule groups`,`bot control managed rule group`
   - web ACL logging: can send logs to cloudwatch logs (5mb per sec), s3 bucket(5 min interval), kinesis data firehose(limited by firehose quotas)
   - solution architecture -- enhance cloudfront origin security with aws waf & aws secrets manager: we can setup aws waf at cloudfront with web acl(some protection), then from cloudfront, we can create `custom HTTP header`(such as x-origin-verify:xxxxxxxx), then setup aws waf in front of our ALB to filter the http header. the http header string can be managed by secret manager with auto-rotate and a lambda used to update the custom http header string.
 - AWS firewall manager
   - overview: manage rules in all accounts of aws organization. rules are applied to new resources as they are created across all accounts or future account in the organization.
     - security policy: `waf rules`, `aws shield advanced`,`security groups`, `aws network firewall(vpc level)`, `route53 resolver DNS firewall`, policies are created at the region level.
     - aws waf, firewall manager, shield
   - policies
     - aws waf: enforce web acls to all albs in all account of aws organization. identify resources that dont comply, but do not auto remediate them. or auto remediate any non-compliant resources
     - shield advanced: enforece shield advanced protections to all accounts in the aws org. option to view only compliance or auto remediate
     - security groups: common(applying sgs to all ec2 instances in all accounts in the aws org). auditing(check and manage sgs rules in all accounts in the aws org). usage audit(monitor unused and redundant sgs and optionally perform cleanup)
     - network firewall(all kinds of traffic, layer3): centrally manage network firewall in all accounts in aws org. `distributed`(create and manage firewall endpoint in each vpc), `centralized`(create and manage firewall endpoint in a centralized vpc), `import existing firewalls`(using `Resource sets`)
     - route53 resolver dns firewall: manage associations between resolver dns firewall rule groups and vpcs in all accounts in aws org
 - Amazon guardduty
   - overview: threats discovery service using ML(30 days trial). input data: vpc flow logs, dns logs, cloudtrail events logs. can setup eventbridge to get notified. can protect against `cryptoCurrency` attacks (has a dedicated finding for it)
   - advanced:
     - multi-account strategy: admin account send invitation through guardduty to member accounts. and admin account can add/remove member account, manage guardduty within the associated member accounts, manage findings, suppression rules, trusted ip lists, threat lists. also you can specify a member account as a delegated admin for guardduty.
     - findings automated response: potential security issues in the aws account, any findings will be sent to eventbridge as events, from which we can send to sqs, sns, lambda. events are published to the admin account and the member account that its orginated from
     - findings: guardduty pulls independent streams of data from cloudtrail logs, vpc flow logs, dns logs or eks logs. each finding has a severity value (0.1 -- 8+)(high, medium, low). can generate sample findings in guardduty to test your automations. naming convention: threatPurpose, resourceTypeAffected, threatFamilyName, detectionMechanism(TCP,udp),Artifact
     - findings types: ec2 finding types, iam finding types, kubernetes audit logs finding types, malware protection finding types, rds protection finding types, s3 finding types
     - architectures
     - trusted and threat ip lists: only for public ip address, only guardduty admin account can manage those lists(trusted IP lists and threat IP lists)
   - cloudformation integration: can enable guardduty using cloudformation template. but it will fail if guardduty already enabled. to solve it, use cloudformation custom resource(lambda) to conditionally enable guardduty if it is not enabled, and we can deploy this stack set to all organization.
 - amazon detective: analyze and investigate and identify the root cause of security issues or suspicious activities using ML and graphs. automatically collects and process events from vpc flow logs, cloudtrail, and guardDuty and create a unified view. produce visualizations with details and context to get to the root cause.
 - amazon inspector
   - overview: security accessments. for ec2 instance, using ssm agent to analyze network accessibility, os known vulnerabilities. for container images push to aws ecr, scan images, for lambda function, scan software vulnerabilities, code, packages. then send reports to aws security hub and send findings to eventbridge. a risk score is associated with all vulnerabilities for priorirization. continuous scanning infra only when needed.
   - EC2 setup: having ssm agent running in the ec2 instance with iam role or default host management config for ssm, outbound 443 to ssm endpoint. then aws inspector is evaluating the inventory results in the ssm.
 - EC2 instance migration using AMIs: create an AMI, then launch/restore a new ec2 instance in another az from the AMI.
   - cross-account AMI sharing: sharing AMI does not affect the ownership of the AMI. can only share AMIs that have unencrypted volumes and encrypted volumes with cmk. if you share AMI with encrypted volumes, you must also share cmk to the other side and iam permissions.
   - cross-account AMI copy: to copy AMI shared with you, you are the owner of the target AMI in your account. the owner must grant the read permission for the storage backing the AMI(ebs snapshot). if shared AMI has encrypted snapshots, then the owner must share the key with you as well. can encrypt AMI with your own cmk while copying.
 - AWS trusted advisor
   - overview: no need to install anything--high level aws account assessment. 6 categories: cost optimization, performance, security(default),fault tolerance,service limits(default),operational excellence. upgrade to `business` or `enterprise` support plan to unlock all checks. 
   - architectures: integrated with eventbridge to monitor some resources such as usage of ec2 instance, or some service quotas (50+ service quotas)
 - AWS secrets manager: newer service, meant for storing secrets. able to force rotation of secrets every x days. automate generation of secrets on rotation(lambda). secrets encrypted using kms, integrated with rds, aurora. mostly meant for rds integration.
   - multi-region secrets: read replicas synced with primary secret. use cases: disaster recovery, multi-region apps, multi-region db...
## Other Services
 - AWS tag editor: allow to manage tags of multiple resources at once. add/update/delete. search tagged/untagged resources in all aws regions
 - AWS quicksight: serverless ML BI service to create interactive dashboards. fast, automatically scalable, embeddable, with per-session pricing. use cases: data visualization, business analytics, ad-hoc analysis, get business insight. integrated with rds, aurora, athena, redshift, s3...
 - AWS glue: managed ETL service, fully serverless service.
   - convert data into parquet format(columnar type, which is good for athena).
   - glue data catalog: catalog of datasets-- glue data crawler to write metadata of data sources into glue data catalog tables(metadata), then athena, redshift spectrum, emr will leverage it
   - things to know:
     - `glue job bookmarks`(prevent re-processing old data).
     - `glue elastic views`: combine and replicate data across multiple data stores using SQL, no custom code, glue monitor for changes in the source data
     - `glue databrew`: clean and normalize data using pre-built transformation
     - `glue studio`: new gui to create, run and monitor etl jobs in glue
     - `glue streaming etl`(built on apache spark structured streaming): compatible with kinesis data streaming, kafka, MSK(aws managed kafka)

## Filling the gap
 - udemy assessment:
   - for aws cloudtrail, it can monitor api activity and generate alerts based on suspicious or unauthorized activity. But it may not provide sufficient visibility into the application and infrastructure performance.
   - While X-Ray can be used to troubleshoot distributed application, it may not provide sufficient visibility into the infrastructure and can be limited to specific application types. It must be paired with CloudWatch to provide visibility into the application and infrastructure, and enable timely detection and resolution of potential issues.
   - While CPU utilization monitoring can provide insights into the health of instances, it does not actively perform health checks or ensure high availability and fault tolerance.
   - aws elasticache cluster serves as a data cache layer, not a data layer.
   - Set up an Amazon EventBridge rule to trigger a series of AWS Lambda functions orchestrated by AWS Step Functions, which is a highly efficient and scalable approach to handle high-volume event streams. because step function is automatic scaling.
   - Using AWS Systems Manager Automation documents enables the automation of remediation actions based on the noncompliant resource findings from AWS Config rules.
   - Set up IAM Identity Providers to establish trust relationships with external identity providers, which enables users to log in using their existing credentials, facilitating centralized access control across multiple AWS accounts and external applications.
   - Configure Amazon Kinesis Data Streams to capture and process incoming events, with AWS Lambda processing functions consuming the events for distributed event-driven processing. This approach provides scalability, fault tolerance, and efficient distribution of events to multiple downstream services. Amazon Kinesis Data Streams is purpose-built for handling high volumes of streaming data and enables real-time processing and analysis.
   - User data scripts provide a way to automate the configuration process during instance initialization. This allows for consistent installation and configuration of agents as instances are launched.
   - AWS Systems Manager Distributor enables controlled deployment of software packages, including agents, across multiple instances, allowing for efficient installation and configuration.
   - Use AWS OpsWorks to automatically install and configure agents on the EC2 instances based on a predefined configuration. This allows for centralized management and automation of the agent installation process.
   - API Gateway is a good option to create API's for event triggers, Lambda is used for processing as it takes care of infrastructure management, and Fargate ia best suited for container orchestration.
   - when it comes to fail to deploy lambda function due to a permission error, we can check cloudwatch logs for the lambda as well as iam role for the lambda function. but please note that the lambda function resource policies is for accessibility not for deployment purpose.
   - Cookbooks can modify the configuration and the state of any system configured as a node on Chef infrastructure and so can be used to automate the configuration and deployment of applications and software on servers. It is primarily used for configuration management. CodePipeline is a continuous delivery service that can be used to model, visualize, and automate deployment and so can be used to automate the build, test, and deployment of code changes.
   - Systems Manager's State Manager allows you to define and automate the configuration of your EC2 instances, ensuring that they are all configured according to your organization's security standards. **note**: AWS Config provides a detailed inventory of your AWS resources and can track configuration changes over time. However, it does not provide direct support for configuring instances to meet compliance requirements.
   - Use AWS CloudTrail logs for comprehensive visibility of your AWS environment activities. Implement Amazon GuardDuty for continuous threat detection and use AWS Lambda functions to trigger real-time responses to identified threats.
   - Use CloudWatch Logs for centralized log collection and set up alarms on custom metrics, along with AWS X-Ray for analyzing application traces for deeper insights
   - Use a combination of latency, request count, and CPU utilization metrics for dynamic scaling, which provides an accurate representation of the application's performance, enabling dynamic scaling based on traffic patterns. This approach can efficiently handle traffic spikes and ensure consistent application performance.
   - To deploy an application using the AWS Elastic Beanstalk environment, the application code should be uploaded first and the configuration options specified. Create an Elastic Beanstalk environment, upload the application code, and then specify the configuration options
   - The design phase typically involves the creation of user stories, wireframes, and high-level architecture designs. During this phase, the requirements gathered during the analysis phase are used to create the design of the application.
   - 

 - practice exam #1
   - you can install patches on a regular basis by scheduling patching to run as a Systems Manager maintenance window task. Systems Manager supports an SSM document for Patch Manager, AWS-RunPatchBaseline, which performs patching operations on instances for both security-related and other types of updates. When the document is run, it uses the patch baseline currently specified as the "default" for an operating system type. The AWS-ApplyPatchBaseline SSM document supports patching on Windows instances only and doesn't support Linux instances. For applying patch baselines to both Windows Server and Linux instances, the recommended SSM document is AWS-RunPatchBaseline.
   - When you perform some operations using the AWS Management Console, Amazon S3 uses a multipart upload if the object is greater than 16 MB in size. In this case, the checksum is not a direct checksum of the full object, but rather a calculation based on the checksum values of each individual part. For example, consider an object 100 MB in size that you uploaded as a single-part direct upload using the REST API. The checksum in this case is a checksum of the entire object. If you later use the console to rename that object, copy it, change the storage class, or edit the metadata, Amazon S3 uses the multipart upload functionality to update the object. As a result, Amazon S3 creates a new checksum value for the object that is calculated based on the checksum values of the individual parts.
   - Amazon EventBridge events for AWS Glue can be used to create Amazon SNS alerts, but the alerts might not be specific enough for certain situations. To receive SNS notifications for certain AWS Glue Events, such as an AWS Glue job failing on retry, you can use AWS Lambda. You can create a Lambda function to do the following: 1)Check the incoming event for a specific string. 2)Publish a message to Amazon SNS if the string in the event matches the string in the Lambda function.
   - The AWS Config Auto Remediation feature automatically remediates non-compliant resources evaluated by AWS Config rules. You can associate remediation actions with AWS Config rules and choose to execute them automatically to address non-compliant resources without manual intervention. You must have AWS Config enabled in your AWS account. The AutomationAssumeRole in the remediation action parameters should be assumable by SSM. The user must have pass-role permissions for that role when they create the remediation action in AWS Config, and that role must have whatever permissions the SSM document requires.
   - cloudwatch agent cannot deliver ec2 instance logs to s3. cloudwatch agent and cloudtrail cannot deliver ec2 logs and api logs from ec2 instances to kinesis data stream.
   - One way to verify the integrity of your object after uploading is to provide an MD5 digest of the object when you upload it. If you calculate the MD5 digest for your object, you can provide the digest with the PUT command by using the Content-MD5 header. After uploading the object, Amazon S3 calculates the MD5 digest of the object and compares it to the value that you provided. Or The entity tag (ETag) for an object represents a specific version of that object. Keep in mind that the ETag reflects changes only to the content of an object, not to its metadata. If only the metadata of an object changes, the ETag remains the same. For objects where the ETag is the Content-MD5 digest of the object, you can compare the ETag value of the object with a calculated or previously stored Content-MD5 digest.
   - (#8--unfinished)CloudFormation StackSets simplify the configuration of cross-account permissions and allow for the automatic creation and deletion of resources when accounts are joined or removed from your Organization.
   - With Amazon CloudWatch cross-account observability, you can monitor and troubleshoot applications that span multiple accounts within a Region. Seamlessly search, visualize, and analyze your metrics, logs, and traces in any of the linked accounts without account boundaries. The shared observability data can include the following types of telemetry: 1. Metrics in Amazon CloudWatch 2. Log groups in Amazon CloudWatch Logs 3. Traces in AWS X-Ray. AWS recommends that you use Organizations so that new AWS accounts created later in the organization are automatically onboarded to cross-account observability as source accounts. This is our use case requirement and hence choosing Organizations to implement the requirements.
   - This option suggests using EC2 Image Builder to create an updated custom AMI with AWS Systems Manager Agent included. The Auto Scaling group is configured to attach the `AmazonSSMManagedInstanceCore` role to the instances, thereby enabling centralized management through Systems Manager, as it grants the EC2 instances the permissions needed for core Systems Manager functionality. Session Manager can be used for secure logins, and session details can be logged to Amazon S3. Additionally, an S3 event notification can be set up to alert the security team about new file uploads using Amazon SNS. This option aligns well with the requirement for centralized access and monitoring.
   - In some cases, a Blue/Green deployment fails during the AllowTraffic lifecycle event, but the deployment logs do not indicate the cause for the failure. This failure is typically due to incorrectly configured health checks in Elastic Load Balancing for the Classic Load Balancer, Application Load Balancer, or Network Load Balancer used to manage traffic for the deployment group. To resolve the issue, review and correct any errors in the health check configuration for the load balancer.
   - Configure Amazon Route 53 to point to API Gateway APIs in Europe and Asia-Pacific regions. Use latency-based routing and health checks in Route 53. Configure the APIs to forward requests to an AWS Lambda function in the respective Regions. Setup the Lambda function to retrieve and update the data in a DynamoDB global table.
   - when you create a role, you create two policies: a trusted policy and a permission policy. to assume a role from a different account, your aws account must be trusted by the role. the trust relationship is defined in the role's trust policy when the role was created. so in this case, Establish a trust relationship in the application account's deployment IAM role for the centralized DevOps account, allowing the sts:AssumeRole action. Also, grant the application account's deployment IAM role the necessary access to the EKS cluster. Additionally, configure the EKS cluster aws-auth ConfigMap to map the role to the appropriate system permissions.
   - If you create a customer-managed key in a different account than the Auto Scaling group, you must use a grant in combination with the key policy to allow cross-account access to the key. This is a two-step process (refer to the image attached): 1.The first policy allows Account A to give an IAM user or role in the specified Account B permission to create a grant for the key. However, this does not by itself give any users access to the key. 2.Then, from Account B which contains the Auto Scaling group, create a grant that delegates the relevant permissions to the appropriate service-linked role. The Grantee Principal element of the grant is the ARN of the appropriate service-linked role. The key-id is the ARN of the key.
   - Launch a CloudFormation stack that deploys all the resources of the stack. Add a Retain attribute to the deletion policy of each of these resources. Delete the original stack. Create a new stack with a different name and import the resources that were retained from the original stack. Remove the Retain attribute from the stack to revert to the original template
   - Attribute-based access control (ABAC) is an authorization strategy that defines permissions based on attributes. In AWS, these attributes are called tags. You can attach tags to IAM resources, including IAM entities (users or roles), and to AWS resources. You can create a single ABAC policy or a small set of policies for your IAM principals.
   - Use AWS CodeDeploy with a deployment type configured to Blue/Green deployment configuration. To terminate the original fleet after two hours, change the deployment settings of the Blue/Green deployment. Set Original instances value to Terminate the original instances in the deployment group and choose a waiting period of two hours. In AWS CodeDeploy Blue/Green deployment type, for deployment groups that contain more than one instance, the overall deployment succeeds if the application revision is deployed to all of the instances. The exception to this rule is that if deployment to the last instance fails, the overall deployment still succeeds. This is because CodeDeploy allows only one instance at a time to be taken offline with the CodeDeployDefault.OneAtATime configuration (If you don't specify a deployment configuration, CodeDeploy uses the CodeDeployDefault.OneAtATime deployment configuration). If you choose Terminate the original instances in the deployment group: After traffic is rerouted to the replacement environment, the instances that were deregistered from the load balancer are terminated following the wait period you specify.
   - AWS WAF doesn't support encryption for AWS Key Management Service keys that are managed by AWS. You can enable logging AWS WAF web ACL traffic, to get detailed information about traffic that is analyzed by your web ACL. Logged information includes the time that AWS WAF received a web request from your AWS resource, detailed information about the request, and details about the rules that the request matched. You can send your logs to an Amazon CloudWatch Logs log group, an Amazon Simple Storage Service (Amazon S3) bucket, or an Amazon Kinesis Data Firehose. Your bucket names for AWS WAF logging must start with aws-waf-logs- and can end with any suffix you want.
   - Use Serverless Application Model (SAM) and leverage the built-in traffic-shifting feature of SAM to deploy the new Lambda version via CodeDeploy and use pre-traffic and post-traffic test functions to verify code. Rollback in case CloudWatch alarms are triggered.
   - 
























































