---
title: "How to deploy Opensearch with Docker Compose and query it using Python"
publish: true
date: 2022-08-04T00:00:00+02:00
---

Imagine that you’re working with a large text dataset. It can be anything:

- a set of tweets
- scraped articles

You have a client that wants to find some information in this dataset or do some analysis with it. You probably cannot fit it inside a single Pandas data frame and query it. That’s where tools like OpenSearch and OpenSearch Dashboards can be useful. You can write a query for a specific term, set of terms or even do fuzzy matching. Not to mention plugins which can do even more useful stuff.

What’d like to show you in this article is:

- Setting up OpenSearch instance using Docker Compose
- Ingesting a sample dataset
- Querying OpenSearch from Python

## What’s OpenSearch

OpenSearch is a data store and a search engine which can be connected with a visualization interface OpenSearch Dashboards to quickly create visualizations and analyze your dataset. OpenSearch can be combined with tools like Logstash for easy data ingestion from databases like PostgreSQL using JDBC or RabbitMQ using RabbitMQ plugin.

It frees us from managing our data. All we do is put it in the database and that’s it — no need to worry about not fitting it inside RAM. It supports development using Docker — there are many examples on how to set it up.

## Setting up OpenSearch

The easiest way to set up OpenSearch is to use Docker and Docker compose. Here’s a link with all the instructions on how to install it:

- [Docker](https://docs.docker.com/engine/install/)
- [Docker compose](https://docs.docker.com/compose/install/)

After installation, create an empty directory and a file docker-compose.yaml Inside this file put this code:

```
version: '3'
services:
  opensearch-node1:
    image: opensearchproject/opensearch:2.1.0
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_master_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true # along with the memlock settings below, disables swapping
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # minimum and maximum Java heap size, recommend setting both to 50% of system RAM
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536 # maximum number of open files for the OpenSearch user, set to at least 65536 on modern systems
        hard: 65536
    volumes:
      - opensearch-data1:/usr/share/opensearch/data
    ports:
      - 9200:9200
      - 9600:9600 # required for Performance Analyzer
    networks:
      - opensearch-net
  opensearch-node2:
    image: opensearchproject/opensearch:2.1.0
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node2
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_master_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - opensearch-data2:/usr/share/opensearch/data
    networks:
      - opensearch-net
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:2.1.0
    container_name: opensearch-dashboards
    ports:
      - 5601:5601
    expose:
      - "5601"
    environment:
      OPENSEARCH_HOSTS: '["https://opensearch-node1:9200","https://opensearch-node2:9200"]' # must be a string with no spaces when specified as an environment variable
    networks:
      - opensearch-net

volumes:
  opensearch-data1:
  opensearch-data2:

networks:
  opensearch-net:
```

If you’re working on linux/WSL there can be a problem (I found the solution [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html)), OpenSearch will not start unless you type this command first (as root):

```sh
sysctl -w vm.max_map_count=262144
```

Now you can start your OpenSearch instance with:

```sh
docker-compose up
```

If nothing went wrong, you should be able to access OpenSearch Dashboards on: http://localhost:5601/ . Login and password: admin

You can play with it, but I won’t use Dashboards in this article.

### Ingesting a sample dataset

Dataset that I’ll be using can be found here: [Kaggle link](https://www.kaggle.com/datasets/shuyangli94/food-com-recipes-and-user-interactions?select=RAW_recipes.csv) it’s a dataset contains recipes. Unzip it and copy RAW_recipes.csv to work directory.

To ingest data we’ll use Pandas chunk feature and opensearch-py client. Install python requirements:

```sh
pip install pandas opensearch-py
```

Ingesting script will use bulk API ([link](https://opensearch-project.github.io/opensearch-py/api-ref/client.html?highlight=bulk#opensearchpy.OpenSearch.bulk)). The idea behind this script is very simple:

- load part of data
- upload it
- repeat until EOF

I’ve disabled verification of cert for dev purposes, never do that in production ;)

```python
from opensearchpy import OpenSearch, helpers
import pandas as pd
FILENAME = "RAW_recipes.csv"
client = OpenSearch(
  hosts = ['https://admin:admin@localhost:9200/'],
  http_compress = True,
  use_ssl = True,
  verify_certs = False,       # DONT USE IN PRODUCTION
  ssl_assert_hostname = False,
  ssl_show_warn = False
)
chunksize = 10 ** 5
with pd.read_csv(
  FILENAME,
  chunksize=chunksize,
  usecols=["name", "id", "description"]
) as reader:
  for i, chunk in enumerate(reader):
    chunk.fillna("", inplace=True)
    docs = chunk.to_dict(orient='records')
    print(f"Uploading chunk nr {i}, total:", i*chunksize)
    helpers.bulk(client, docs, index='recipes', raise_on_error=True, refresh=True)
```

### Querying OpenSearch from Python

We’ll reuse `client` object. Match queries can be performed using `client.search` — that’s how we query our search engine. Here’s example code that queries it:

```python
from pprint import pprint
query = {
    "size": 2, # Return 2 results
    "query": {
        "match": {
            "name": "chocolate cake"
        }
    }
}
response = client.search(
    body = query,
    index = "recipes" # the same as before
)
pprint(response["hits"]["hits"]
```

And sample response:

![[0960a7eb9f7a3840898293b032108b9a_MD5.png]]

As you can see, we’ve got a match. Bot name and descripion fields mention words of interest. If you’d want to increase the number of returned documents, change the value of size parameter in query.

## Summary

In this article, we’ve learned:

- What’s OpenSearch and how to set it up using Docker compose
- How to upload example dataset to OpenSearch with Bulk API
- How to perform match query — our primitive search engine

I hope you’ve enjoyed it. In my next series of articles, I’ll show how to use:

- Huggingface Transformers and OpenSeach to create a much better search engine
- Fuzzy query (for those who make typos)

Stay tuned! And follow for more.

_Pracę przygotowano w ramach realizacji projektu pt.: „Hackathon Open Gov Data oraz stworzenie innowacyjnych aplikacji, z wykorzystaniem technologii GPU”, dofinansowanego przez Ministra Edukacji i Nauki ze środków z budżetu państwa
w ramach programu „Studenckie koła naukowe tworzą innowacje”._

![[9d3e0fae3c4bb593e8cad2583206ae8a_MD5.png]]
