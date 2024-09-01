# Prometheus Stack

## Summary

This stack deploys Prometheus monitoring system consisting of:

- Prometheus
- Grafana
- Alertmanager
- Docker socket proxy
- Cadvisor
- Node-exporter

Alertmanager is configured to page into a telegram group.

The following systems are being polled for metrics apart from Prometheus itself:

- Authentik: `http://aegis.lan:9300/metrics`
- Vault: `https://aegis.lan:8200/v1/sys/metrics`
- Cadvisor: `dockerswarm tasks discovery`
- Node-exporter: `dockerswarm tasks discovery`
- Proxmox: `https://pve.lan:9221/pve`
- Swarm cluster nodes: `dockerswarm nodes discovery`

## Prerequisites

A few steps below are necessary for deployment to succeed.

### Authentik OIDC provider for grafana

Create an additional OIDC provider and application in Authentik:

Provider:

- Name: `grafana`
- Redirect URIs: `https://grafana.swarm.lan/`
- Scopes: `email, openid, profile`
- Subject mode: `based on user's email`

`client_id` and `secret_id` should be placed in Vault as described later.

Application:

- Name: `grafana`
- Slug: `grafana`
- Provider: `grafana`

### Authentik Proxy provider for Prometheus

Create an additional proxy provider and application in Authentik:

Provider:

- Name: `devops_access_prometheus`
- ForwardAuth (single application) / External Host: `https://prometheus.swarm.lan`
- Scopes: `email, openid, profile`
- Subject mode: `based on user's email`

Application:

- Name: `devops_access_prometheus`
- Slug: `devops_access_prometheus`
- Provider: `devops_access_prometheus`
- Launch URL: `https://prometheus.swarm.lan`

Outposts:

- Edit  `authentik Embedded Outpost`
- Add `devops_access_prometheus` in application sections

### Authentik Proxy provider for Alertmanager

Create an additional proxy provider and application in Authentik:

- Name: `devops_access_alertmanager`
- ForwardAuth (single application) / External Host: `https://alertmanager.swarm.lan`
- Scopes: `email, openid, profile`
- Subject mode: `based on user's email`

Application:

- Name: `devops_access_alertmanager`
- Slug: `devops_access_alertmanager`
- Provider: `devops_access_alertmanager`
- Launch URL: `https://alertmanager.swarm.lan`

Outposts:

- Edit  `authentik Embedded Outpost`
- Add `devops_access_alertmanager` in application sections

### Telegram bot

Here are the basic steps to register a Telegram bot. A more detailed version is available [here](https://core.telegram.org/bots/features#botfather).

Use this link to access Telegram's bot creation bot: [https://t.me/botfather](https://t.me/botfather)

WARNING: Double-check that there is a blue verified mark near its name, as there are many scam bots with similar names.

Use the `/newbot` command to create a new bot. `@BotFather` will ask you for a name, and username, and then it will generate a token for your bot.

To illustrate, let's create a bot with the name `xzjsnfsvvs`:

```text
> /newbot
Alright, a new bot. How are we going to call it? Please choose a name for your bot.

> xzjsnfsvvs
Good. Now let's choose a username for your bot. It must end in `bot`.
Like this, for example: TetrisBot or tetris_bot.

> xzjsnfsvvs_bot
Done! Congratulations on your new bot. You will find it at t.me/xzjsnfsvvs_bot.
You can now add a description, about section and profile picture for your bot, see /help for a list of commands.
By the way, when you've finished creating your cool bot, ping our Bot Support if you want a better username for it.
Just make sure the bot is fully operational before you do this.

Use this token to access the HTTP API:
7301465204:AAHPjwyWx6Y6Jd1Eu1vpdaOrsxbP9tNE8Ys
Keep your token secure and store it safely, it can be used by anyone to control your bot.

For a description of the Bot API, see this page: https://core.telegram.org/bots/api
```

Now create a Telegram group and add the bot there. Then go to `Manage group->` `Administrators->` `Add administrator->` `Add Administrator` and select `xzjsnfsvvs_bot` as admin.

Now post any message in the group. And then open
`https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates` in a browser, or use `curl`  to get chat's id:

```bash
$ curl -s https://api.telegram.org/bot7301465204:AAHPjwyWx6Y6Jd1Eu1vpdaOrsxbP9tNE8Ys/getUpdates | jq '.result[-1].message.chat.id'
-4479691870
```

NOTE: `id` may start with the minus `-` sign. No worries, it is a part of `id`.

Both `chat_id` and `bot_token` would be required to configure a pager in `./config_alertmanager/alertmanager.yaml`, though the token should be placed in Vault as described below.

### Vault / Swarm secrets

Stack expects a few docker swarm secrets to be present.

Secret should be [created in Vault via terraform](https://github.com/graysievert-lab/Homelab-030_Secrets_and_Auth/tree/master/stack-110-prometheus) and populated with values via UI or API.

The transition of secrets to Docker Swarm is scripted in the ansible play `120-Configuration_ansible/play-secrets-stack-110-prometheus.yaml`. Using scripts and steps described in [Homelab-040_Docker_Swarm](https://github.com/graysievert-lab/Homelab-040_Docker_Swarm?tab=readme-ov-file#initialize-environment) one needs to sign a SSH key to make it valid for connections to `swarm.lan` node and execute the play:

```bash
$ ANSIBLE_CONFIG=120-Configuration_ansible/ansible.cfg \
ansible-playbook -v 120-Configuration_ansible/play-secrets-stack-110-prometheus.yaml
```

### Proxmox metrics exporter service

To squeeze prometheus metrics out of Proxmox we will mostly follow this [guide](https://community.hetzner.com/tutorials/proxmox-prometheus-metrics)

Create Proxmox VE API User:

```bash
$ pveum user add pve-exporter@pve
```

Grant the user read-only `PVEAuditor` role:

```bash
$ pveum acl modify / -user pve-exporter@pve -role PVEAuditor
```

Create API token:

```bash
$ pveum user token add pve-exporter@pve pve-exporter --privsep 0 --output-format=yaml
```

Create a dedicated system user for running the exporter as a service:

```bash
$ useradd -s /bin/false pve-exporter
```

Create `python3` virtual environment:

```bash
$ apt update
$ apt install -y python3-venv
$ python3 -m venv /opt/prometheus-pve-exporter
```

Activate the virtual environment:

```bash
$ source /opt/prometheus-pve-exporter/bin/activate
```

Install `prometheus-pve-exporter`:

```bash
$ pip install prometheus-pve-exporter
```

Leave the virtual environment:

```bash
$ deactivate
```

Create the configuration file for exporter (change `<YOUR_TOKEN_VALUE>` for the one from the output of the `pveum user token add` command above):

```bash
$ mkdir -p /etc/prometheus-pve-exporter && \
cat <<EOF> /etc/prometheus-pve-exporter/pve.yml
default:
  user: pve-exporter@pve
  token_name: "pve-exporter"
  token_value: "<YOUR_TOKEN_VALUE>"
  verify_ssl: true
EOF
```

Also, set the file owner and the permissions:

```bash
$ chown root:pve-exporter /etc/prometheus-pve-exporter/pve.yml
$ chmod 640 /etc/prometheus-pve-exporter/pve.yml
```

Now let's create the service.

For the exporter to respond via HTTPS we would need to provide it with access to the server's TLS certificate and key:

```bash
$ usermod -aG www-data pve-exporter
```

Add the following content to the `/etc/systemd/system/prometheus-pve-exporter.service` file:

```bash
$ cat <<EOF>/etc/systemd/system/prometheus-pve-exporter.service
[Unit]
Description=Prometheus Proxmox VE Exporter
Documentation=https://github.com/prometheus-pve/prometheus-pve-exporter

[Service]
Environment="REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt"
Restart=always
RuntimeMaxSec=24h
User=pve-exporter


## Chose one to uncomment:
#
## This will respond on http.
# ExecStart=/opt/prometheus-pve-exporter/bin/pve_exporter \
# --config.file /etc/prometheus-pve-exporter/pve.yml
#
## This will respond on https.
ExecStart=/opt/prometheus-pve-exporter/bin/pve_exporter \
--config.file /etc/prometheus-pve-exporter/pve.yml \
--server.keyfile /etc/pve/local/pveproxy-ssl.key \
--server.certfile /etc/pve/local/pveproxy-ssl.pem

[Install]
WantedBy=multi-user.target
EOF
```

Reload systemd, enable and start the service.

```bash
$ systemctl daemon-reload
$ systemctl enable prometheus-pve-exporter.service
$ systemctl start prometheus-pve-exporter.service
```

Test:

```
$ curl https://pve.lan:9221/pve?target=pve.lan
```

One may find the corresponding prometheus `scrape_config` in `./config_prometheus/scrape_configs/job-proxmox.yaml`

## Stack deployment

To deploy this stack:

- Go to `Stacks` in Portainer dashboard and `+ Add Stack`
- Name: `110-prometheus`
- Build method: `Repository`
- Repository URL: `https://github.com/graysievert-lab/Homelab-041_Docker_Swarm_Portainer_controlled_stacks`
- Compose path: `/stack-110-prometheus/compose.yaml`
- GitOps updates: `ON`
- Mechanism: `Webhook`  (copy URL, we need it later)
- Force redeployment: `ON`
- Enable relative path volumes: `ON`
- Network filesystem path: `/mnt/stacks`

Now create the file `webhook` in the stack's folder and paste there the webhook URL. Commit changes.

Deploy the stack and visit:

- `https://prometheus.swarm.lan` - Prometheus.
- `https://alertmanager.swarm.lan` - Alertmanager.
- `https://grafana.swarm.lan` - Grafana. Use `Sign in with authentik` at the bottom of the login page.