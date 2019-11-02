## IAM roles for service accounts

For some strange reason, this part is not documented in AWS. The good thing is, we were able to find a blog post that covers it: https://medium.com/@marcincuber/amazon-eks-with-oidc-provider-iam-roles-for-kubernetes-services-accounts-59015d15cb0c

After EKS control plane is created, need to go to IAM -> Identity providers and click on `Create Provider`, entering the following details:

Provider type: OpenID Connect
Provider URL: OpenID Connect provider URL on the EKS cluster screen
Audience: sts.amazonaws.com
