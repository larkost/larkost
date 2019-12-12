Setting up APM in `docker-compose`
========================

In one of the projects for work I decided I needed to redirect local dev setups in docker
to target all local services rather than target production ones (even with a "dev" setting).
Since everything else was going to be wrapped up in Docker containers (sensibly) I decided that
these would similarly be. This particular project uses Elastic.co's APM (Application Performance
Monitoring) to do most of its logging from various distributed parts, so I needed that to complete
the setup.

It is worth emphasising that I am only setting this up for testing purposes. I don't think that
there are any license problems here, but I am not doing any of the things a real production setup
would need (e.g.: kernel and docker setting changes to make this work at scale).

I am using the 7.5.0 version of the "Elastic Stack", and based on the documentation I have read the
setup on the 6.x versions is a bit different. I am also using the docker containers from 
[the Elastic.co website](https://www.docker.elastic.co).

Difficulties
------------

Being a neophyte in the "Elastic Stack" I probably shot my self in the foot more often than I should
have in this project, but there were some things that could have been better documented in the process:
1. [APM](https://www.elastic.co/products/apm) does require obvious once you get going, but should be
   more clear (e.g.: a compose setup would be a good idea rather then the `docker run` version given
   as an example).
2. There are a number of different license levels, and it seems that all of the products are licensed
   by looking to the ElasticSearch server for its level. That in turn defaults to the `open source`
   license.
3. APM (and maybe Kibana?) require at least the `basic` license to be in effect on the ElasticSearch
   server (which defaults to `open source`). Fortunately you can set ElasticSearch into `basic` mode
   with a simple settings change (done below), without needing to do things like register.

You should note that I am not taking care of making sure that ElasticSearch comes up before starting
the two other services. A proper setup would wait until it is up (with `entrypoint` scripts) before
letting them launch, but (at least for now) their retry mechanisms are working for me. If your system
is more loaded (or my CI system gets slow) then someone may need to revisit this.

Also note that this `compose` file does not fully setup Kibana. Particularly the index on the APM data
needs to be created. How to do this is not the most obvious to me, but it is readily done in the web 
interface. But since I am working on a testing system, automating that is out-of-scope for the moment.

The files file
----------------

Here is the meat of the article. It is a bit anti-climactic at 25 lines, but it works (for me). Note
that my real `compose` file has the rest of my service in this file with it, and a few other tweaks.

```docker-compose
version '3.7'
services:
  apm:
    image: docker.elastic.co/apm/apm-server:7.5.0
    environment:
      - output.elasticsearch.hosts=["elasticsearch:9200"]
    # volumes:
    #  - ./apm-server.docker.yml:/usr/share/apm-server/apm-server.yml:ro
    # uncomment above if you need more setup, and provide your apm-server.yml file
    ports:
      - "8200:8200"

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
    environment:
      - discovery.type=single-node
      - xpack.license.self_generated.type=basic
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:7.5.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
```