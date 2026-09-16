# Message Lab

Run the constrained deployment stack on the disposable host:

```sh
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml pull
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml --profile tools run --rm migrate
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml up -d web worker postgres redis
```

Verify the services, application dependencies and current resource use:

```sh
curl --fail http://127.0.0.1:8000/up
curl --fail http://127.0.0.1:8000/up/databases
docker compose --env-file deploy.env -f compose.deploy.yaml -f compose.micro.yaml ps
docker stats --no-stream
```
