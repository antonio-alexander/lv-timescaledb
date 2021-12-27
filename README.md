# lv-timescaledb (github.com/antonio-alexander/lv-timescaledb)

THIS IMPLEMENTATION IS SPECIFIC TO PC/WINDOWS

There are a plethora of options for storing timeseries data with LabVIEW applications. Generally, we find that the least complicated first-party solutions for LabVIEW are often LabVIEW specific (e.g. TDMS) or not scalable (e.g CSV files). Databases are an obvious solution, but they're a language unto themselves: they require some significant domain knowledge.

If you're concerned about using a database versus TDMS/CSVs, understand that databases are limited by CPU, RAM and disk I/O; your ability to get data into and out of a database is dependent on those resources. If it doesn't work, or isn't fast enough, assume you don't know how to "fix" it yet.

[timescaledb](https://www.timescale.com/) is a plugin for [postgres](https://www.postgresql.org/) which provides some high-level functionality to make storage of timeseries data more scalable. In short, it provides the following:

- An ability to seamlessly compress timeseries data so it takes up less space on disk
- Creation of hypertable to logically separate your (hopefully giant) table into multiple smaller tables
- Creation of indexes to optimize your timeseries queries
- Ability to perform data retention as a background process
- Ability to perform compression as a background process

Although the primary goal for this library/package is to provide an API for timescaledb; it's useless/unfair to NOT provide a complete example of how you would distribute it along with a project, look at [github.com/antonio-alexander/lv-timescaledb-example](github.com/antonio-alexander/lv-timescaledb-example) to see a complete implementation.

## postgres/timescaledb server

Its much much much much easier to use Docker for testing/development for deployment of timescaledb; as such i've included a docker compose that gets timescaledb running with a level of persistence (the data will persist as long as the volume isn't deleted).

You can also install postgresql AND the timescaledb driver locally (or on another machine), look at the timescale and postgres websites for instructions.

If you're running a setup like myself where you're running a linux-based host and windows within virtualbox/vmware you'll need socat to expose the docker daemon via TCP IF you want to be able to access the docker daemon from Windows:

```yaml
socat:
  container_name: "socat"
  hostname: "socat"
  image: bobrik/socat
  command: ["TCP-LISTEN:2375,fork", "UNIX-CONNECT:/var/run/docker.sock"]
  restart: "always"
  ports:
    - "2375:2375"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

## Installation/Pre-requisites

The API builds upon the built-in database connectivity toolkit, you'll need to install an ODBC driver [https://odbc.postgresql.org/](https://odbc.postgresql.org/) in order to communicate specifically with postgres. Be aware that you'll need to bundle this with wherever your application lives, it'll need to be installed (and provided in the connection string). Although Docker is the easier way to get everything up and running, you can also follow these instructions: [https://docs.timescale.com/install/latest/](https://docs.timescale.com/install/latest/) to get timescale and postgres.

Keep in mind that you DO NOT need to have postgres (timescale) installed on the same machine you're running the API on..

For examples about how to handle distribution and implementation see: [http://github.com/antonio-alexander/lv-timescaledb-example](http://github.com/antonio-alexander/lv-timescaledb-example)

## Getting Started

You need a running postgres server installation with the timescaledb plugin installed, the easiest way to get things running is to use docker (see this [docker-compose.yml](./docker/docker-compose.yml)), if you wish to install it manually see: [https://docs.timescale.com/install/latest/](https://docs.timescale.com/install/latest/).

The expected use case for this API is for you to create a table, then create a hyper table based on that table, enable compression and finally read/write data. If policies are set timescaledb will periodically compress chunks, in the event you need to modify/backfill data that already exists or is within a chunk that's compressed, you'll need to de-compress the chunk.

## API

The API uses the database and connectivity toolkit exclusively, if you're integrating into the database using an alternate method (that can talk to Postgres), this API **may not** work as expected. The API is opinionated and doesn't implement the entire API, just (what I think is) the minimum requirements to work with it. We implement the following API:

- [add_compression_policy.vi](https://docs.timescale.com/api/latest/compression/add_compression_policy/)
- [add_retention_policy.vi](https://docs.timescale.com/api/latest/data-retention/add_retention_policy/)
- [compress_chunk.vi](https://docs.timescale.com/api/latest/compression/compress_chunk/)
- [create_hyper_table.vi](https://docs.timescale.com/api/latest/hypertable/create_hypertable/#create-hypertable)
- [decompress_chunk.vi](https://docs.timescale.com/api/latest/compression/decompress_chunk/)
- [disable_compression.vi](https://docs.timescale.com/api/latest/compression/alter_table_compression/)
- [drop_chunk.vi](https://docs.timescale.com/api/latest/hypertable/drop_chunks/)
- [enable_compression.vi](https://docs.timescale.com/api/latest/compression/alter_table_compression/)
- [remove_compression_policy.vi](https://docs.timescale.com/api/latest/compression/remove_compression_policy/)
- [remove_retention_policy.vi](https://docs.timescale.com/api/latest/data-retention/remove_retention_policy/)
- [set_chunk_time_interval.vi](https://docs.timescale.com/api/latest/hypertable/set_chunk_time_interval/)
- [show_chunks.vi](https://docs.timescale.com/api/latest/hypertable/show_chunks/)
- show_hypertables.vi - this is a "custom" VI that queries timescaledb tables

The API is implemented with the following in mind:

- It DOES NOT support concurrent calls (all VIs are non re-entrant)
- It uses parameters (where possible) specifically to mitigate SQL injection where strings are used as inputs
- Any operations that involve timestamps expect UTC (see note below regarding timestamps and LabVIEW)

The API is mostly a proof of concept or useful in a situation where you'd want to create some UI around administrative functions, generally MOST of this API would have zero use cases within an application being consumed by non-administrators. Most of the API is **currently** safe from SQL injection, except for APIs that enable or disable compression.
