# Message Lab

Validate the production Compose definition without exposing secret values:

```sh
SECRET_KEY=validation-only POSTGRES_PASSWORD=validation-only docker compose -f compose.production.yaml config --quiet
```

Run the repository's application tests through its toolchain:

```sh
./run test
```

Inspect a deployed Coolify host without changing it:

```sh
free -h
df -h /
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
docker stats --no-stream
```
