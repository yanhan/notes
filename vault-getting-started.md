# About

Summarized version of https://www.vaultproject.io/intro/getting-started/index.html

## Install

Just download the zipped binary. Extracting it gives us the `vault` binary. Make sure to put in in a directory on the PATH.

## Autocomplete

```
vault -autocomplete-install
```

This will append 2 lines to the zshrc file.

Make sure to `exec ${SHELL}` or just create a new shell session.

## Start dev server

```
vault server -dev
```

There will be some output. Make sure to export the `VAULT_ADDR` and the root token. Example:
```
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_DEV_ROOT_TOKEN_ID='s.QrWs53fcO5ky3LSQAZQBFFit'
```

Also save the unseal key somewhere.

## Verify dev server is running properly

In the terminal session with the above 2 exported variables, run:
```
vault status
```

You should see some output and no error messages.

## First secrets

NOTE: The `secret` prefix is important.
```
vault kv put secret/hello foo=world
```

Version 2 of `secret/hello` with 2 key value pairs:
```
vault kv put secret/hello foo=world excited=yes
```

Retrieving the secret:
```
vault kv get secret/hello
```

Get one field:
```
vault kv get -field=excited secret/hello
```

Output in JSON format and extract using jq:
```
vault kv get -format=json secret/hello | jq -r .data.data.excited
```

Delete the secret (not really, this just deletes the most recent version of the secret, you can still retrieve the non deleted versions):
```
vault kv delete secret/hello
```

## Secrets engines

If you try to do the following:
```
vault write foo/bar a=b
vault kv put foo/bar a=b
```

They will fail because there are no secrets engine at the path `foo`. The calls earlier to write key-value pairs to `secret/hello` worked because there is a secrets engine at the path `secret` whose engine type is `kv`.

Enable a secrets engine at path `potato` with type `kv`:
```
vault secrets enable -path=potato kv
```

You can list all the secrets engines using:
```
vault secrets list
```

Time to write some secrets to our new `potato` secrets engine:
```
vault write potato/my-secret value="s3c(eT"
vault write potato/hello target=world
vault write potato/airplane type=boeing class=787
vault list potato
```

Disable the secrets engine:
```
vault secrets disable potato/
```

## Dynamic Secrets (using AWS)

NOTE: Please try this at home using our own account. Also verify if a new IAM user is created each time we request for the secret.

Essentially, dynamic secrets are generated on the fly when requested. Prior to that, they do not exist in Vault.

To enable the AWS secrets engine:
```
vault secrets enable -path=aws aws
```

Configure the engine with access keys for a sufficiently privileged account. While we do not know what are the exact IAM permissions required, please DO NOT use the AWS root account for this;
```
vault write aws/config/root \
  access_key=YOUR_AWS_ACCESS_KEY_ID \
  secret_key=YOUR_AWS_SECRET_ACCESS_KEY
```

Create a role, whose access key and secret key you will be requesting soon:
```
vault write aws/roles/my-role \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF
```

`aws/roles/:name` is a special path that tells vault to create an IAM role and attach the given IAM policy whenever a user asks for credentials for `my-role`.

**NOTE:** The AWS credentials given to the `vault write aws/config/root` account must have permissions to create an IAM user, create inline IAM policies and attach them.

Request for the secret (which causes it to be generated):
```
vault read aws/creds/my-role
```

This should show you something like:
```
Key                Value
---                -----
lease_id           aws/creds/my-role/757bf32d-6e3e-4882-9eeb-6515a5b5cfb0
lease_duration     768h
lease_renewable    true
access_key         AKIA345ZDFG91AS01F
secret_key         KSDF45m+asd/490618fJSYMz354
security_token     <nil>
```

You can then make use of the `access_key` and `secret_key` in your application.

Each time you run the `vault read ...` command, a new IAM user will be created, along with new access key pair.

Take note of the `lease_id`, which can be used for renewal, revocation and inspection.

To revoke this credential:
```
vault lease revoke aws/creds/my-role/757bf32d-6e3e-4882-9eeb-6515a5b5cfb0
```

This deletes the IAM user on AWS.

## Built-in Help

For instance, for AWS. Enable the AWS secrets engine first:
```
vault secrets enable -path=aws aws
```

Then browse:
```
vault path-help aws
```

This also works for other secrets engines (just supply their path as the final argument):
```
vault path-help secret
```

To browse the help for say an AWS credentials pair (you can just type the command below verbatim; it will work even if the role doesn't actually exist):
```
vault path-help aws/creds/non-existent-role
```

## Authentication

Authentication is the process by which user or machine supplied information is verified can converted into a Vault token with matching policies.

Vault has pluggable auth methods, so it's easy to authenticate with Vault using many different methods.

Token authentication is enabled by default and cannot be disabled. Running a Vault dev server will output your root token, which has root privileges.

You can create more tokens using:
```
vault token create
```

which will create a child token of your current token that inherits all the same policies. Tokens always have a parent and when the parent token is revoked, children can be revoked in a single operation. This makes it easy when removing access for a user and access for all sub-tokens the user created.

Revoke a token using:
```
vault token revoke <TOKEN>
```

To authenticate using a token:
```
vault login <TOKEN>
```

As a best practice, operators should not use `vault token create` to generate tokens. Instead, users and machines that require tokens should authenticate to Vault using configured auth methods such as GitHub, LDAP, etc.

For details on the various authentication methods, please refer to https://www.vaultproject.io/api/auth/index.html


## Policies

Used for authorization. The `root` and `default` policies cannot be removed.

Example policy (written in HCL):
```
# Normal servers have version 1 of KV mounted by default, so will need these
# paths:
path "secret/*" {
  capabilities = ["create"]
}

path "secret/foo" {
  capabilities = ["read"]
}

# Dev servers have version 2 of KV mounted by default, so will need these
# paths:
path "secret/data/*" {
  capabilities = ["create"]
}

path "secret/data/foo" {
  capabilities = ["read"]
}
```

A user with this policy will be able to write to the secret engine named `secret`, but not to `secret/foo`.

Suppose the policy file is named `my-policy.hcl`. To format the policy file and check for errors:
```
vault policy fmt my-policy.hcl
```

The policy format uses a prefix matching system on the API path to determine access control. The most specific defined policy is used, either an exact match or the longest-prefix glob match.

To upload the policy:
```
vault policy write my-policy my-policy.hcl
```

You should see `my-policy` listed when you do:
```
vault policy list
```

To view the contents of the policy that we uploaded:
```
vault policy read my-policy
```

To test out the policy, create a token with it and login using the token:
```
vault token create -policy=my-policy
vault login THE_TOKEN_CREATED_WITH_THE_COMMAND_BEFORE_THIS
```

Then try writing some tokens:
```
vault kv put secret/big fun=times
vault kv put secret/foo black=mirror
```

The first write should succeed while the second should fail.

To switch back to the original permissions we have, login using the root token:
```
vault login ${VAULT_DEV_ROOT_TOKEN_ID}
```

There is a way to map policies to auth method, but it needs some kind of auth method which we are not trying out here.

## Deploying Vault

NOTE: This is more realistic than using memory to store Vault secrets, but is still not a production grade deployment. For that to happen, we need a production grade deployment of Consul.

Download consul and put it somewhere on your PATH. Then run:
```
consul agent -dev
```

Create the following file named `config.hcl`:
```
storage "consul" {
  address = "127.0.0.1:8500"
  path = "vault/"
}

listener "tcp" {
  address = "127.0.0.1:8200"
  tls_disable = 1
}
```

To let Vault use `mlock` without running the process as root:
```
sudo setcap cap_ipc_lock=+ep $(readlink -f $(which vault))
```

Then start vault:
```
vault server -config=config.hcl
```

We need to initialize the vault so it generates encryption keys, unseal keys and sets up the initial root token:
```
export VAULT_ADDR='http://127.0.0.1:8200'
vault operator init
```

Save the unseal keys and initial root token somewhere.

In a real setup, the unseal keys would be encrypted by several GPG keys, each belonging to different people. See https://www.vaultproject.io/docs/concepts/pgp-gpg-keybase.html for using Keybase / GPG for that.

Vault starts out in a sealed state and is unable to decrypt any data. We need to unseal the Vault so it knows how to do so. The threshold number of unseal keys need to be supplied.

This has to happen each time vault is started.

To unseal the vault:
```
vault operator unseal
```

When the `Sealed = false`, the vault is unsealed.

Then login using the root token:
```
vault login ROOT_TOKEN
```

To seal the vault:
```
vault operator seal
```

## Example using the API

Create a file `disk_config.hcl`:
```
backend "file" {
  path = "vault"
}

listener "tcp" {
  tls_disable = 1
}
```

Then start vault using:
```
vault server -config=./disk_config.hcl
```

Initialize Vault:
```
curl -XPOST --data '{"secret_shares": 1, "secret_threshold": 1}' http://127.0.0.1:8200/v1/sys/init
```

Record down the base64 unseal key and root token.

Then run:
```
export VAULT_TOKEN=ROOT_TOKEN
curl -XPOST --data '{"key": "REPLACE WITH base64 unseal key"}' http://127.0.0.1:8200/v1/sys/unseal
```

To enable AppRole authentication:
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  -XPOST \
  --data '{"type": "approle"}' \
  http://127.0.0.1:8200/v1/sys/auth/approle
```

Using the `my-policy.hcl` file created above, pass it to the `base64` program. Then manually join all the lines output by the base64 program together:
```
base64 my-policy.hcl
```

Then create a file named `my-policy.json` with the following content:
```
{
  "policy": "the base64 output above, concatenated into 1 line"
}
```

Issue an API call to create the policy:
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  -XPUT \
  --data @my-policy.json \
  http://127.0.0.1:8200/v1/sys/policies/acl/my-policy
```

To read the policy:
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  http://127.0.0.1:8200/v1/sys/policies/acl/my-policy
```

Create an AppRole named `my-role` with the desired policies:
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  -XPOST \
  --data '{"policies": ["my-policy"]}' \
  http://127.0.0.1:8200/v1/auth/approle/role/my-role
```

The AppRole backend expects a hard to guess role ID and a secret ID. First we retrieve the role ID:
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  http://127.0.0.1:8200/v1/auth/approle/role/my-role/role-id | jq
```

There will be a `data` key with `role_id` key.

Then create a new secret ID under `my-role`:
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  -XPOST \
  http://127.0.0.1:8200/v1/auth/approle/role/my-role/secret-id | jq
```

There will be a `data` key with a `secret_id` key.

With the role ID and secret ID, we can get a new Vault token from the login endpoint:
```
curl \
  -XPOST \
  --data '{"role_id": "THE ROLE ID", "secret_id": "THE SECRET ID"}' \
  http://127.0.0.1:8200/v1/auth/approle/login | jq
```

The Vault token is in the `auth` -> `client_token`. This token will be authorized with all the policies used to create it.

To try writing a key to `secret/hello`:
```
export VAULT_TOKEN=REPLACE_WITH_new_vault_token
curl \
  -XPOST \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -d '{"apple": "pear"}' \
  http://127.0.0.1:8200/v1/secret/foo
```

Retrieve the secret using:
```
curl -H "X-Vault-Token: $VAULT_TOKEN" http://127.0.0.1:8200/v1/secret/foo
```

## Copyright

Copyright to HashiCorp, Inc.
