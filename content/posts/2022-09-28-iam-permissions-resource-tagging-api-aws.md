---
title: List of IAM permissions required for tagging account wide supported resources via resource tagging API
date: 2022-09-28
---

I was working on a task that requires to update certain tags on all the resources in an AWS account. While there are multiple ways to apporach this problem, I opted to use a AWS native solution [AWS Resource Groups Tagging API](https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/overview.html). It provides a nice simple API to get resources by filters and modify tags on those resources. Another way to achieve this is by putting a [Conformance Pack](https://docs.aws.amazon.com/config/latest/developerguide/conformance-packs.html) with both rules and possibly a custom lambda based remediation. Anyways this post is about resource tagging api.

## Challenge

It sounds easy unless you try to drill down each and every action that different services provides. For example if you want to add tags to a `ec2 instance` then your role should have permission `ec2:CreateTags` assigned. Similary you should have permission `elasticbeanstalk:AddTags` in order to put tags on a `elasticbeanstalk` application. As you can see it can become quite cumbersome and for worst, AWS service authorization does not guarantee any naming conventions.


## Solution

Luckily, one of my coworker found a URL which actually AWS calls itself for their policy simulator UI [https://awspolicygen.s3.amazonaws.com/js/policies.js]. This gives us a map of service prefixes and their respective allowed actions. 

With this data, It becomes very easy to filter out all the tagging related operations for every (fingers crossed) AWS service. I just had to write a little dirty script 

```python
import json
from urllib.request import Request, urlopen
from pprint import pprint

# This is a helper file to generate most of the tagging permissions defined by various AWS sevices.
def get_iam_action_prefixes() -> str:
    """Gets all action prefixes in the IAM policy generator."""
    policies_url = "https://awspolicygen.s3.amazonaws.com/js/policies.js"
    with urlopen(Request(policies_url)) as response:
        body: str = response.read().decode()
    policies_markup = body.split("=")[1]
    policies_doc = json.loads(policies_markup)
    action_prefixes = {
        service["StringPrefix"]: [action for action in service['Actions'] if 'Tag' in action and not(action.startswith(('Describe', 'List', 'Get', 'Delete', 'Remove')))] for service in policies_doc["serviceMap"].values()
    }
    action_prefixes = dict((k, v) for k, v in action_prefixes.items() if v)

    string = f""
    for service, actions in action_prefixes.items():
        for action in actions:
            string = string + f"- '{service}:{action}'" + "\n"

    return string

print(get_iam_action_prefixes())
```

Running this script will output a formatted list of permissions that can be easily pasted into a cloudformation template

```yaml
- 's3:GetObject'
- 'cloudformation:UpdateStack'
- 'comprehend:TagResource'
- 'elasticfilesystem:CreateTags'
- 'elasticfilesystem:TagResource'
- 'glue:TagResource'
- 'iotthingsgraph:TagResource'
- 'evidently:TagResource'
- 'savingsplans:TagResource'
- 'ssm:AddTagsToResource'
- 'ssm:GetParameters'
- 'sso:TagResource'
- 'iot:TagResource'
- 'fis:TagResource'
- 'lambda:TagResource'
- 'mgn:TagResource'
- 'dataexchange:TagResource'
- 'machinelearning:AddTags'
- 'auditmanager:TagResource'
- 'guardduty:TagResource'
- 'events:TagResource'
- 'lex:TagResource'
- 'proton:TagResource'
- 'ram:TagResource'
- 'mediaconnect:TagResource'
- 's3:PutBucketTagging'
- 's3:PutJobTagging'
- 's3:PutObjectTagging'
- 's3:PutObjectVersionTagging'
- 's3:PutStorageLensConfigurationTagging'
- 's3:GetBucketTagging'
- 's3:ReplicateTags'
- 'sagemaker:AddTags'
- 'lakeformation:AddLFTagsToResource'
- 'lakeformation:CreateLFTag'
- 'lakeformation:SearchDatabasesByLFTags'
- 'lakeformation:SearchTablesByLFTags'
- 'lakeformation:UpdateLFTag'
- 'aps:TagResource'
- 'globalaccelerator:TagResource'
- 'profile:TagResource'
- 'forecast:TagResource'
- 'clouddirectory:TagResource'
- 'mediatailor:TagResource'
- 'route53:ChangeTagsForResource'
- 'sts:TagSession'
- 'mediapackage:TagResource'
- 'cassandra:TagResource'
- 'resiliencehub:TagResource'
- 'athena:TagResource'
- 'mobiletargeting:TagResource'
- 'route53domains:UpdateTagsForDomain'
- 'opsworks:TagResource'
- 'codedeploy:AddTagsToOnPremisesInstances'
- 'codedeploy:TagResource'
- 'iam:TagInstanceProfile'
- 'iam:TagMFADevice'
- 'iam:TagOpenIDConnectProvider'
- 'iam:TagPolicy'
- 'iam:TagRole'
- 'iam:TagSAMLProvider'
- 'iam:TagServerCertificate'
- 'iam:TagUser'
- 'route53resolver:TagResource'
- 'workmail:TagResource'
- 'route53-recovery-readiness:TagResource'
- 'iotanalytics:TagResource'
- 'connect:TagResource'
- 'ce:TagResource'
- 'ce:UpdateCostAllocationTagsStatus'
- 'synthetics:TagResource'
- 'elastic-inference:TagResource'
- 'refactor-spaces:TagResource'
- 'gamesparks:TagResource'
- 'sqlworkbench:TagResource'
- 'inspector2:TagResource'
- 'appflow:TagResource'
- 'config:TagResource'
- 'rds:AddTagsToResource'
- 'swf:TagResource'
- 'appsync:TagResource'
- 'acm:AddTagsToCertificate'
- 'ssm-incidents:TagResource'
- 'xray:TagResource'
- 'rum:TagResource'
- 'cloudfront:CreateStreamingDistributionWithTags'
- 'cloudfront:TagResource'
- 'eks:TagResource'
- 'fms:TagResource'
- 'kinesis:AddTagsToStream'
- 'ds:AddTagsToResource'
- 'iotsitewise:TagResource'
- 'codestar-notifications:TagResource'
- 'frauddetector:TagResource'
- 'worklink:TagResource'
- 'codestar-connections:TagResource'
- 'workspaces:CreateTags'
- 'lookoutvision:TagResource'
- 'chime:TagAttendee'
- 'chime:TagMeeting'
- 'chime:TagResource'
- 'elasticache:AddTagsToResource'
- 'iotwireless:TagResource'
- 'firehose:TagDeliveryStream'
- 'storagegateway:AddTagsToResource'
- 'elasticmapreduce:AddTags'
- 'batch:TagResource'
- 'connect-campaigns:TagResource'
- 'iotevents:TagResource'
- 'billingconductor:TagResource'
- 'cloudtrail:AddTags'
- 'dynamodb:TagResource'
- 'es:AddTags'
- 'deepracer:TagResource'
- 'voiceid:TagResource'
- 'emr-containers:TagResource'
- 'schemas:TagResource'
- 'networkmanager:TagResource'
- 'cognito-identity:SetPrincipalTagAttributeMap'
- 'cognito-identity:TagResource'
- 'appconfig:TagResource'
- 'apprunner:TagResource'
- 'license-manager:TagResource'
- 'a4b:TagResource'
- 'acm-pca:TagCertificateAuthority'
- 'states:TagResource'
- 'wisdom:TagResource'
- 'greengrass:TagResource'
- 'redshift:CreateTags'
- 'deepcomposer:TagResource'
- 'managedblockchain:TagResource'
- 'waf:TagResource'
- 'appstream:TagResource'
- 'quicksight:TagResource'
- 'wafv2:TagResource'
- 'cases:TagResource'
- 'dlm:TagResource'
- 'wellarchitected:TagResource'
- 'kendra:TagResource'
- 'ivs:TagResource'
- 'lightsail:TagResource'
- 'cloudsearch:AddTags'
- 'emr-serverless:TagResource'
- 'iotfleetwise:TagResource'
- 'backup:TagResource'
- 'databrew:TagResource'
- 'braket:TagResource'
- 'dms:AddTagsToResource'
- 'network-firewall:TagResource'
- 'ssm-contacts:TagResource'
- 'transcribe:TagResource'
- 'mediapackage-vod:TagResource'
- 'devicefarm:TagResource'
- 'groundstation:TagResource'
- 'signer:TagResource'
- 'resource-groups:Tag'
- 'honeycode:TagResource'
- 'amplifyuibuilder:TagResource'
- 'workspaces-web:TagResource'
- 'ecr-public:TagResource'
- 'snow-device-management:TagResource'
- 'rolesanywhere:TagResource'
- 'elemental-activations:TagResource'
- 'grafana:TagResource'
- 'appmesh:TagResource'
- 'kafka:TagResource'
- 'codeguru-reviewer:TagResource'
- 'codeguru-reviewer:UnTagResource'
- 'memorydb:TagResource'
- 'cloudwatch:TagResource'
- 'autoscaling:CreateOrUpdateTags'
- 'shield:TagResource'
- 'iottwinmaker:TagResource'
- 'secretsmanager:TagResource'
- 'fsx:TagResource'
- 'amplify:TagResource'
- 'kinesisvideo:TagResource'
- 'kinesisvideo:TagStream'
- 'medialive:CreateTags'
- 'geo:TagResource'
- 'kms:TagResource'
- 'cloudhsm:AddTagsToResource'
- 'cloudhsm:TagResource'
- 'ec2:CreateTags'
- 'datapipeline:AddTags'
- 'monitron:TagResource'
- 'rbin:TagResource'
- 'macie2:TagResource'
- 'migrationhub-orchestrator:TagResource'
- 'm2:TagResource'
- 'outposts:TagResource'
- 'gamelift:TagResource'
- 'iotfleethub:TagResource'
- 'route53-recovery-control-config:TagResource'
- 'opsworks-cm:TagResource'
- 'timestream:TagResource'
- 'ivschat:TagResource'
- 'discovery:CreateTags'
- 'codecommit:TagResource'
- 'codeguru-profiler:TagResource'
- 'iotdeviceadvisor:TagResource'
- 'sns:TagResource'
- 'cognito-idp:TagResource'
- 'elasticbeanstalk:AddTags'
- 'elasticbeanstalk:UpdateTagsForResource'
- 'applicationinsights:TagResource'
- 'elasticloadbalancing:AddTags'
- 'lookoutequipment:TagResource'
- 'lookoutmetrics:TagResource'
- 'waf-regional:TagResource'
- 'ecs:TagResource'
- 'ecr:PutImageTagMutability'
- 'ecr:TagResource'
- 'dax:TagResource'
- 'tag:TagResources'
- 'logs:TagLogGroup'
- 'redshift-serverless:TagResource'
- 'backup-gateway:TagResource'
- 'servicecatalog:AssociateTagOptionWithResource'
- 'servicecatalog:CreateTagOption'
- 'servicecatalog:DisassociateTagOptionFromResource'
- 'servicecatalog:TagResource'
- 'servicecatalog:UpdateTagOption'
- 'drs:TagResource'
- 'mq:CreateTags'
- 'nimble:TagResource'
- 'airflow:TagResource'
- 's3-object-lambda:PutObjectTagging'
- 's3-object-lambda:PutObjectVersionTagging'
- 'personalize:TagResource'
- 'cloud9:TagResource'
- 'elemental-appliances-software:TagResource'
- 'detective:TagResource'
- 'transfer:TagResource'
- 'panorama:TagResource'
- 'access-analyzer:TagResource'
- 'app-integrations:TagResource'
- 'finspace:TagResource'
- 's3-outposts:PutBucketTagging'
- 's3-outposts:PutObjectTagging'
- 'mediastore:TagResource'
- 'bugbust:TagResource'
- 'healthlake:TagResource'
- 'iot1click:TagResource'
- 'codepipeline:TagResource'
- 'securityhub:TagResource'
- 'imagebuilder:TagResource'
- 'sqs:TagQueue'
- 'servicediscovery:TagResource'
- 'glacier:AddTagsToVault'
- 'rekognition:TagResource'
- 'mediaconvert:TagResource'
- 'servicequotas:TagResource'
- 'inspector:SetTagsForResource'
- 'robomaker:TagResource'
- 'qldb:TagResource'
- 'codestar:TagProject'
- 'codeartifact:TagResource'
- 'directconnect:TagResource'
- 'datasync:TagResource'
- 'organizations:TagResource'
- 'kinesisanalytics:TagResource'
```

Hope it helps :)