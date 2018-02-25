# About

Some notes on Docker

## Clean up all dangling data

From: https://stackoverflow.com/a/32723127

```
docker system prune
```

This deletes all dangling data, so please be careful before using it.


## overlay2 taking up a lot of disk space

The directory `/var/lib/docker/overlay2` can take up a lot of disk space. There seems to be little way for Docker to prune it, so be prepared to invest in more disk space for it.

- https://forums.docker.com/t/some-way-to-clean-up-identify-contents-of-var-lib-docker-overlay/30604/10
- https://github.com/docker/distribution/blob/master/ROADMAP.md#deletes
