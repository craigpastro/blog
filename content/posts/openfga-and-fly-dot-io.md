---
title: "Deploy OpenFGA to Fly.io"
date: 2022-12-25T14:32:25-08:00
draft: true
---

## Prerequisites

You will need to install [flyctl](https://fly.io/docs/getting-started/installing-flyctl/) and have a [Fly.io](https://fly.io) account.

We will deploy OpenFGA from its Dockerfile. Clone the [OpenFGA repo](https://github.com/openfga/openfga) and add the last line below to its Dockerfile:

```Dockerfile
FROM golang:1.19-alpine AS builder

WORKDIR /app

COPY . .
RUN go build -o ./openfga ./cmd/openfga

FROM alpine as final
EXPOSE 8081
EXPOSE 8080
EXPOSE 3000
COPY --from=builder /app/openfga /app/openfga
COPY --from=builder /app/assets /app/assets
WORKDIR /app
ENTRYPOINT ["./openfga"]
CMD ["run"] # NEED TO ADD THIS
```

## Create a Fly Postgres Cluster

```
fly postgres create
```
and follow the prompts. Let's name the app `pg-openfga`.

After some time this command should output something that looks like:
```
Postgres cluster pg-openfga created
  Username:    postgres
  Password:    QkPQcru3eTuiPYz
  Hostname:    pg-openfga.internal
  Proxy port:  5432
  Postgres port:  5433
  Connection string: postgres://postgres:QkPQcru3eTuiPYz@pg-openfga.internal:5432
```

Set the connection string for the app using `fly secrets`:
```
flyctl secrets set OPENFGA_DATASTORE_URI=<connStr>
```

## Fly launch

Clone the OpenFGA repo, cd in, and run `fly launch`.

Set the app name to "openfga".

```toml
app = "openfga"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[env]
  OPENFGA_DATASTORE_ENGINE = "postgres"
  OPENFGA_PLAYGROUND_ENABLED = "false"

[experimental]
  allowed_public_ports = []
  auto_rollback = true

[deploy]
  release_command = "migrate"

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

## View the status

```
$ fly status
App
  Name     = openfga          
  Owner    = personal         
  Version  = 2                
  Status   = running          
  Hostname = openfga.fly.dev  
  Platform = nomad            

Deployment Status
  ID          = e4c952a0-21f2-b3d9-c090-172e7997da0e         
  Version     = v2                                           
  Status      = successful                                   
  Description = Deployment completed successfully            
  Instances   = 1 desired, 1 placed, 1 healthy, 0 unhealthy  

Instances
ID              PROCESS VERSION REGION  DESIRED STATUS  HEALTH CHECKS           RESTARTS        CREATED   
1c73bb82        app     2       sea     run     running 1 total, 1 passing      0               2m49s ago
```

## Destroy the app

```
fly destroy openfga
```

Check the the app is destroyed by checking that the app is no longer listed in
```
fly list apps
```
