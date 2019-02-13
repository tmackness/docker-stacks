# Getting Started With Traefik

I am very new to DevOps including Swarm, so if you have any pointers please create a pull request or issue.

I hope this helps others in any way.

## Quirks of Traefik

There are a few things you need to watch out for or you will get 404 or the like errors.

### 1. Stack Network Referencing

When using networks defined in a stack file (not an externally defined network) e.g.

```yml
networks:
  proxy:
    driver: overlay
    name: proxy
    driver_opts:
      encrypted: "true"
```

In addition to you deploying a stack as below named `proxy`.

```bash
docker stack deploy -c stack-proxy.yml proxy
```

Then you can reference the network as `proxy`, however, without the `name: proxy` field then you would have to reference the network as `proxy_proxy` due to the stack prefixing itself.

### 2. Traefik File Backend

From my understanding (and lack of success) you are unable to add `file` configurations as command parameters e.g. the follow **doesn't work**:

```yml
command:
  - ...
  - --file
  - --file.backends.backend1.servers.server1.url=http://193.242.31.20:8500
  - --file.frontends.frontend1.routes.consul.rule=Host:consul.domain.com
  - --file.frontends.frontend1.backend=backend1
```

There are a few ways around this:

1. Manually add them into Consul via UI
2. Use Consul API or CLI

For me, the best solution I could see was integrating the CLI into my build process with Terraform. I created a null_resource that adds my required configuration on the first `terraform apply` only (e.g. is used for initial setup only).

```yml
/*
 # Add Consul KV settings
 - No triggers as only runs once on initial apply
*/

resource "null_resource" "consul_kv" {
  count = "${var.swarm_manager_count}"
  depends_on = ["null_resource.consul_settings_managers"]

  connection {
    host        = "${element(digitalocean_droplet.swarm_managers.*.ipv4_address, count.index)}"
    type        = "ssh"
    user        = "${var.node_user}"
    port        = 2222
    private_key = "${file(var.ssh_key)}"
    timeout     = "2m"
  }

  # Docs https://docs.traefik.io/user-guide/kv-config/
  provisioner "remote-exec" {
    inline = [
      "consul kv put traefik/backends/backend${count.index}/servers/server${count.index}/url http://${element(digitalocean_droplet.swarm_managers.*.ipv4_address_private, count.index)}:8500",
      "consul kv put traefik/frontends/frontend${count.index}/routes/consul/rule Host:consul.${var.domain_name}",
      "consul kv put traefik/frontends/frontend${count.index}/backend backend${count.index}",
    ]
  }
}
```

### 3. Traefik Placement

Because we are using Docker Socket Proxy to handle our socket connection more securely, Traefik is then able to run on workers as well. Conventionally, it was restricted to run on managers due to the volume `- /var/run/docker.sock:/var/run/docker.sock:ro`. If you have enough worker nodes then you may constrain traefik to them using replicas or global deployment.

### 4. Using Consul Backend with Traefik

I elected to use Consul on my Host rather then containers. As such, you are unable to connect using an overlay network. The best solution I could find is using the bridge network to communicate from a container to the host. As you can see in `traefik-consul-backend-docker-socket-proxy.yml` stack I have an ENV named **CONSUL_HOST** which should be set pointing to either public or private IP of a Consul Node or in my case use the bridge IP. You can see your bridge IP by the following:

```bash
ip route show | awk '/docker0/ {print $9}'
```

You should set this ENV prior to deploying your stack e.g.

```bash
export CONSUL_HOST=$(ip route show | awk '/docker0/ {print $9}')
# deploy
docker stack deploy -c stack-proxy.yml proxy
```

The good thing about using the bridge IP is that the container will always talk to the Consul Agent on the same Node. As opposed to using a Public/Private IP which all containers will use throughout the cluster.
