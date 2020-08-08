---
title: "Automating AMI Creation"
toc: false
tags:
  - AWS, EC2, HashiCorp, Packer
---

This article briefly delves into Hashicorp Packer and what it can do for your AWS AMI creation workflow.

Setting up an AMI in the usual way is a well-defined sequence of steps:

1. Create an instance from an existing AMI, private or public.

2. SSH into the instance.

3. Execute a series of commands to configure it.

4. Stop the instance.

5. Create an AMI from the instance.

6. Terminate the instance.

7. If you need the AMI in a different region or account, do a manual copy operation or explicitly grant rights to a different account.

8. Repeat steps 1-7 whenever you need a change in the AMI.

Since AMI creation is such an involved and lengthy process, we might sometimes prefer to have an installation script that runs every time an instance is launched with the AMI and have the script configure the instance with the latest updates to suit our needs. This renders the launched instance unusable for a period of time until the configuration process is complete.

Justifiably, this cost might be acceptable considering the alternative of following the lengthy workflow laid out above every time you need your AMI updated with a new package.

But what if we could automate this process and treat AMI creation like a git push or a simple configuration change?

**Enter Packer.**

Packer automates every step in the AMI creation workflow using a JSON template that you define, which means that your workflow for setting up an AMI will now look like this:

```
1. Create a JSON template to configure Packer to create your AMI.

2. packer validate template.json

3. packer build template.json
```

When you need to update the AMI, modify your template to make the changes you need, and run steps 2 and 3.

The Packer documentation is comprehensive and can be found [here](https://www.packer.io/docs). But in order to get a quick intuition of what Packer can do, let's explore a simple example.


## Example Use Case

You need to build an AMI with the following properties:

* You have a series of existing AMIs that all start with the prefix `sample_ami_v_`. You will also be following this naming guideline for your new AMI.

  > Versioning the AMI names with predictable prefixes allows you to filter them programmatically.

* The `redis-server` apt package needs to be added to the AMI.

* The AMI needs to be primarily available in the `us-east-1` region, and additionally in the `ap-southeast-1` and `ap-south-1` regions.

* The AMIs should be accessible from a different AWS account with the account ID: `123456789123`.

### JSON Template

```json
{
  "variables": {
    "aws_access_key": "{{env `AWS_ACCESS_KEY_ID`}}",
    "aws_secret_key": "{{env `AWS_SECRET_ACCESS_KEY`}}"
  },
  "builders": [
    {
      "type": "amazon-ebs",
      "access_key": "{{user `aws_access_key`}}",
      "secret_key": "{{user `aws_secret_key`}}",
      "region": "us-east-1",
      "source_ami_filter": {
        "filters": {
          "name": "sample_ami_v_*"
        },
        "owners": ["987654321987"],
        "most_recent": true
      },
      "instance_type": "t2.medium",
      "ssh_username": "ubuntu",
      "ami_name": "sample_ami_v_{{timestamp}}",
      "run_tags": {
        "Name": "ami_creator"
      },
      "ami_regions": ["ap-southeast-1", "ap-south-1"],
      "ami_users": ["123456789123"]
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline_shebang": "/bin/bash -e",
      "inline": ["sudo apt install redis-server"]
    }
  ]
}
```
