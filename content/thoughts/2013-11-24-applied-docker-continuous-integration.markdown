---
title: "Applied Docker: Continuous Integration"
created_at: 2013-11-24 12:03:00 -0800
kind: article
---

*tl;dr*

* Docker and Jenkins (or any similar tool) are a perfect match for building continuous integration
* The `Dockerfile` empowers developers to manage their own OS package requirements
* Containers as build artifacts significantly simplifies integration scenarios
* [12-factor app](http://12factor.net) best practices fall out naturally from this strategy for
  continuous integration

*Introduction*

Make no mistake; [Docker](http://www.docker.io/) is the new hotness, but there aren't many clear
explanations of where Docker fits best with modern web stacks today. Frequently I'll encounter
comments along the lines of "Disclaimer: I've never tried using Docker, but have you tried it yet?"
or "I'm no expert on LXC or docker (never used it, but I've read a lot about it) but ..." on Hacker
News.

Some of my close friends in the industry have shared similar thoughts - in a world where best
practices around configuration management, deployment, and virtualization are well-established, why
is Docker interesting?

<blockquote class="twitter-tweet" lang="en">
  <p>It just hit me: Docker is the new curl | sh</p>&mdash; Brett Hoerner (@bretthoerner)&nbsp;
  <a href="https://twitter.com/bretthoerner/statuses/395237114331148288">October 29, 2013</a>
</blockquote>

Like any other tool, Docker offers a solution to a specific class of challenges; in particular,
Docker exposes a Git-like interface to building and executing lightweight
[LXC containers](http://linuxcontainers.org). One direct application of Linux containers is
powering Heroku-style PaaS runtime environments - it's a natural extension of the Docker project's
history.

However, most of us aren't building Heroku knockoffs (nor is it clear that deploying
your production application via a `git push prod master` ad-hoc strategy is a good idea). Since
most companies are not Heroku or Google, where else can Docker provide value in cutting edge web
architectures? Right now, there aren't many resources online showing "real-life" applied Docker use
cases for the benefit of others.

In this post, I'd like to describe my recent implementation of continuous integration at
[Standard Treasury](http://standardtreasury.com) as an example of Docker's strengths. Our stack is
a traditional Flask/SQLAlchemy web architecture, with dependencies on Postgres and RabbitMQ -
fairly typical of many teams, though the content of this post is applicable regardless of language
or framework.

*Overview*

The continuous integration flow I've selected works like this:

* Build a Docker container and mount the repo inside (via `ADD` in a Dockerfile)
* Execute tests inside the container, starting & linking external resources as needed
* Release build artifacts - this can mean one of two paths.
  * For shared libraries, push to private PyPI index, commit a git tag, etc.
  * For web applications, tag the container and push to private Docker repository
* Clean up temporary containers, remove docker images

My strategy for continuous integration is to isolate test runs by building a new container for each
commit. I'll be using Jenkins in the examples here, but any build tool that can access your git
repository and
[execute docker commands via sudo](http://blog.docker.io/2013/08/containers-docker-how-secure-are-they/)
should work with this approach. Each repository that I'll be testing will include its own
`Dockerfile`, which will install shared libraries, 3rd-party dependencies, and otherwise perform
system setup required for test runs to pass.

For tests that require an external resource (database, redis, cassandra, etc), the build tool will
start a separate Docker container and link it to the test execution container. Linking is a recent
feature added to Docker in the 0.6.5 release on October 31. Some advantages of launching external
services as containers include shared-nothing service isolation and simplicity starting / destroying
services.

I've opted to use a wrapper script called `env.sh` in each repo to take variables
exposed by linked containers and create new ones that the application expects, `exec`'ing the
remaining arguemnts. The Dockerfile for each application sets the ENTRYPOINT as `env.sh`.

*Build*

The build step is a single shell command:

    sudo docker build -t $JOB_NAME/$BUILD_NUMBER .

Each repo contains a top-level Dockerfile that specifies all the steps required to provision a
Linux OS from a vanilla image. Since each step is cached, each test run after the initial one
should be very fast. Since we have lots of repositories, I've opted to create a base python
container that each application inherits (via the `FROM` Dockerfile directive).

[Base python Dockerfile](https://gist.github.com/mikeclarke/7620172)
<script src="https://gist.github.com/mikeclarke/7620172.js"></script>

The application-specific Dockerfile is largely uncached, since every RUN step after the ADD will
need to execute for each test run.

[Application-specific Dockerfile](https://gist.github.com/mikeclarke/7620210)
<script src="https://gist.github.com/mikeclarke/7620210.js"></script>

The ENTRYPOINT script `env.sh` in this example is used to map the Docker-exposed environment
variables to the values our application expects. I've covered this in detail
[in my previous blog post](/2013/11/docker-links-and-runtime-env-vars/).

*Test*

The test step is also a short shell script:

    DB_NAME="/$JOB_NAME-$BUILD_NUMBER-db"
    DB_CONTAINER=$(sudo docker run -d -name $DB_NAME <private repository URL>/database-schema)
    sudo docker run -link $DB_NAME:db -t $JOB_NAME/$BUILD_NUMBER nosetests

In this example, the database-schema container represents the latest tagged Postgres schema from
a previous test run (against the repository that tracks our schema changes). In this way, tests
can layer on top of each other to ensure commits integrate properly with other repositories.

*Release*

As mentioned earlier, the release step either tags the release via `git` or tags the container as
the latest stable release container, pushing the tag to the remote private docker repository.

    sudo docker tag $JOB_NAME/$BUILD_NUMBER <private repository URL>/${JOB_NAME}-master
    sudo docker push <private repository URL>/${JOB_NAME}-master >/dev/null

*Cleanup*

Not much interesting in this step, but you'll want to `docker kill` running containers needed only
for the test run, `docker rm` killed containers, and `docker rmi` the built image.

    sudo docker kill ...
    sudo docker rm ...
    sudo docker rmi ...

*Conclusion*

As a non-1.0 tool, Docker is still a young project with a lot of potential. Testing infrastructure
is a great place to dip your toe into the LXC ecosystem in a low-risk, high-value way. The testing
strategy discussed in this post capitalizes on the best of what Docker offers.

The final *complete* testing script that you can add as a shell script to Jenkins is this:

[Full Jenkins test script](https://gist.github.com/mikeclarke/7636835)
<script src="https://gist.github.com/mikeclarke/7636835.js"></script>

Another nice feature of this approach is that it enforces best practices from the
[12 factor application manifesto](http://12factor.net/). For your tests to connect to the linked
services, environment variables *must* be used to configure your application. This addresses both
[config](http://12factor.net/config) and [backing services](http://12factor.net/backing-services)
recommendations. If you chose to run the tested container in production, you'll be far along the
path towards meeting the requirements of [dev/prod parity](http://12factor.net/dev-prod-parity).
Even if you opt not to use docker in production, the interface docker enforces will be an asset
regardless of deployment technique.

Recommendations:

* Use a wrapper shell script as your Dockerfile ENTRYPOINT to manipulate environment variables
  exposed via `-link`'ed containers.
* Push service containers into a private Docker repository
* Tag stable containers once tests pass successfully
* For integration testing, use the latest stable tagged containers (and make assertions as needed)

*Further reading*

* [How Are You Blog - Continuous Delivery with Docker and Jenkins - part I](http://blog.howareyou.com/post/62157486858/continuous-delivery-with-docker-and-jenkins-part-i)
* [How Are You Blog - Continuous Delivery with Docker and Jenkins - part II](http://blog.howareyou.com/post/65048170054/continuous-delivery-with-docker-and-jenkins-part-ii)
