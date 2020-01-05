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


## Enable secrets engine

```
vault secrets enable -path=REPLACE_WITH_PATH  REPLACE_WITH_SECRETS_ENGINE
```

Eg.
```
vault secrets enable -path=secret kv-v2
```
