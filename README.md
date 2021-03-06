| Service  | Docker Image           | Build Status |
|:---------|:-----------------------|:-------------|
| API      | subfuzion/vote-api     | [![Docker Build Status](https://img.shields.io/docker/build/subfuzion/vote-api.svg)](subfuzion/vote-api)
| Worker   | subfuzion/vote-worker  | [![Docker Build Status](https://img.shields.io/docker/build/subfuzion/vote-worker.svg)](subfuzion/vote-worker)
| Auditor  | subfuzion/vote-auditor | [![Docker Build Status](https://img.shields.io/docker/build/subfuzion/vote-auditor.svg)](subfuzion/vote-auditor)

| Node.js Packages    | npm                    | Build Status |
|:--------------------|:-----------------------|:------------ |
| @subfuzion/database | [![npm (scoped)](https://img.shields.io/npm/v/@subfuzion/database.svg)](@subfuzion/database) | [![Travis](https://img.shields.io/travis/subfuzion/docker-voting-app-nodejs.svg)](subfuzion/docker-voting-app-nodejs)
| @subfuzion/queue    | [![npm (scoped)](https://img.shields.io/npm/v/@subfuzion/queue.svg)](@subfuzion/queue) | [![Travis](https://img.shields.io/travis/subfuzion/docker-voting-app-nodejs.svg)](subfuzion/docker-voting-app-nodejs)

# Docker Voting App (Node.js version)

This app is inspired by the existing [Example Voting App](https://github.com/dockersamples/example-voting-app)
by [ManoMarks](https://twitter.com/ManoMarks), head of developer relations at Docker.

The existing app is an excellent demonstration of how Docker can be used to containerize any of the
processes of a modern application regardless of the programming language used and runtime environment
needed for any specific one.

It is an effective example, particularly from a devops perspective. However, if one is interested in
studying the application source code itself, then the use of multiple languages, even for a simple example,
potentially raises the barrier to comprehension.

This version of the voting app has been developed to support an introductory course on Docker and is
meant to be easier to follow and comprehend due to symmetric use of a single programming language for
all of the application service and client code, especially for those programmers who have already had
some exposure to [JavaScript](https://www.javascript.com/) and [Node.js](https://nodejs.org/).

While JavaScript has its quirks, the code for the various packages in this example are written using
the latest EcmaScript support available in recent (`8.0+`) versions of Node. In particular, the use of
[async functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
should make control flow easier to follow for developers not as familiar with Node.js asynchronous callback
conventions.

As modest as the app is in terms of actual functionality (it only supports casting votes and querying
the vote tally), it has nevertheless been designed to reflect principles of a relatively
sophisticated architecture implemented as a modern [12-Factor App](https://12factor.net/),
well suited to showcasing the benefits of Docker.

## License

The Voting App is open source and free for any use in compliance with the terms of the
[MIT License](https://github.com/subfuzion/example-voting-app-nodejs/blob/master/LICENSE).

## Introduction to Docker

The app will be used for an introductory course called **Software Containerization with Docker for Developers**.
The course is offered through the
[UC Davis Extension](https://extension.ucdavis.edu/online-learning) online program,
available through [Coursera](https://www.coursera.org/) in early 2018.

See the [wiki](https://github.com/subfuzion/docker-ucdavis-coursera/wiki) for more detail about the course modules.

## Application Architecture

![Voting app architecture](https://raw.githubusercontent.com/subfuzion/docker-ucdavis-coursera/master/images/voting-app-architecture.png)

The application consists of the following:

 * **Voter** - this is the client application that users use to cast votes.
 * **Vote** - this is the service that provides a REST API for casting votes and
   retrieving vote tallies. The `voter` application is the client that uses
   this API. When a vote is posted to the API, the service pushes it to a queue
   for subsequent, asynchronous processing. When a request is made for vote
   results, the service submits a query to a database where votes are ultimately
   stored by a worker processing the queue.
 * **Worker** - this is a background service that watches a queue for votes and stores
   them in a database. The worker represents a typical service component designed
   to scale as needed to handle various asynchronous processing tasks, typically pulled from a queue,
   so that the main service doesn't get bogged down handling requests.
 * **Queue** - this is a service that allows the vote service to post votes without
   slowing down to do any special processing or waiting for a database to save
   votes. The queue is an in-memory data store (using Redis) that enhances performance
   by not requiring the vote service to wait on the database since database operations
   that involve disk I/O are much slower; consequently, the vote service is ready
   to continue to accept new API requests faster. Redis (and similar tools) are
   typical components of many real-world applications that require message queue,
   publish-subscribe, key-value storage, or caching support.
 * **Database** - this service (using MongoDB) provides structured data storage and query
   support. One or more types of database are typical service components of most
   real world applications.


## Launching the app

The process is intentionally manual for the introduction course to allow gaining
familiarity with various processes. We will build on this until ultimately we have
a fully automated processes that starts the app services in a Docker swarm.

### Create a bridge network

Create a bridge network that will be shared by the services for communication with
each other:

    $ docker network create -d bridge demonet

### Start a MongoDB container

MongoDB is a NoSQL database that we will use for storing votes. There is already
an existing image that we can use:

    $ docker run -d --network bridgenet --name database mongo

### Start a Redis container

Redis provides a fast in-memory data store that we will use as a queue. Votes will
be pushed to the queue, where workers will dequeue them asynchronously for saving
to the MongoDB database for subsquent query processing.

Like, MongoDB, there is already an existing image that we can use:

    $ docker run
        --detach \
        --name=queue \
        --network=bridgenet \
        --health-cmd='[ $(redis-cli ping) = "PONG" ] || exit 1'
        --health-timeout=5s \
        --health-retries=5 \
        --health-interval=5s \
        redis

or as one line:

    $ docker run --detach --name=queue --network=demonet --health-cmd='[ $(redis-cli ping) = "PONG" ] || exit 1' --health-timeout=5s --health-retries=5 --health-interval=5s redis

### Start a Vote Worker container

A vote worker pulls votes from the queue and saves them to the database for
subsequent query processing.

You will need to build the image first:

    $ cd worker
    $ docker build -t worker .

Then you can start it:

    $ docker run -d --network=bridgenet --name=worker worker
    
### Start a Vote API container 

The `vote` service provides the API that clients will use to submit votes and fetch
voting tally results. The vote service receives votes that it then pushes to the
queue (where they will subsequently pulled by workers and saved to the database),
and it also queries the database to tally the votes. 

You will need to build the image first:

    $ cd vote
    $ docker build -t vote .

Then you can start it:

    $ docker run -d --network=bridgenet --name=vote vote

### Run a Vote client container

The `vote` app is a terminal program that you run to cast votes and fetch the
voting results.

    $ docker run -it --rm --network=demonet --name=voter subfuzion/voter <cmd>

where `<cmd>` is either `vote` or `results` (if you don't enter any command,
then usage help will be printed to the terminal).

### Run the assessor

The `assessor` is for evaluating the performance of the Voting App
running under Docker. It works by monitoring the logs of each service
for patterns that must be matched to indicate success. The assessor
produces a report when complete or when the evaluation times out. 

See [here](https://github.com/subfuzion/example-voting-app-nodejs/wiki#final-project)
for instructions on running an assessment for the final project.

