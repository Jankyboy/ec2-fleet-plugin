# EC2 Fleet Plugin for Jenkins

The EC2 Fleet Plugin scales your [Auto Scaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html), [EC2 Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet.html), or [Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html) 
automatically for your Jenkins workload.

[![Build Status](https://ci.jenkins.io/buildStatus/icon?job=Plugins/ec2-fleet-plugin/master)](https://ci.jenkins.io/blue/organizations/jenkins/Plugins%2Fec2-fleet-plugin/activity) [![](https://img.shields.io/jenkins/plugin/v/ec2-fleet.svg)](https://github.com/jenkinsci/ec2-fleet-plugin/releases)
[![Gitter](https://badges.gitter.im/ec2-fleet-plugin/community.svg)](https://gitter.im/ec2-fleet-plugin/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
[![Jenkins Plugin Installs](https://img.shields.io/jenkins/plugin/i/ec2-fleet.svg?color=blue)](https://plugins.jenkins.io/ec2-fleet)

* [Jenkins Page](https://wiki.jenkins.io/display/JENKINS/Amazon+EC2+Fleet+Plugin)
* [Report Issue](https://github.com/jenkinsci/ec2-fleet-plugin/issues/new)
* [Overview](#overview)
* [Features](#features)
* [Change Log](#change-log)
* [Usage](#usage)
  * [Setup](#setup)
  * [Scaling](#scaling)
  * [Groovy](#groovy)
  * [Preconfigure Agent](#preconfigure-agent)
  * [Label Based Configuration (beta)](docs/LABEL-BASED-CONFIGURATION.md)
  * [Windows Agent](docs/SETUP-WINDOWS-AGENT.md)
  * [Configuration As Code](docs/CONFIGURATION-AS-CODE.md)
* [Development](#development)

# Overview

The EC2 Fleet Plugin scales your [Auto Scaling Group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html), [EC2 Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet.html), or [Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html)
automatically for your Jenkins workload. It handles launching new instances that match the criteria set in your ASG, EC2 Fleet, or Spot Fleet e.g. [allocation strategy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-mixed-instances-groups.html#allocation-strategies), 
and terminating idle instances that breach that criteria or those in your Jenkins Cloud configuration.

> [!WARNING]  
> AWS strongly discourages using the SpotFleets because it is now categorized as legacy API with no planned investment. 
Use Auto Scaling Groups instead.

# Features
Minimum Jenkins version: 2.479.3

> [!NOTE]
> [Jenkins version 2.403](https://github.com/jenkinsci/jenkins/releases/tag/jenkins-2.403) includes [significant changes to cloud management](https://issues.jenkins.io/browse/JENKINS-70729).
> If you are using that version, and see unexpected behavior, create an issue and let us know. 

- Supports EC2 Fleet, Spot Fleet, or Auto Scaling Group as Jenkins Workers
- Supports all features provided by EC2 Fleet, Spot Fleet, or Auto Scaling Groups e.g. [multiple instance types across Spot and On-Demand](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-mixed-instances-groups.html#allocation-strategies)
- Auto resubmit failed jobs caused by Spot interruptions
- No delay scale up strategy: enable ```No Delay Provision Strategy``` in configuration
- Add tags to EC2 instances used by plugin, for easy search, tag format ```ec2-fleet-plugin:cloud-name=<MyCloud>```
- Allow custom EC2 API endpoint
- Auto Fleet creation based on Job label ([details](docs/LABEL-BASED-CONFIGURATION.md))
- Set a maximum total uses to terminate nodes after running the set number of jobs
- Set a minimum spare size to keep nodes ready for incoming jobs, even when idle
- Default unique cloud names for UI users (shown as a suggestion in the form) and JCasC users (`name: ""` will signal the plugin to generate a default name)

## Comparison to EC2-Plugin

[EC2-Plugin](https://plugins.jenkins.io/ec2/) is a similar Jenkins plugin that will request EC2 instances when excess workload is
detected. The main difference between the two plugins is that EC2-Fleet-Plugin uses ASG, EC2 Fleet, and Spot Fleet to request and manage
instances instead of doing it manually with EC2 RunInstances. This gives EC2-Fleet-Plugin all the benefits of ASG, EC2 Fleet, and Spot Fleet:
allocation strategies, automatic availability zone re-balancing (ASG only), access to launch templates and launch configurations
, instance weighting, etc. 
See [which-spot-request-method-to-use](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html#which-spot-request-method-to-use).

| EC2-Fleet-Plugin                                                   | EC2-Plugin                                    |
|--------------------------------------------------------------------|-----------------------------------------------|
| Supports On-Demand & Spot Instances                                | Supports On-Demand & Spot Instances           |
| Scales with ASG, EC2 Fleet, or Spot Fleet                          | Scales with RunInstances                      |
| ASG, EC2 Fleet, and Spot Fleet Allocation Strategies               | No Allocation Strategies                      |
| Use launch template/config to set instance settings                | Manually set instances settings within plugin |
| Custom instance weighting                                          | No custom instance weighting                  |
| Supports mixed configuration like instance types, purchase options | Supports single instance type only            |

# Change Log

This plugin is using [SemVersion](https://semver.org/) which means that each plugin version looks like 
```
<major>.<minor>.<bugfix>

major = increase only if non back compatible changes
minor = increase when new features
bugfix = increase when bug fixes
```

As a result, you can safely update the plugin to any version until the first number is different than what you have.

Releases: https://github.com/jenkinsci/ec2-fleet-plugin/releases

# Usage

## Setup

#### 1. Create AWS Account

Go to [AWS account](http://aws.amazon.com/ec2/) and follow instructions.

#### 2. Create IAM User

Specify ```programmatic access``` during creation and record the credentials. These will
be used by Jenkins EC2 Fleet Plugin to connect to your EC2 Fleet or Spot Fleet.

*Alternatively, you may use [AWS EC2 instance roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)*

#### 3. Configure User permissions

Add an inline policy to the IAM user or EC2 instance role to allow it to use EC2 Fleet, Spot Fleet, and Auto Scaling Group.
[AWS documentation about this](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet-requests.html#spot-fleet-prerequisites)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSpotFleetInstances",
        "ec2:ModifySpotFleetRequest",
        "ec2:CreateTags",
        "ec2:DescribeRegions",
        "ec2:DescribeInstances",
        "ec2:TerminateInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeSpotFleetRequests",
        "ec2:DescribeFleets",
        "ec2:DescribeFleetInstances",
        "ec2:ModifyFleet",
        "ec2:DescribeInstanceTypes"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "autoscaling:UpdateAutoScalingGroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:ListInstanceProfiles",
        "iam:ListRoles"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": [
        "*"
      ],
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": [
            "ec2.amazonaws.com",
            "ec2.amazonaws.com.cn"
          ]
        }
      }
    }
  ]
}
```

#### 4. Create an Auto Scaling Group, EC2 Fleet, or Spot Fleet

https://docs.aws.amazon.com/cli/latest/reference/autoscaling/create-auto-scaling-group.html
Here is a [getting started tutorial for ASG](https://docs.aws.amazon.com/autoscaling/ec2/userguide/GettingStartedTutorial.html).

Make sure that you:
- Specify an SSH key that will be used later by Jenkins.
- Follow [Spot best practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html), if using Spot.

> [!WARNING]  
> AWS strongly discourages using the SpotFleets because it is now categorized as legacy API with no planned investment.
Use Auto Scaling Groups instead.
> [Spot Fleet documentation](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_RequestSpotFleet.html)

#### 5. Configure Jenkins

Once your ASG, EC2 Fleet, or Spot Fleet is ready, you can use it by adding a new **EC2 Fleet** cloud in the Jenkins configuration.

1. Goto ```Manage Jenkins > Plugin Manager```
1. Install ```EC2 Fleet Jenkins Plugin```
1. Goto ```Manage Jenkins > Configure Clouds```
1. Click ```Add a new cloud``` and select ```Amazon EC2 Fleet```
1. Configure AWS credentials, or leave empty to use the EC2 instance role
1. Specify Auto Scaling Group, EC2 Fleet, or Spot Fleet to use

More information on the configuration options can be found [here](https://github.com/jenkinsci/ec2-fleet-plugin/blob/master/docs/CONFIGURATION-OPTIONS.md).

## Scaling
You can specify the scaling limits in your cloud settings. By default, Jenkins will try to scale the fleet up
if there are enough tasks waiting in the build queue and scale down idle nodes after a specified idleness period.

You can use the History tab in the AWS console to view the scaling history.

## Groovy

Below is a Groovy script to setup Spot Fleet Plugin for Jenkins and configure it. You can
run the script with [Jenkins Script Console](https://wiki.jenkins.io/display/JENKINS/Jenkins+Script+Console).

```groovy
import com.amazonaws.services.ec2.model.InstanceType
import com.cloudbees.jenkins.plugins.sshcredentials.impl.BasicSSHUserPrivateKey.DirectEntryPrivateKeySource
import com.cloudbees.jenkins.plugins.sshcredentials.impl.BasicSSHUserPrivateKey
import com.cloudbees.jenkins.plugins.awscredentials.AWSCredentialsImpl
import hudson.plugins.sshslaves.SSHConnector
import hudson.plugins.sshslaves.verifiers.NonVerifyingKeyVerificationStrategy
import com.cloudbees.plugins.credentials.*
import com.cloudbees.plugins.credentials.domains.Domain
import hudson.model.*
import com.amazon.jenkins.ec2fleet.EC2FleetCloud
import jenkins.model.Jenkins

// just modify this config other code just logic
config = [
    region: "us-east-1",
    // Spot Fleet ID, EC2 Fleet ID, or Auto Scaling Group Name
    fleetId: "...", 
    idleMinutes: 10,
    minSize: 0,
    maxSize: 10,
    numExecutors: 1,
    awsKeyId: "...",
    secretKey: "...",
    ec2PrivateKey: '''-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----'''
]

// https://github.com/jenkinsci/aws-credentials-plugin/blob/aws-credentials-1.23/src/main/java/com/cloudbees/jenkins/plugins/awscredentials/AWSCredentialsImpl.java
AWSCredentialsImpl awsCredentials = new AWSCredentialsImpl(
  CredentialsScope.GLOBAL,
  "aws-credentials",
  config.awsKeyId,
  config.secretKey,
  "my aws credentials"
)

BasicSSHUserPrivateKey instanceCredentials = new BasicSSHUserPrivateKey(
  CredentialsScope.GLOBAL,
  "instance-ssh-key",
  "ec2-user",
  new DirectEntryPrivateKeySource(config.ec2PrivateKey),
  "",
  "my private key to ssh ec2 for jenkins"
)
// find detailed information about parameters on plugin config page or
// https://github.com/jenkinsci/ec2-fleet-plugin/blob/master/src/main/java/com/amazon/jenkins/ec2fleet/EC2FleetCloud.java
EC2FleetCloud ec2FleetCloud = new EC2FleetCloud(
  "", // fleetCloudName
  null,
  awsCredentials.id,
  config.region,
  "",
  config.fleetId,
  "ec2-fleet",  // labels
  "", // fs root
  new SSHConnector(22,
                   instanceCredentials.id, "", "", "", "", null, 0, 0,
                   // consult doc for line below, this one say no host verification, but you can use more strict mode
                   // https://github.com/jenkinsci/ssh-slaves-plugin/blob/master/src/main/java/hudson/plugins/sshslaves/verifiers/NonVerifyingKeyVerificationStrategy.java
                   new NonVerifyingKeyVerificationStrategy()),
  false, // if need to use privateIpUsed
  false, // if need alwaysReconnect
  config.idleMinutes, // if need to allow downscale set > 0 in min
  config.minSize, // minSize
  config.maxSize, // maxSize
  0,
  config.numExecutors, // numExecutors
  false, // addNodeOnlyIfRunning
  false, // restrictUsage allow execute only jobs with proper label
  "",
  false,
  180,
  null,
  30,
  true,
  new EC2FleetCloud.NoScaler()
)

// get Jenkins instance
Jenkins jenkins = Jenkins.get()
// get credentials domain
def domain = Domain.global()
// get credentials store
def store = jenkins.getExtensionList('com.cloudbees.plugins.credentials.SystemCredentialsProvider')[0].getStore()
// add credential to store
store.addCredentials(domain, awsCredentials)
store.addCredentials(domain, instanceCredentials)
// add cloud configuration to Jenkins
jenkins.clouds.add(ec2FleetCloud)
// save current Jenkins state to disk
jenkins.save()
```

## Preconfigure Agent

Sometimes you need to prepare an agent (an EC2 instance) before Jenkins can use it.
For example, you need to install some software which is required by your builds like Maven, etc.

For those cases you have a few options, described below:

### Amazon EC2 AMI

[AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html) allows you to 
create custom images for your EC2 instances. For example, you can create an image with
Linux plus Java, Maven etc. Then, when EC2 Fleet launches new EC2 instances with
this AMI they will automatically get all the required software. Nice =)

1. Create a custom AMI as described [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html#creating-an-ami)
2. Create EC2 Fleet or Spot Fleet with this AMI

### EC2 Instance User Data

EC2 instances allow you to specify a [User Data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
script that is executed when an instance first launches. This allows you to customize the setup for a particular instance.

#### SSH Prefix Verification

EC2 instances don't provide any information about the User Data script execution status,
so Jenkins could start a task on a new instance while the script is still in progress. Most of the time Jenkins will
repeatedly try to connect to the instance during this time and print out errors until the script completes and Jenkins
can connect.

To avoid those errors, you can use the Jenkins SSH Launcher ```Prefix Start Agent Command``` setting
to specify a command which should fail if User Data is not finished. In that way Jenkins will
not be able to connect to the instance until the User Data script is done. More information on configuring the SSH
launcher can be found [here](https://github.com/jenkinsci/ssh-slaves-plugin/blob/master/doc/CONFIGURE.md).

1. Open Jenkins
2. Go to ```Manage Jenkins > Configure System```
3. Find proper fleet configuration and click ```Advanced...``` for SSH Launcher
4. Add checking command into field ```Prefix Start Agent Command```
   - example ```java -version && ```
5. To apply for existing instances, restart Jenkins or Delete Nodes from Jenkins so they will be reconnected

# FAQ

Check out the FAQ & Gotchas page [here](https://github.com/jenkinsci/ec2-fleet-plugin/blob/master/docs/FAQ.md).

# Development

Plugin usage statistics per Jenkins version can be found [here](https://stats.jenkins.io/pluginversions/ec2-fleet.html)

## Releasing

https://jenkins.io/doc/developer/publishing/releasing/

```bash
mvn release:prepare release:perform
```

### Jenkins 2 can't connect by SSH 

https://issues.jenkins-ci.org/browse/JENKINS-53954

### Install Java 8 on EC2 instance 

Regular script:

```bash
sudo yum install java-1.8.0 -y
sudo yum remove java-1.7.0-openjdk -y
java -version 
```

User Data Script:

*Note* ```sudo``` is not required, ```-y``` suppresses confirmation.
Don't forget to encode with Base64

```bash
#!/bin/bash
yum install java-1.8.0 -y && yum remove java-1.7.0-openjdk -y && java -version
```

# Contributing
 
Contributions are welcome! Please read our [guidelines](https://github.com/jenkinsci/ec2-fleet-plugin/blob/master/CONTRIBUTING.md)
and our [Code of Conduct](https://github.com/jenkinsci/ec2-fleet-plugin/blob/master/CODE_OF_CONDUCT.md).
