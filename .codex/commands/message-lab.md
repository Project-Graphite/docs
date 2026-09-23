# Message Lab

Validate the production Compose definition without exposing secret values:

```sh
SECRET_KEY=validation-only POSTGRES_PASSWORD=validation-only docker compose -f compose.production.yaml config --quiet
```

Run the CI test pipeline. `./run ci:install-deps` installs `wait-until`, which `ci:test` needs:

```sh
./run ci:install-deps
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
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
sudo docker stats --no-stream
```
