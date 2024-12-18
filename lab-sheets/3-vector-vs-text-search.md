# Lab 3: Semantic Search vs Lexical Search

Now we have enriched our movie data with vectors, it's time to compare traditional lexical search with vector search and see what kinds of queries they accept. For this lab, please ensure you have completed the [data ingestion](./1-data-ingestion) and [vector embedding generation](./2-vector-embeddings) modules as a minimum.

## What is Semantic Search?

[Semantic search](https://www.elastic.co/guide/en/elasticsearch/reference/current/semantic-search.html) is an approach that helps you find data based on the contextual meaning. Unlike lexical search, it finds documents based on intent using [vector search to find documents close to the query within the vector space](./2-vector-embeddings.md).

![Star Wars Sample Vector Space](./screenshots/3/lab-3-query-in-vector-space.png)

Vector search is the fundamental algorithm that underpins semantic search, which you will have seen in search engines where you ask questions and they return relevant results whose keywords may not exactly match. 

*kNN, or k-Nearest Neighbour* search, compares the embedded query with the documents in the vector space and returns the documents closest to the query through [Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance):

![kNN search overview](./screenshots/3/lab-3-knn-search-overview.png)

## Steps

*Please ensure you use the index `movies` for the below queries within the **Dev Tools** Console in Kibana. The example queries make use of the facilitators index `movies-carly-richmond`.*

1. Perform a simple [match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) for `Deadpool`, emulating a traditional query. Look at the first few results and try to find the terms that match. 

```json
GET movies-carly-richmond/_search
{
    "query": {
        "match": {
          "text": "Deadpool"
        }
    }
}
```

2. Perform a simple [match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) for `Films to make me laugh`. Look at the first few results and try to find the terms that match. 

```json
GET movies-carly-richmond/_search
{
    "query": {
        "match": {
          "text": "Films to make me laugh"
        }
    }
}
```

3. Generate a vector using model `mxbai-embed-large` and prompt `Films to make me laugh`:

```zsh
curl http://localhost:11434/api/embeddings -d '{
  "model": "mxbai-embed-large",
  "prompt":"Why is the sky blue?"
}' > embedding.json
```

4. Use a [kNN query](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/query-dsl-knn-query.html) to find the 2 closest results to the generated vector for query `Films to make me laugh` evaluating 2 candidates per shard:

```json
GET vector-movies-carly-richmond/_search
{
    "knn": {
      "field": "embedding",
      "k": 2,
      "num_candidates": 2,
      "query_vector": [
    -0.1843881756067276,
    -0.04423162341117859, ..
    ]
  }
}
```

If you're struggling with your generated vector, please use the example in [data/sample-embedding.json](../movie-rag/src/ingestion/data/sample-embedding.json).

5. Change the above query to evaluate *5*, *10* and *20* candidates: 

```json
// 5 candidates
GET vector-movies-carly-richmond/_search
{
    "knn": {
      "field": "embedding",
      "k": 2,
      "num_candidates": 5,
      "query_vector": [
    -0.1843881756067276,
    -0.04423162341117859, ..
    ]
  }
}

// 10 candidates
GET vector-movies-carly-richmond/_search
{
    "knn": {
      "field": "embedding",
      "k": 2,
      "num_candidates": 10,
      "query_vector": [
    -0.1843881756067276,
    -0.04423162341117859, ..
    ]
  }
}

// 20 candidates
GET vector-movies-carly-richmond/_search
{
    "knn": {
      "field": "embedding",
      "k": 2,
      "num_candidates": 20,
      "query_vector": [
    -0.1843881756067276,
    -0.04423162341117859, ..
    ]
  }
}
```

What differences do you notice in terms of execution time and results?

## Expected Result

Comparing the lexical and vector search queries, we see that the results of the vector search return comedy films that may not contain the term *laugh*, however the keyword search gives us results that either contain *laugh* or give us unexpected results swayed by the other terms in the query.

In terms of the number of candidates, we see that the results returned are different as we are evaluating more documents. The more candidates we compare, the longer the query takes to return: 
  
| `num_candidates` | Time Taken (ms) |
| ---------------- | --------------- |
| 2                | 1               |
| 5                | 7               |
| 10               | 1               |
| 20               | 2               |

**[num_candidates] cannot exceed [10000]**: the number of nearest neighbor candidates to consider per shard cannot exceed 10,000, [as per the documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search-api.html)