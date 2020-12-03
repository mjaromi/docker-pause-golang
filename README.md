# docker-pause-golang

## intro

There are some cases that you have to start a docker container, execute a single or multiple commands and stop a docker container. There are several ways in which you can achieve that.

* you can build separate image just to execute specific command,
* you can use the same image and just simple override `entrypoint` and `cmd`,
* you can run your docker image with entrypoint set to `sleep` for example: `sleep 99999` or `sleep infinity`,
* you can run your docker image with entrypoint set to `pause` like in following case
* in AWS ECS you can run scheduled tasks (cron) [2]
* in AWS ECS you can define separate task definition and run task on demand, which will run provided command, for example:
```shell
aws ecs run-task --cluster $AWS_ECS_CLUSTER_NAME --task-definition $AWS_ECS_TASK_DEFINITION
```

## example based on oryd/hydra

Since ORY Hydra 0.8.0, migrations are no longer run automatically on boot and sometimes you need to initialize the database schemas and this required to run `hydra migrate sql ${PSQL_URL}` command.

oryd/hydra based on the scratch docker image so there are two ways to run migrate command. One option is [1], the other has been described below:

```Dockerfile
FROM oryd/hydra:v1.9.0

FROM golang:1.15-alpine3.12
LABEL maintainer='mateusz.jaromi@gmail.com'
COPY pause.go .
RUN go build pause.go

FROM alpine:3.12
COPY --from=0 /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=0 /usr/bin/hydra /usr/bin/
COPY --from=1 /go/pause /usr/bin/pause
ENTRYPOINT [ "pause" ]
```

```shell
docker build -t hydra-pause:latest .
docker run -dit hydra-pause:latest

on windows:
docker exec -it $(docker ps -aq | select -f 1) sh

on linux:
docker exec -it $(docker ps -aq | head -1) sh

/ # hydra migrate sql ${PSQL_URL}
```

## docs
[1] https://github.com/ory/hydra/blob/ddd781bd5e9c545423b64601b5d8252c4faca18e/docs/faq/migrations.md

[2] https://docs.aws.amazon.com/AmazonECS/latest/developerguide/scheduled_tasks.html
