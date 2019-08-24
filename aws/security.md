# About

On AWS security.

## The model

Access key + secret key + session token

## Lambda

- Environment variables can be encrypted using KMS. There's an interface on the AWS Web console to do it. Within the code, use KMS to decrypt the env vars.
