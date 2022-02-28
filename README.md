# deephaven-debezium-demo

The docker compose file in this directory starts a compose with images for mysql, Debezium, Redpanda (kafka implementation) and Deephaven, plus an additional image to generate an initial mysql schema and then generate updates to the tables over time for a simple e-commerce demo.

The demo follows closely the one defined for Materialize here:
https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md

The load generation script is in `loadgen/generate_load.py`.

It is possible to configure the update rate for both purchase (mysql updates) and pageviews (kafka pageview events) via ENVIRONMENT arguments set for the loadgen image in the docker-compose.yml file.

# How to run in Deephaven

First, to run this demo you will need to clone our [github examples repo](https://github.com/deephaven-examples/deephaven-debezium-demo)

```
gh repo clone deephaven-examples/deephaven-debezium-demo
```

To build you need have the these dependances for any Deephaven dockerized initialization such as docker and docker-compose

For more detailed instructions see our [documentation](/core/docs/tutorials/quickstart/).

```
cd deephaven-debezium-demo
docker-compose up -d
```

Then start a Deephaven web console (will be in python mode by default per the command above) by navigating to

```
http://localhost:10000/ide
```

and cut & paste to it from `/scripts/demo.py`.  

Suggesting that you cut&paste instead of automatically setting the script to run is intentional, so that you can see tables as they are created and populated and watch them update before you execute the next command.

If you want to load everything in one command, however, you can do it as the `demo.py` file is available inside the DH server container under `/scripts/demo.py`.

You can load that in its entirety on the DH console with `exec(open('/scripts/demo.py').read())`

In DH, the `pageviews_summary` table can help track the last pageview seen.

# How to run in Materialize

The file `debezium/scripts/demo.sql` contains the original
[Materialize script](https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md)


To load the Materialize script, run the materialized command line interface (cli) via:
 
 `docker-compose run mzcli`

Once in the materialize cli, run:
`\i /scripts/demo.sql`

You should see a number of `CREATE SOURCE` and `CREATE VIEW` lines of output.
  
  
If the host has `psql` installed, we can use the shell watch command to run a select statement to can help track the last pageview seen:
```
watch -n1 "psql -c '
SELECT
  total,
  to_timestamp(max_received_at) max_received_ts,
  mz_logical_timestamp()/1000.0 AS logical_ts_ms,
  mz_logical_timestamp()/1000.0 - max_received_at AS dt_ms
FROM pageviews_summary;'  -U materialize -h localhost -p 6875
```

# Memory and CPU requirements

The parameters used for images in the docker compose file in this directory are geared towards high message throughput.  While Deephaven itself is running with the same default configuration used for general demos (as of this writing, 4 cpus and 4 Gb of memory), the configurations for redpanda, mysql, and debezium are tweaked to reduce their impact in end-to-end latency and throughput measurements; we make extensive use of RAM disks (tmpfs) and increase some parameters to ones closer to production (e.g., redpanda's number of cpus and memory per core).  To get a full picture of the configuration used, consult the files:

- `.env`
- `docker-compose.yml`

Once started the compose will take around 6 Gb of memory from the host; as events arrive and specially if event rates are increased, it will increase to 10-16 Gb or more.

For increased event rates (eg, 50,000 pageviews per second), CPU utilization will spike to 14 CPU threads or more.

# Attributions

Files in this directory are based on demo code by Debezium, Redpanda, and Materialize

* [Debezium](https://github.com/debezium/debezium)
* [Redpanda](https://github.com/vectorizedio/redpanda)
* [Materialize](https://github.com/MaterializeInc/materialize)
* [Materialize e-commerce demo](https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md)
