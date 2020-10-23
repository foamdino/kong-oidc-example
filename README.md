## Pre-reqs

* docker
* docker-compose
* jq (optional)

## Purpose

This repo contains the files needed to demonstrate how to protect an api using Kong community edition and an opensource OIDC plugin.

_It is not production ready and should only be used for demonstrations._

# Kong
We'll  start with getting Kong running first, then add protection via keyclock.

## Create new docker image

To use add the [OIDC plugin](https://github.com/nokia/kong-oidc) we need to build a new docker image.

In `docker/kong` the included `Dockerfile` contains the needed instructions to add the OIDC plugin correctly.

`docker build -t kong:2.1-centos-oidc docker/kong/`

## Spin-up kong-db

Kong uses Postgres to store configuration information and settings.  We use `docker-compose` to handle the communications between `kong` and `postgres`.

`docker-compose up -d kong-db`

## Run db migrations

`docker-compose up kong-migrations`

## Spin-up Kong

`docker-compose up -d kong`

After starting kong check that it running:
`docker-compose ps`

Check that the OIDC plugin is installed correctly:
`curl -s http://localhost:8001 | jq .plugins.available_on_server.oidc`

## Create services and Routes in Kong via the Admin API

First create a 'mock-service' using [mockbin](mockbin.org)

`curl -s -X POST http://localhost:8001/services -d name=mock-service -d url=http://mockbin.org/request | python -mjson.tool`

In the returned output there will be a json object with an id - save this value as we need it for this command:

`curl -s -X POST http://localhost:8001/routes -d service.id=<saved id value> -d 'paths[]=/mock' | python -mjson.tool`

## Spin-up keycloak-db

`docker-compose up -d keycloak-db`

Check that it's running:

`docker-compose ps`

## Spin-up keycloak

`docker-compose up -d keycloak`

## Configure keycloak

* In your browser go to `http://localhost:8180`
* Select the "Administrative Console" and  click "Clients" on the left-hand navigation.
* Click "Create" and on the "Add Client" page use 'kong' as the Client ID, then click "Save"
* Click on the Details for the kong client and enter 'confidential' for Access Type, http://localhost:8000 for Root URL and /mock/* for Valid Redirect URIs, then click "Save"
* From the new Credentials tab, select the secret value and record this somewhere - you will need to provide this later
* Click "Users" from the left-hand navigation and click "Add user" button
* Enter user for username, switch on Email Verified and click "Save" button
* On the credentials tab enter a password along with confirmation and set Temporary to OFF

## Configure Kong

Find your ip address:
`HOST_IP=$(ipconfig getifaddr en0)`

Set the client secret from Keycloak credentials earlier:
`CLIENT_SECRET=<client_secret_from_keycloak>`

Configure the OIDC plugin:

`curl -s -X POST http://localhost:8001/plugins -d name=oidc -d config.client_id=kong -d config.client_secret=${CLIENT_SECRET} -d config.discovery=http://${HOST_IP}:8180/auth/realms/master/.well-known/openid-configuration | python -mjson.tool`

## Testing

Open `http://localhost:8000/mock` in your browser and kong should redirect you to keycloak, enter the user/password you configured and you should then be authenticated and redirected back to mockbin.org