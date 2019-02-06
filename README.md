# Docker Stack Files

A central place for me to keep track of my stack files.

## Running on Swarm

### Login Securely

If any of your stack files reference private image registries.

```bash
cat ~/password.txt | docker login registry.gitlab.com -u <username> --password-stdin
```

### Deploying Stack

Order may matter if you have other stack files that reference others e.g. app-stack may have a external volume that is created within proxy-stack.

```bash
docker stack deploy --with-registry-auth -c traefik.yml proxy
```
