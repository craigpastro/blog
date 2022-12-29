---
title: "Deploy OpenFGA to Fly.io"
date: 2022-12-28T14:32:25-08:00
draft: true
---

## Prerequisites

- You will need to install [flyctl](https://fly.io/docs/getting-started/installing-flyctl/) and have a [Fly.io](https://fly.io) account.
- Clone the [OpenFGA repo](https://github.com/openfga/openfga) and cd into it. We will work in there.

## fly launch

To create the initial config run
```console
fly launch
```
There will be a number of prompts:
1. For the app name choose: openfga.
2. Choose any region.
3. Add a Postgres database using the Development configuration. After a minute or so, some output will be produced, the most important of which is the connection string. Make a note of it.
4. Do not set up Redis.
5. Do not deploy now.

`fly launch` should have generated `fly.toml`. We will need to edit it in the next section.

## Some configuration

We need to do a bit of extra configuration. Let's first set the connection string for Postgres:
```console
fly secrets set OPENFGA_DATASTORE_URI=<connStr>
```
While we are at it, let's set an API key as well. Generate an API key and run:
```console
fly secrets set OPENFGA_AUTHN_PRESHARED_KEYS=<apiKey>
```

Now we need to edit the generated `fly.toml` file. We will need to add some fields to the `env`, `deploy` and `experimental` sections.

TODO: add some explaination to these config options.

The `env` section should look like:
```toml
[env]
OPENFGA_PLAYGROUND_ENABLED = "false"
OPENFGA_DATASTORE_ENGINE = "postgres"
OPENFGA_DATASTORE_MAX_IDLE_CONNS = "20"
OPENFGA_AUTHN_METHOD = "preshared"
```

The `deploy` section should look like:
```toml
[deploy]
release_command = "migrate"
```

And, finally, the `experimental` should look like:
```toml
[experimental]
cmd = ["run"]
allowed_public_ports = []
auto_rollback = true
```


In the end your `fly.toml` should look like this:

```toml
app = "openfga"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[env]
OPENFGA_PLAYGROUND_ENABLED = "false"
OPENFGA_DATASTORE_ENGINE = "postgres"
OPENFGA_DATASTORE_MAX_IDLE_CONNS = "20"
OPENFGA_AUTHN_METHOD = "preshared"

[deploy]
release_command = "migrate"

[experimental]
cmd = ["run"]
allowed_public_ports = []
auto_rollback = true

[[services]]
http_checks = []
internal_port = 8080
processes = ["app"]
protocol = "tcp"
script_checks = []
[services.concurrency]
hard_limit = 25
soft_limit = 20
type = "connections"

[[services.ports]]
force_https = true
handlers = ["http"]
port = 80

[[services.ports]]
handlers = ["tls", "http"]
port = 443

[[services.tcp_checks]]
grace_period = "1s"
interval = "15s"
restart_limit = 0
timeout = "2s"
```

## Deploy

Deploy OpenFGA to Fly.io:
```console
fly deploy
```
Once deployment is finished, OpenFGA should be running at `https://openfga.fly.dev`. Try to create a store
```
curl -XPOST 'https://openfga.fly.dev/stores' \
-H 'Authorization: Bearer <apiKey>'
-H 'Content-Type: application/json' \
-d '{
    "name": "openfga-on-fly-dot-io"
}'
```
Then write a model and tuples, authorize some users.

## View the status or logs

You can view the status of your deployment with `fly status` and logs with `fly logs`.

## Destroy the app

You can destroy the app and the database with the following commands:
```console
fly destroy openfga
fly destroy openfga-db
```

Check the the app is destroyed by checking that the app is no longer listed in
```console
fly list apps
```

## Deploy from the Docker image

TODO:

```toml
[build]
  image = "openfga/openfga:latest"
```
