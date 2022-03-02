# deephaven-debezium-demo

The demo follows closely the one defined for [Materialize](https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md). We want to showcase how you can accomplish the same workflow in Deephaven, with lightning-fast performance.

The docker-compose file in this directory starts a compose with images for MySQL, [Redpanda](https://redpanda.com/), [Debezium](https://debezium.io/), and Deephaven, plus an additional image to generate an initial MySQL schema and then generate updates to the tables over time for a simple e-commerce demo.

![img](./debezium.png)


### Components

* `docker-compose.yml` - The Docker Compose file for the application. This is mostly the same as the [Deephaven docker-compose file](https://raw.githubusercontent.com/deephaven/deephaven-core/main/containers/python/docker-compose.yml) with modifications to run Redpanda, MySQL, Debezium, and the scripts to generate the simulated website.
* `.env` - The environmental variables used in this demo.
* `scripts/demo.py` - The Deephaven commands used in this demo.
* `scripts/demo.sql` - The Materialize demo script.
* `loadgen/*` - The load generation scripts.

Configure the update rate for both purchase (MySQL updates) and pageviews (Kafka pageview events) via ENVIRONMENT arguments set for the loadgen image in the docker-compose.yml file.

## Quick Start

First, to run this demo you will need to clone our [github examples repo](https://github.com/deephaven-examples/deephaven-debezium-demo):

```
gh repo clone deephaven-examples/deephaven-debezium-demo
```

To build, you need have the these dependancies for any Deephaven dockerized initialization such as docker and docker-compose.

For more detailed instructions, see our [Quickstart guide](/core/docs/tutorials/quickstart/).

1. Launch via Docker:

```
cd deephaven-debezium-demo
docker-compose -f docker-compose.yml up -d
```

2. Then start a [Deephaven web console](http://localhost:10000/ide) (this will be in Python mode by default, per the command above) by navigating to:

```
http://localhost:10000/ide
```

3. Copy and paste to it from `/scripts/demo.py`.  

As you copy and paste the script, you can see tables as they are created and populated and watch them update before you execute the next command.  Details below.


## Details to implement

The numbered steps below are from the demo defined for the [Redpanda + Materialize Demo](https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md).

4. _Optional_ Confirm that everything is running as expected:

```shell session
docker stats
```

5. _Optional_ Log in to MySQL to confirm that tables are created and seeded:

```shell
docker-compose -f docker-compose.yml run mysql mysql -uroot -pdebezium -h mysql shop
```

```sql
SHOW TABLES;

SELECT * FROM purchases LIMIT 1;
```

6. _Optional_ `exec` in to the `redpanda` container to look around using Redpanda's amazing `rpk` CLI:

```shell session
docker-compose -f docker-compose.yml exec redpanda /bin/bash

rpk debug info

rpk topic list

rpk topic create dd_flagged_profiles

rpk topic consume pageviews
```

You should see a live feed of JSON formatted pageview Kafka messages:

```
{
    "key": "3290",
    "message": "{\"user_id\": 3290, \"url\": \"/products/257\", \"channel\": \"social\", \"received_at\": 1634651213}",
    "partition": 0,
    "offset": 21529,
    "size": 89,
    "timestamp": "2021-10-19T13:46:53.15Z"
}
```


### Deephaven Commands

You have now launched a docker-compose file that starts images for MySQL, Debezium, Redpanda (Kafka implementation), and Deephaven, plus an additional image to generate an initial MySQL schema and then generate updates to the tables over time for a simple e-commerce demo.

Here, we will show you the analogous Deephaven commands and continue the bullet numbers from the [Materialize Demo](https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md). You can follow along in the [Deephaven IDE](http://localhost:10000/ide) by entering the commands into theconsole.

7. Now that you're in the Deephaven IDE, define all of the tables in `mysql.shop` as Kafka sources:

```python skip-test
import deephaven.ConsumeCdc as cc
import deephaven.ConsumeKafka as ck
import deephaven.ProduceKafka as pk
import deephaven.Types as dh
from deephaven import Aggregation as agg, as_list
import deephaven.TableManipulation.WindowCheck as wck

server_name = 'mysql'
db_name='shop'

kafka_base_properties = {
    'group.id' : 'dh-server',
    'bootstrap.servers' : 'redpanda:9092',
    'schema.registry.url' : 'http://redpanda:8081',
}


def make_cdc_table(table_name:str):
    return cc.consumeToTable(
        kafka_base_properties,
        cc.cdc_short_spec(server_name,
                          db_name,
                          table_name)
    )

users = make_cdc_table('users')
items = make_cdc_table('items')
purchases = make_cdc_table('purchases')

consume_properties = {
    **kafka_base_properties,
    **{
        'deephaven.partition.column.name' : '',
        'deephaven.timestamp.column.name' : '',
        'deephaven.offset.column.name' : ''
    }
}

pageviews = ck.consumeToTable(
    consume_properties,
    topic = 'pageviews',
    offsets = ck.ALL_PARTITIONS_SEEK_TO_BEGINNING,
    key = ck.IGNORE,
    value = ck.json([ ('user_id', dh.long_),
                      ('url', dh.string),
                      ('channel', dh.string),
                      ('received_at', dh.datetime) ]),
    table_type = 'append'
)
```

Because the first three sources are pulling message schema data from the registry, Deephaven knows the column types to use for each attribute. The last source is a JSON-formatted source for the page views.

Now you should _automatically_ see the four sources we created in the IDE. These are fully interactable. In the UI, you can [sort](/core/docs/how-to-guides/user-interface/work-with-columns/), [filter](/core/docs/how-to-guides/user-interface/filters/) or scroll through _all_ the data without any other commands.

![img](./debezium1.png)

8. Next, we'll create a table for staging the page views. We can use this to aggregate information later.

```python skip-test
pageviews_stg = pageviews \
    .updateView(
        'url_path = url.split(`/`)',
        'pageview_type = url_path[1]',
        'target_id = Long.parseLong(url_path[2])'
    ).dropColumns('url_path')
```

### Analytical views

9. Let's create a couple analytical views to get a feel for how it works.

Start simple with a table that aggregates purchase stats by item:

```python skip-test
purchases_by_item = purchases.aggBy(
    as_list([
        agg.AggSum('revenue = purchase_price'),
        agg.AggCount('orders'),
        agg.AggSum('items_sold = quantity')
    ]),
    'item_id'
)
```

The next query creates something similar that uses our `pageview_stg` static view to quickly aggregate page views by item:

```python skip-test
pageviews_by_item = pageviews_stg \
    .where('pageview_type = `products`') \
    .countBy('pageviews', 'item_id = target_id')
```

Now let's show how you can combine and stack views by creating a single table that brings everything together:

```python skip-test
item_summary = items \
    .view('item_id = id', 'name', 'category') \
    .naturalJoin(purchases_by_item, 'item_id') \
    .naturalJoin(pageviews_by_item, 'item_id') \
    .dropColumns('item_id') \
    .moveColumnsDown('revenue', 'pageviews') \
    .updateView('conversion_rate = orders / (double) pageviews')
```

We can _automatically_ see that it's working by watching these tables update in the IDE. To see just the top elements, we can filter the data with two new tables:

```python skip-test
top_viewed_items = item_summary \
    .sortDescending('pageviews') \
    .head(20)

top_converting_items = item_summary \
    .sortDescending('conversion_rate') \
    .head(20)
```

![img](./debezium1.gif)

Another useful table is `pageviews_summary` that counts the total number of pages seen:

```python skip-test
pageviews_summary = pageviews_stg \
    .aggBy(
        as_list([
            agg.AggCount('total'),
            agg.AggMax('max_received_at = received_at')])) \
    .updateView('dt_ms = (DateTime.now() - max_received_at)/1_000_000.0')
```

### User-facing data views

10.  [Redpanda](https://redpanda.com/) is often used in building rich data-intensive applications. Let's try creating a view meant to power something like the "Who has viewed your profile" feature on Linkedin:

User views of other user profiles:

```python skip-test
minute_in_nanos = 60 * 1000 * 1000 * 1000

profile_views_per_minute_last_10 = \
    wck.addTimeWindow(
        pageviews_stg.where('pageview_type = `profiles`'),
        'received_at',
        10*minute_in_nanos,
        'in_last_10min'
    ).where(
        'in_last_10min = true'
    ).updateView(
        'received_at_minute = lowerBin(received_at, minute_in_nanos)'
    ).view(
        'user_id = target_id',
        'received_at_minute'
    ).countBy(
        'pageviews',
        'user_id',
        'received_at_minute'
    ).sort(
        'user_id',
        'received_at_minute'
    )
```

Confirm that this is the data we could use to populate a "profile views" graph for user `10`:

```python skip-test
profile_views = pageviews_stg \
    .view(
        'owner_id = target_id',
        'viewer_id = user_id',
        'received_at'
    ).sort(
        'received_at'
    ).tailBy(10, 'owner_id')
```

Next, let's use a `naturalJoin` to get the last five users who have viewed each profile:

```python skip-test
profile_views_enriched = profile_views \
    .naturalJoin(users, 'owner_id = id', 'owner_email = email') \
    .naturalJoin(users, 'viewer_id = id', 'viewer_email = email') \
    .moveColumnsDown('received_at')
```

### Demand-driven query

11. Since Redpanda has such a nice HTTP interface, it makes it easier to extend without writing lots of glue code and services. Here's an example where we use pandaproxy to do a "demand-driven query".

Add a message to the `dd_flagged_profiles` topic:

```python skip-test
dd_flagged_profiles = ck.consumeToTable(
    consume_properties,
    topic = 'dd_flagged_profiles',
    offsets = ck.ALL_PARTITIONS_SEEK_TO_BEGINNING,
    key = ck.IGNORE,
    value = ck.simple('user_id_str', dh.string),
    table_type = 'append'
).view('user_id = Long.parseLong(user_id_str.substring(1, user_id_str.length() - 1))')  # strip quotes
```

Now let's join the `flagged_profile` id to a much larger dataset:

```python skip-test
dd_flagged_profile_view = dd_flagged_profiles \
    .join(pageviews_stg, 'user_id')
```

12. Sink data back out to Redpanda.

Let's create a view that flags "high-value" users that have spent $10k or more total:

```python skip-test
high_value_users = purchases \
    .updateView(
        'purchase_total = purchase_price * quantity'
    ).aggBy(
        as_list([
            agg.AggSum('lifetime_value = purchase_total'),
            agg.AggCount('purchases'),
        ]),
        'user_id'
    ) \
    .where('lifetime_value > 10000') \
    .naturalJoin(users, 'user_id = id', 'email') \
    .view('id = user_id', 'email', 'lifetime_value', 'purchases')  # column rename and reorder
```

Then, a sink to stream updates to this view back out to Redpanda:

```python skip-test
schema_namespace = 'io.deephaven.examples'

cancel_callback = pk.produceFromTable(
    high_value_users,
    kafka_base_properties,
    topic = 'high_value_users_sink',
    key = pk.avro(
        'high_value_users_sink_key',
        publish_schema = True,
        schema_namespace = schema_namespace,
        include_only_columns = [ 'user_id' ]
    ),
    value = pk.avro(
        'high_value_users_sink_value',
        publish_schema = True,
        schema_namespace = schema_namespace,
        column_properties = {
            "lifetime_value.precision" : "12",
            "lifetime_value.scale" : "4"
        }
    ),
    last_by_key_columns = True
)
```

This is a bit more complex because it is an `exactly-once` sink. This means that across Deephaven restarts, it will never output the same update more than once.

We won't be able to preview the results with `rpk` because it's AVRO formatted. But we can actually stream it BACK into Deephaven to confirm the format!

```python skip-test
hvu_test = ck.consumeToTable(
    consume_properties,
    topic = 'high_value_users_sink',
    offsets = ck.ALL_PARTITIONS_SEEK_TO_BEGINNING,
    key = ck.IGNORE,
    value = ck.avro('high_value_users_sink_value'),
    table_type = 'append'
)
```

## Conclusion

You now have Deephaven doing real-time views on a changefeed from a database and page view events from Redpanda. You have complex multi-layer views doing joins and aggregations in order to distill the raw data into a form that's useful for downstream applications.

You have a lot of infrastructure running in Docker containers - don't forget to run `docker-compose down` to shut everything down!

Next time, if you want to load everything in one command, you can load that in its entirety on the DH console with `exec(open('/scripts/demo.py').read())`


## Taking it further

You've how fun and easy it is to use Deephaven. Within the IDE, you can see your query results and interact with all the data. You can take this even further with Deephaven's plotting capabailities, such as by visualizing the aggregrate page views in real time.


# Attributions

Files in this directory are based on demo code by Debezium, Redpanda, and Materialize:

* [Debezium](https://github.com/debezium/debezium)
* [Redpanda](https://github.com/vectorizedio/redpanda)
* [Materialize](https://github.com/MaterializeInc/materialize)
* [Materialize e-commerce demo](https://github.com/MaterializeInc/ecommerce-demo/blob/main/README_RPM.md)
