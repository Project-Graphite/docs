# Retiring a project

Retirement has a reversible registry phase and a destructive runtime phase. Do not combine them.

## Hide and stop future deployment records

Change the platform entry to:

```yaml
status: archived
deploy:
  kind: none
```

Keep the repository, authors, showcase, and tag history. Validate and merge this change first.

## Preserve state

Before any cluster deletion, identify persistent volume claims, create and verify a database export,
record the deployed image tags, and decide the retention period. A backup that has not been restored
in a test is not verified.

## Remove runtime resources

Namespace deletion removes workloads, Secrets, and persistent volume claims and may delete the
underlying disks. Cloudflare hostname and tunnel changes are separate external mutations. Prepare
the exact commands, inspect the current Kubernetes context and resources, and obtain specific
approval for each destructive step.

Do not reuse a retired hostname immediately. Keeping the archived registry entry prevents accidental
reassignment and preserves the Hub history.
