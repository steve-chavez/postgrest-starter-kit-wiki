## What We're Building

As tradition goes we'll build and API for a To-do app but we'll go much further than the concept of tasks. We'll have users, clients, projects, tasks and (real-time) comments, all the core models to build a complete API for a project management tool. We'll also have a codename for this secret project, "Khumbu Icefall", you get extra points if you can figure out why I chose the name. 


Unlike most tutorials that just explain a few basic concept with a toy example, this one will take you through all the steps of building a real-world, complex API that is ready to go into production. You've probably heard (marketing) phrases like "build a production ready API in 5 minutes", tried it and found out it's ... marketing, a production ready toy, not a real system. This tutorial is not going to take 5 minutes but it's also not going to take days to complete (or even months, because usually, that is the timeframe you are looking at when building APIs). We are looking at 1-2 hours of your time. If you are in a hurry or maybe you'd like to see why you should spend two hours on this, skip to the end and try interacting with a live version of Khumbu Icefall (KI) or run it directly on your computer.

When presented with systems like PostgREST, in some cases the conversation goes like "Yes, I guess it's an excellent tool for prototyping but my API is not a 1:1 map of my data models and it has a lot of business logic so I can not use it in production". In this tutorial, we'll show how a PostgREST based API is not coupled to your tables and how you can implement your custom business logic and even integrate/interface with 3rd party systems (and not just by calling webhooks).

## Installation
Clone the project

```sh
git clone --single-branch https://github.com/subzerocloud/postgrest-starter-kit khumbuicefall
cd khumbuicefall
```
Familiarize yourself a bit with the [project structure](https://github.com/subzerocloud/postgrest-starter-kit/wiki/Architecture-and-Project-Structure)

Edit the file `.env` file and give the project a distinct name

```sh
COMPOSE_PROJECT_NAME=khumbuicefall
```

Bring up the system to check everything works

```sh
docker-compose up -d # wait for 5-10s before running the next command
curl http://localhost:8080/rest/todos?select=id
```

The result should look something like this
```json
[{"id":1},{"id":3},{"id":6}]
```
Install and bring up the [developer tools](https://github.com/subzerocloud/postgrest-starter-kit/wiki/Installation#developer-tools)

Now that we have the API and `sz` up and running, let's see what we can do.

Run this request again and see it reflected in the dev console

```sh
curl http://localhost:8080/rest/todos?select=id
```

Now let's make an authorized request

```sh
export JWT_TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJyb2xlIjoid2VidXNlciJ9.vAN3uJSleb2Yj8RVPRsb1UBkokqmKlfl6lJ2bg3JfFg
curl -H "Authorization: Bearer $JWT_TOKEN" http://localhost:8080/rest/todos?select=id,todo
```
```json
[{"id":1,"todo":"item_1"},{"id":3,"todo":"item_3"},{"id":6,"todo":"item_6"}]
```

Notice the first item in the list `{"id":1,"todo":"item_1"}`.
Open the file `db/src/sample_data/data.sql` and change `item_1` to `updated` and save the file.

Look at the console window again, you should see in the bottom window the text `Ready --------------`.

Run the last request again

```sh
curl -H "Authorization: Bearer $JWT_TOKEN" http://localhost:8080/rest/todos?select=id,todo
```

```json
[{"id":1,"todo":"updated"},{"id":3,"todo":"item_3"},{"id":6,"todo":"item_6"}]
```

This process works for any `*.sql` `*.lua` `*.conf` file, you just make the change, save the file and you can run your next request, assuming your changes were valid (when working with sql, look at the PostgreSQL window).
