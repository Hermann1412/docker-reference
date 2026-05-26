# L09-04

## Build the app

    docker compose build

This builds the `web-fe` image from the local Dockerfile.

## Run the app

    docker compose up -d

When the app is running, open it in your browser: http://localhost:5000

Both containers start — `web-fe` on port `5000` and `redis` on a mapped port.

## List the containers

    docker compose ps

Output shows both running services:

    NAME                      IMAGE                   PORTS
    docker-compose-redis-1    redis:alpine            0.0.0.0:52573->6379/tcp
    docker-compose-web-fe-1   docker-compose-web-fe   0.0.0.0:5000->5000/tcp

You can also use the standard Docker command:

    docker ps

## Look at the web-fe container logs

    docker compose logs -f web-fe

The Flask app starts in debug mode and logs incoming HTTP requests. Press `Ctrl+C` to stop following logs.

## Compose V2 commands

`ls` lists all currently running Compose projects:

    docker compose ls

## Deploy a second version using a different project name

Use the `-p` flag to assign a different project name, which namespaces all resources (containers, networks, images):

    docker compose -p test up -d

This succeeds and runs a second isolated instance alongside the first. Verify both are running:

    docker compose ls

Output:

    NAME             STATUS        CONFIG FILES
    docker-compose   running(2)    ...\docker-compose.yaml
    test             running(2)    ...\docker-compose.yaml

## Cleanup

Bring down the default project, then the named one:

    docker compose down
    docker compose ls
    docker compose -p test down
    docker compose ls