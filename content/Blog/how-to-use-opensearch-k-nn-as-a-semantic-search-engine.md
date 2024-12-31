---
title: How to use OpenSearch k-NN as a semantic search engine
date: 2022-08-16T00:00:00+02:00
publish: true
tags:
  - python
---

In my previous [article](https://dev.to/finloop/building-your-first-search-engine-5d1i), I showed how to create a simple search engine using OpenSearch and its fuzzy query. This time I'll show something much more powerful - semantic search engine.

## Getting started

To build our search engine, we'll need:

1. Embeddings of text. Embeddings enable algorithms to do similarity search, a search that can find sentences with words not used in a query, that had a similar meaning to those in a query.
2. Database to store the embeddings.
3. Algorithm to find similar embeddings.

OpenSearch can do both (2) and (3) with its k-NN plugin. You can find how to set up OpenSearch in my previous article [Building your first search engine](https://dev.to/finloop/building-your-first-search-engine-5d1i), I'll assume that you have an OpenSearch instance running.

Embeddings can be generated with Huggingface Transformers or Sentence Transformers. Here's my short article on how to use them: [Sequence Transformers for Polish language](https://dev.to/finloop/sequence-transformers-for-polish-language-224i).

## Field mapping

Before, we didn't need to provide a mapping for fields - OpenSearch did that automatically. For k-NN plugin to work, we need to define at least one field of type `knn_vector` and define its dimensions.

```json
"mappings": {
    "properties": {
      "name":    { "type" : "text" },
      "id":     { "type" : "integer" },
      "description":{ "type" : "text" },
      "embedding": {
        "type": "knn_vector",
        "dimension": 384,
      }
    }
  }
```

I defined a mapping for documents with 4 fields. These 4 fields will represent a recipe: id, name, description, embedding. `embedding` is our vector of 384 numbers (this number depends on your model).

Here's a python script that will create this mapping:

```python
from opensearchpy import OpenSearch, helpers

INDEX_NAME = "recipes"

client = OpenSearch(
    hosts=["https://admin:admin@localhost:9200/"],
    http_compress=True,
    use_ssl=True,  # DONT USE IN PRODUCTION
    verify_certs=False,  # DONT USE IN PRODUCTION
    ssl_assert_hostname=False,
    ssl_show_warn=False,
)

# Create indicies
settings = {
    "settings": {
        "index": {
            "knn": True,
        }
    },
    "mappings": {
        "properties": {
            "name": {"type": "text"},
            "id": {"type": "integer"},
            "description": {"type": "text"},
            "embedding": {
                "type": "knn_vector",
                "dimension": 384,
            },
        }
    },
}

res = client.indices.create(index=INDEX_NAME, body=settings, ignore=[400])
print(res)
```

Please don't run it yet, we still need the part that uploads the data.

## Embedding and uploading recipes

I'll use recipes dataset from kaggle: [Food.com Recipes and Interactions](https://www.kaggle.com/datasets/shuyangli94/food-com-recipes-and-user-interactions). For simplicity, I choose columns: id, name and description. Unfortunately, I cannot embed this whole dataset in one go, as it won't fit into my VRAM, so I'll split it into chunks and upload it chunk by chunk. I'll also use TQDM, for readability purposes - it'll show a progress bar (the embedding can take a while).

Paste this to the end of previous script:

```python
import pandas as pd
import torch
from tqdm import tqdm
from sentence_transformers import SentenceTransformer


FILENAME = "RAW_recipes.zip"
device = "cuda:0" if torch.cuda.is_available() else "cpu"

model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")


# Upload dataset to Openserach
chunksize = 300
lines = 267782  # wc -l RAW_recipes.csv
reader = pd.read_csv(
    FILENAME, chunksize=chunksize, usecols=["name", "id", "description"]
)

with tqdm(total=lines) as pbar:
    for i, chunk in enumerate(reader):
        # Remove NaN
        chunk.fillna("", inplace=True)
        docs = chunk.to_dict(orient="records")

        # Embed description
        with torch.no_grad():
            mean_pooled = model.encode([doc["description"] for doc in docs])
        for doc, vec in zip(docs, mean_pooled):
            doc["embedding"] = vec

        # Upload documents
        helpers.bulk(client, docs, index=INDEX_NAME, raise_on_error=True, refresh=True)

        # Clear CUDA cache
        del mean_pooled
        torch.cuda.empty_cache()

        pbar.update(chunksize)
```

That's the whole script for embedding an uploading. Run it and wait.

### A comment about the code

This part is responsible for chunking, it will load a part of csv file and pass it so that it can be encoded and indexed.

```python
reader = pd.read_csv(
    FILENAME, chunksize=chunksize, usecols=["name", "id", "description"]
)
for i, chunk in enumerate(reader):
    ...
```

By default, TQDM won't show a progress bar, as it doesn't know how long the csv file is. To count how long it is you can use `wc -l RAW_recipes.csv` and the results to `lines` variable.

```python
chunksize = 300
lines = 267782  # wc -l RAW_recipes.csv
reader = pd.read_csv(
    FILENAME, chunksize=chunksize, usecols=["name", "id", "description"]
)
with tqdm(total=lines) as pbar:
    for i, chunk in enumerate(reader):
        ...
        pbar.update(chunksize)
```

There was also a problem with `CUDA Run out of memory` on my 4GB VRAM card. That's why I'm deleting the embedding and emptying CUDA cache after each chunk.

```python
# Clear CUDA cache
del mean_pooled
torch.cuda.empty_cache()
```

## How to use this search engine

To use it we need some text from the user. It'll be embedded and send to OpenSearch, which'll return most similar documents (in our case descriptions of recipes).

OpenSearch query will look like this:

```json
{
    "size": 2,
    "query": {
        "knn": {
            "embedding": {
                "vector": ["your long vector - 384 numbers"],
                "k": 2
            }
        }
    },
    "_source": False,
    "fields": [
        "id",
        "name",
        "description"
    ]
}
```

This `query` is of type `knn`, on an `embedding` field given a `vector` of numbers, it'll find documents with most similar vectors to given one. I disabled the `_source` as it is not needed and selected fields: id, name, description because `embedding` is not readable (and not needed for us).

The whole script for searching the database is here:

```python
from sentence_transformers import SentenceTransformer
from opensearchpy import OpenSearch
import torch
from pprint import pprint

FILENAME = "/home/pk/Projects/blog/simmilarity-search-using-knn/RAW_recipes.zip"
INDEX_NAME = "recipes"
device = "cuda:0" if torch.cuda.is_available() else "cpu"

model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

client = OpenSearch(
    hosts=["https://admin:admin@localhost:9200/"],
    http_compress=True,
    use_ssl=True,
    verify_certs=False,  # DONT USE IN PRODUCTION
    ssl_assert_hostname=False,
    ssl_show_warn=False,
)

text = input("What you're looking for? ")
with torch.no_grad():
    mean_pooled = model.encode(text)

query = {
    "size": 2,
    "query": {"knn": {"embedding": {"vector": mean_pooled, "k": 2}}},
    "_source": False,
    "fields": ["id", "name", "description"],
}

response = client.search(body=query, index=INDEX_NAME)  # the same as before
pprint(response["hits"]["hits"])
```

Here are some of my queries:

```
What you're looking for? Quick dinner
[{'_id': 'MScJpoIBm1k5Fu0QAuOB',
  '_index': 'recipes',
  '_score': 1.0,
  'fields': {'description': ['quick dinner'],
             'id': [448397],
             'name': ['pulled chicken sandwichs with white bbq sauce']}},
 {'_id': 'wyUApoIBm1k5Fu0QXFsX',
  '_index': 'recipes',
  '_score': 0.8753014,
  'fields': {'description': ['quick easy dinner.'],
             'id': [244971],
             'name': ['10 min cheesy gnocchi with seafood sauce']}}]
```

```
What you're looking for? healthy breakfast
[{'_id': 'WigKpoIBm1k5Fu0Qp171',
  '_index': 'recipes',
  '_score': 0.8946137,
  'fields': {'description': ['a healthy breakfast option'],
             'id': [259963],
             'name': ['spinach toast']}},
 {'_id': 'biUApoIBm1k5Fu0Qu3bT',
  '_index': 'recipes',
  '_score': 0.85431683,
  'fields': {'description': ['yummy healthy breakfast'],
             'id': [156942],
             'name': ['apple pancake bake']}}]
```

Go on and play with the search for yourself, I had quite a fun creating it and learned lots of stuff. I hope that you've found this article helpful and learned a thing or two. Leave a follow if you want to read more articles like this.

_Pracę przygotowano w ramach realizacji projektu pt.: „Hackathon Open Gov Data oraz stworzenie innowacyjnych aplikacji, z wykorzystaniem technologii GPU”, dofinansowanego przez Ministra Edukacji i Nauki ze środków z budżetu państwa
w ramach programu „Studenckie koła naukowe tworzą innowacje”._

![[9d3e0fae3c4bb593e8cad2583206ae8a_MD5.png]]
