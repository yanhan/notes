# About

Notes on Terraform.


## Renaming module without destroying resources

If you rename a module or refactor a resource which is not in a module into a module, Terraform will destroy and create the resource again, even if everything about the resource remains the same. This is true as of Terraform 0.12.5.

To avoid this needless destruction and creation, use the `terraform state mv` command. For instance, to rename a module from say `abc` to `def` while all details about the resources remain the same, modify your code, then run:

```
terraform init    # required because of the module rename
terraform state mv module.abc module.def
```

This can also be used to shift individual resources from anywhere to a module.

Source: [https://stackoverflow.com/a/49112331](https://stackoverflow.com/a/49112331)


## Managing secrets

Use:

- [aws-vault](https://github.com/99designs/aws-vault) to manage the AWS access key and secret key
- [chamber](https://github.com/segmentio/chamber) to manage Terraform secrets

We will not go through how to install aws-vault and chamber. Just read their documentation on GitHub.

On AWS, create an IAM user that has programmatic access but no direct permissions. We will be giving this IAM user the power to assume another role that has the permissions. (This is to handle the quirks of how aws-vault works).

Still on AWS, create an IAM role whose trusted entity is `Another AWS account`. Enter your own AWS account id. There is no need for External ID or MFA. This IAM role should have all the necessary privileges for Terraform.

Get the ARN of the newly created IAM role. Attach the following IAM policy to the IAM user we created earlier:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/YOUR_AWS_IAM_ROLE"
    }
  ]
}
```

Now, setup aws-vault using (replace `PROJECT_NAME` accordingly, say eg. `prd`):
```
aws-vault add PROJECT_NAME
```

On Linux, it will prompt you to enter the AWS access key and secret key. Thereafter, it will create a keyring named `awsvault`, so be prepared with a password for it.

Modify `~/.aws/config` such that it looks similar to:
```
[profile PROJECT_NAME]
region = YOUR_AWS_REGION

[profile YOUR_IAM_USER_NAME]
source_profile = PROJECT_NAME
role_arn = arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/YOUR_AWS_IAM_ROLE
region = YOUR_AWS_REGION
```

Replacing all the values in capital letters accordingly. Of course, instead of using the IAM user for the second profile, you could use something else.

The reason for doing all that is to align with aws-vault's behavior of using IAM roles to get temporary credentials. By doing so, when running aws-vault using the `YOUR_IAM_USER_NAME` profile, you will be using the `PROJECT_NAME` profile's AWS credentials to assume the role specified in the `role_arn` of the `YOUR_IAM_USER_NAME` profile. See [https://github.com/99designs/aws-vault#assuming-roles](https://github.com/99designs/aws-vault#assuming-roles) for more information.

On AWS, create a KMS key whose alias is `parameter_store_key`. This is for chamber. See [https://github.com/segmentio/chamber#setting-up-kms](https://github.com/segmentio/chamber#setting-up-kms) for more details.

Write all the secrets you need to using chamber (replacing `YOUR_IAM_USER_NAME` and `CHAMBER_PROJECT_NAME` accordingly):

```
aws-vault exec YOUR_IAM_USER_NAME -- chamber write CHAMBER_PROJECT_NAME keyOne valueOne
```

Then when running Terraform plan and apply, add the following prefix:
```
aws-vault exec YOUR_IAM_USER_NAME -- chamber exec CHAMBER_PROJECT_NAME --
```

For instance:
```
aws-vault exec YOUR_IAM_USER_NAME -- chamber exec CHAMBER_PROJECT_NAME -- terraform plan -out=plan-file
```

### Use IAM user with MFA

In `~/.aws/config`, add the `mfa_serial` field for the `YOUR_IAM_USER_NAME` profile.

Modify all calls to `aws-vault exec` to `aws-vault exec -n` so that you get prompted to enter the MFA token. Note that each MFA token can only be used once, so you will need to use a fresh token for each call to `aws-vault exec -n`.

## References

- [Rename Terraform resource without deleting](https://stackoverflow.com/a/49112331)
- [Support renaming / moving resources within the state](https://github.com/hashicorp/terraform/issues/591#issuecomment-235454327)
- [aws-vault: assuming roles](https://github.com/99designs/aws-vault#assuming-roles)
- [Create IAM role to delegate permissions to another IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html)
- [Using IAM role in AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html)
- [chamber - a CLI for managing secrets](https://github.com/segmentio/chamber)
- [chamber example: DevOps Stack Exchange](https://devops.stackexchange.com/a/4628)
