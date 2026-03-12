# Nextcloud

## Useful `occ` commands

`occ` commands can be executed through the AIO docker container using `docker exec`. E.g. for filescan:

```bash
docker exec -u www-data nextcloud-aio-nextcloud php occ files:scan --all
```

Where `www-data` is the user and `nextcloud-aio-nextcloud` is the container name. You can find other occ commands in the official nextcloud documentation.
