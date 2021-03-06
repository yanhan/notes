# About

AWS Systems Manager.

Purpose: Enables automated configuration and ongoing management of systems at scale.

## Preliminaries

- Agent based.
- Comes pre-installed with Amazon Linux and some Ubuntu Server AMIs. Just do a `ps aux` and grep for `amazon-ssm-agent`.
  - Also supports other Linux and Windows.
  - Works for on-prem too. But the installation will be slightly different.
  - For instances that don't have the agent, we will need to install it.
    - GitHub: https://github.com/aws/amazon-ssm-agent
    - Docs: https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-manual-agent-install.html

### IAM wise

- For IAM user to use SSM, attach the `AmazonSSMFullAccess` AWS managed IAM policy.
- For EC2 to be managed by SSM, attach the `AmazonEC2RoleforSSM` AWS managed IAM policy to its IAM role.

### Check for SSM managed EC2 instances

To see if an EC2 is managed by SSM, go to either of the following:

- EC2 -> Systems Manager Shared Resources -> Managed Instances
- AWS Systems Manager -> Shared Resources -> Managed Instances

If you don't see anything:

- Make sure you have attached the IAM policy to the IAM role.
- Ensure that the ssm agent is installed on the instance and is running.
- Check the logs of the ssm agent, by going to `/var/log/amazon/ssm`. Also run `systemctl status amazon-ssm-agent`. Restart the agent if necessary.
- If above is done but you still don't see anything, stop the EC2 instance and start it again.


## Run Command

Running commands on managed instances, without having to open up ports like SSH.

No additional cost. All actions are logged and access can be controlled.

### Quick example

NOTE: Please ensure that there is at least 1 EC2 instance managed by SSM. Otherwise you will not be able to do anything here.

This is an example to show you how to run a Shell command using SSM.

Go to AWS Systems Manager -> Actions -> Run Command.

1. Select the `AWS-RunShellScript` Command document.
2. For Commands, type `ls -al`.
3. For Targets, choose one of the EC2 instances.
4. Output is truncated to the final 2500 characters. If you want to see the full output of the command, you will need to modify `Output options` to use an S3 bucket or CloudWatch logs.

There are other configuration options. Feel free to try them out.

Once you run the command and it is done, select the instance under `Targets and outputs`, then click the `View output` button.

### Concepts

1. Managed Instance. Either on AWS or on-premise. As long as it has the ssm agent installed and the requisite IAM role (for AWS hosted) or access key (for on-prem).
2. Document. A series of steps executed in sequence. Can pass arguments at runtime to it. Can be predefined or customized. Can be shared across accounts and versioned. IAM can be used to lock down documents and instances to perform them on.
3. Command. Action to perform on a set of instances. Consists of a document, a set of targets and any runtime arguments.
4. Command Invocation. Instantiation of a command on a particular instance. Can view status and output on a particular instance.

### Security Notes

Based on https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-restrict-root-level-commands.html

Any IAM user is effectively running the commands as root. To harden security slightly, you should:

- Restrict the documents the IAM user is allowed, by limiting the `Resource` to a select set of Documents, for the action `ssm:SendCommand`.
- Restrict the instances the IAM user is allowed to send commands to, again for the action `ssm:SendCommand` but the `Resource` this time is EC2 instances. Perhaps those with a certain tag. See https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-rc-setting-up-cmdsec.html for more information.

### Running commands at "scale"

NOTE: Please try this to find out.

Scale is very subjective. Hence the quotes.

On awscli, there are the `--max-concurrency` and `--max-errors` flags. The first can be used to control the number of instances to run the command concurrently, while the second will stop further command invocations if the specified number of instances have errored out.

It is possible to specify these in percentages.


## Services in SSM

- Run command. Literally run commands on the hosts, without having to SSH into them.
- State manager. Define and maintain desired state of system. Mainly configuration related.
- Inventory. Collect and query software inventory.
- Patch Manager. Select and deploy patches across large groups of instances.
- Automation. Simplifies creating and updating AMIs.
- Parameter store. Securely store configuration data, eg. passwords. Can be used by other services such as Lambda.
- Maintenance windows. Built-in integration with Run command and Patch Manager.


## Building Blocks

- Document. A series of steps to be executed in sequence. Run command, State manager, Automation.


## Other experimentation

NOTE: Some of the information here may not be accurate.

- Need to create resource groups to do anything meaningful.
  - It is a must to use tags to find the required AWS resources (such as EC2 instances).
  - One can also choose the type of resources to include, such as EC2 instance only, EC2 instance + Internet Gateway, or other combinations.

## References

- [AWS videos on SSM](https://www.youtube.com/watch?v=zwS8lssaY_k&list=PLhr1KZpdzukeH5jKyYi55ef9tEWAllypB&index=1)
- [Restrict access to Documents and Instances](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-restrict-root-level-commands.html)
- [Restricting Run Command accessed based on Instance Tags](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-rc-setting-up-cmdsec.html)
