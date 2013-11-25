---
title: "Docker Links and Runtime Environment Variables"
created_at: 2013-11-24 11:03:00 -0800
kind: article
---

Linking is a recent feature added to Docker in the 0.6.5 release on October 31. Passing the `-link`
argument to `docker run` commands will enable a new container to access an existing container's
exposed ports via environment variables.

To link containers, start a container with the `-name` argument and use the name when starting the
second container like this:

    docker run -name first -p 5432 -d <container hash>
    docker run -name second -link first:db <hash>

When used correctly, the "second" container will have additional environment variables exposed that
describe the IP address and port of the mapped container, e.g.:

    DB_NAME=/violet_wolf/db
    DB_PORT_6379_TCP_PORT=6379
    DB_PORT=tcp://172.17.0.33:6379
    DB_PORT_6379_TCP=tcp://172.17.0.33:6379
    DB_PORT_6379_TCP_ADDR=172.17.0.33
    DB_PORT_6379_TCP_PROTO=tcp

The string after the `:` value is used as the prefix for the set of environment variables added to
the second container.

Using these environment variables is non-trivial; the additional variables may not match up to the
values your application is anticipating, so some transformation may be necessary. For example, if
your application is expecting `SQLALCHEMY_DATABASE_URI` to contain the database URI, you'll need to
use the now-exposed DB_PORT_XXX_TCP_ADDR to set SQLALCHEMY_DATABASE_URI.

You might be tempted to try the `ENV` parameter available for Dockerfile, but this won't work; the
environment variables exposed by `-link` arguments are only available inside a running docker
container, not via the Dockerfile.

To solve this, I've opted to use a wrapper script called `env.sh` in each repo to take variables
exposed by linked containers and create new ones that the application expects, `exec`'ing the
remaining arguemnts. The Dockerfile for each application sets the ENTRYPOINT as `env.sh`.

    ENTRYPOINT ["/path/to/env.sh"]

Here is the `env.sh` wrapper script that is used at the ENTRYPOINT for each repo's container.

[Environment variable wrapper script - env.sh](https://gist.github.com/mikeclarke/7620336)
<script src="https://gist.github.com/mikeclarke/7620336.js"></script>

*Example Usage*

    DB_NAME="/$JOB_NAME-$BUILD_NUMBER-db"
    DB_CONTAINER=$(sudo docker run -d -name $DB_NAME <private repository URL>/database)
    sudo docker run -link $DB_NAME:db -t $JOB_NAME/$BUILD_NUMBER nosetests

In my next post, I'll describe how this wrapper script is used to expose services in a continuous
integration flow.
