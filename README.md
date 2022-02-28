# deephaven-debezium-demo

The docker compose file in this directory starts a compose with images for mysql, Debezium, Redpanda (kafka implementation) and Deephaven, plus an additional image to generate an initial mysql schema and then generate updates to the tables over time for a simple e-commerce demo.

The demo follows closely the one defined for Materialize here:
https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md

The load generation script is in `loadgen/generate_load.py`.

It is possible to configure the update rate for both purchase (mysql updates) and pageviews (kafka pageview events) via ENVIRONMENT arguments set for the loadgen image in the docker-compose.yml file.

### Components

* `docker-compose.yml` - The Docker Compose file for the application. This is mostly the same as the [Deephaven docker-compose file](https://raw.githubusercontent.com/deephaven/deephaven-core/main/containers/python/docker-compose.yml) with modifications to run Redpanda, mysql, debezium and the scripts to generate the simulated website.
* `.env` - The environmental variables used in this demo.
* `scripts/demo.py` - The Deephaven commands used in this demo.
* `scripts/demo.sql` - The Materialize demo script.
* `loadgen/*` - The load generation scripts.

# How to run in Deephaven

First, to run this demo you will need to clone our [github examples repo](https://github.com/deephaven-examples/deephaven-debezium-demo)

```
gh repo clone deephaven-examples/deephaven-debezium-demo
```

To build you need have the these dependances for any Deephaven dockerized initialization such as docker and docker-compose

For more detailed instructions see our [documentation](/core/docs/tutorials/quickstart/).

```
cd deephaven-debezium-demo
docker-compose -f docker-compose.yml up -d
```

Then start a Deephaven web console (will be in python mode by default per the command above) by navigating to

```
http://localhost:10000/ide
```

Cut and paste to it from `/scripts/demo.py`.  

As you cut & paste the script, you can see tables as they are created and populated and watch them update before you execute the next command.

If you want to load everything in one command, however, you can do it as the `demo.py` file is available inside the DH server container under `/scripts/demo.py`.

You can load that in its entirety on the DH console with `exec(open('/scripts/demo.py').read())`

In DH, the `pageviews_summary` table can help track the last pageview seen.


# Attributions

Files in this directory are based on demo code by Debezium, Redpanda, and Materialize

* [Debezium](https://github.com/debezium/debezium)
* [Redpanda](https://github.com/vectorizedio/redpanda)
* [Materialize](https://github.com/MaterializeInc/materialize)
* [Materialize e-commerce demo](https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md)
