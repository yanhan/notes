# About

How to easily expose a service running on localhost to the internet for testing purposes, when you don't have a domain name.


## ngrok

Register for an account on https://ngrok.com . Download the binary for your OS, then run:
```
ngrok authtoken TOKEN_ON_DASHBOARD
```

ngrok supports exposing via HTTPS or TCP. To expose a service running on say port 4032 to the internet and get a HTTPS URL for it, run:
```
ngrok http 4032
```

You should see a the assigned URL


## localhost.run

Instructions at https://localhost.run/

For basic usage, suppose you have a HTTP service running on port 4032. To expose it, run:
```
ssh -R 80:127.0.0.1:4032 nokey@localhost.run
```


## localtunnel

Instructions at https://localtunnel.github.io/www/

To install:
```
npm install -g localtunnel
```

Suppose you have a HTTP server listening on port 4032 and wish to expose it. Run:
```
lt --port 4032
```


## References

- https://dev.to/levivm/exposing-localhost-server-to-the-internet-in-one-minute-2713
- https://localhost.run/
- https://localtunnel.github.io/www/
