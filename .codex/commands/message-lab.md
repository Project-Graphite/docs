# Message Lab

Validate the production Compose definition without exposing secret values:

```sh
SECRET_KEY=validation-only POSTGRES_PASSWORD=validation-only docker compose -f compose.production.yaml config --quiet
```

Run the complete repository CI path through its toolchain:

```sh
./run ci:test
```

Verify the public site and privacy-filtered API:

```sh
curl -I https://message-lab.project-graphite.com
curl https://message-lab.project-graphite.com/api/messages/
```

Inspect a deployed Coolify host without changing it:

```sh
free -h
df -h /
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
docker stats --no-stream
```
