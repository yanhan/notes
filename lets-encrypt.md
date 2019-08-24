# About

Set up TLS certs using Lets Encrypt.


## AWS NLB

On NLB, you will first need to create a listener on port 80. This will be used by certbot to access the endpoint that verifies that you control the domain name for which you are trying to a TLS cert for.

On your own computer:
```
sudo apt-get install -y certbot
sudo certbot certonly --manual
```

You will need to enter the domain name for which you are requesting the TLS cert for. Suppose it is `d01.mydomain.io`.

Thereafter, certbot will ask you to create a path at that domain name that returns a fixed value in order to do verify that you control the domain. Once that is done, press Enter.

The relevant files will be saved in the `/etc/letsencrypt/live/d01.mydomain.io/` directory. These are the files you need to upload to ACM:

- cert.pem
- fullchain.pem
- privkey.pem


## References

- https://itnext.io/node-express-letsencrypt-generate-a-free-ssl-certificate-and-run-an-https-server-in-5-minutes-a730fbe528ca
