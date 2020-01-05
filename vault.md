# About

Commonly used vault commands.


## Initialization

Starting server:
```
vault server -config=config.hcl
```

Initialization:
```
vault operator init
```

Unseal:
```
vault operator unseal
```


## Secrets engines

List enabled secrets engines:
```
vault secrets list
```

Enable secrets engine:
```
vault secrets enable -path=REPLACE_WITH_PATH  REPLACE_WITH_SECRETS_ENGINE
```

Eg.
```
vault secrets enable -path=secret kv-v2
```


## kv-v2 secrets engine

kv-v2 is allows versioned secrets.

Note that deleted versions of secrets can be undeleted. But the same cannot be said for destroyed secrets.

Suppose the secrets engine is at path `secret/`.

Create secret:
```
vault kv put secret/some/company name="A big company" email="big@bigco.com"
```

Update the secret:
```
vault kv put secret/some/company name="A big company" email="bigco@bigco.com"
```

Note that you have to supply all the fields you want to be in the versioned secret. Hence if just a single field needs to be updated, use patch:
```
vault kv patch secret/some/company revenue=9999999
```

Read the secret:
```
vault kv get secret/some/company
```

Read version 1 of the secret:
```
vault kv get -version=1 secret/some/company
```

Delete version 2 of the secret:
```
vault kv delete -versions=2 secret/some/company
```

Get the metadata of a secret:
```
vault kv delete -versions=2 secret/some/company
```

Undelete version 2 of the secret:
```
vault kv undelete -versions=2 secret/some/company
```

Restrict the number of versions of a secret to keep, globally on the path:
```
vault write secret/config max_versions=3
```

Permanently delete a version of a secret:
```
vault kv destroy -versions=1 secret/some/company
```

Permanently delete all versions of a secret:
```
vault kv delete metadata secret/some/company
```


## Tokens

List token accessors:
```
vault list auth/token/accessors
```

Revoke token by accessor
```
vault token revoke -accessor TOKEN_ACCESSOR
```


## Dynamic Database secrets

I believe this is good for humans to get short lived credentials to access DB.

Enable the database secrets engine:
```
vault secrets enable database
```

Configure the engine:
```
vault write database/config/postgresql plugin_name=postgresql-database-plugin allowed_roles=myrole connection_url="postgresql://root:REPLACE_WITH_ROOT_PASSWORD@127.0.0.1:5432/postgres?sslmode=disable"
```

Create the following file `myrole.sql`:
```
CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "{{name}}";
```

Then create the role:
```
vault write database/roles/myrole db_name=postgresql creation_statements=@myrole.sql default_ttl=1h max_ttl=24h
```

Get credentials:
```
vault read database/creds/myrole
```

Each time you run the `vault read database/creds/myrole` command, you will get new credentials.

## References

- https://learn.hashicorp.com/vault/developer/sm-versioned-kv
- https://learn.hashicorp.com/vault/developer/sm-dynamic-secrets
- https://www.idkrtm.com/hashicorp-vault-managing-tokens/
