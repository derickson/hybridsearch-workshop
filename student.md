# Hybrid Search Workshop — Student Guide

> **Login**: Open Kibana and sign in with username `workshop` / password `elastic123`
>
> **Where to run commands**: Navigate to **Management > Dev Tools** (or use the search bar to find "Dev Tools")
>
> Copy and paste each command block below into the Dev Tools console and click the play button (or press Ctrl+Enter) to run it.

---

## Part 1: Text Search Fundamentals

We'll start with traditional text search using BM25 — the default scoring algorithm in Elasticsearch. These queries run against the `listings` index.

---

### Exercise 1: Simple Match Query

A `match` query analyzes the search text and finds documents containing those terms. BM25 scores higher when terms appear frequently in a document but rarely across the index.

```
GET listings/_search
{
  "query": {
    "match": {
      "description": "waterfront"
    }
  }
}
```

> **Try it:** Change `"waterfront"` to `"modern kitchen"` — notice how more results match because both "modern" and "kitchen" appear in many listings.

---

### Exercise 2: Multi-Match Query

Search across multiple fields at once. Elasticsearch scores each field independently and takes the best score.

```
GET listings/_search
{
  "query": {
    "multi_match": {
      "query": "mountain views",
      "fields": ["title", "description"]
    }
  }
}
```

> **Observe:** Documents matching in the `title` field may score differently than those matching only in `description`.

---

### Exercise 3: Multi-Match with Field Boosting

Use `^` to boost a field's importance. Here, a match in `title` counts 3x more than a match in `description`.

```
GET listings/_search
{
  "query": {
    "multi_match": {
      "query": "mountain views",
      "fields": ["title^3", "description"]
    }
  }
}
```

> **Compare:** Run this side-by-side with Exercise 2. Notice how the ranking changes — listings with "mountain views" in the title get promoted.

---

### Exercise 4: Fuzziness + Rescore with Geo Distance Decay

Use `fuzziness` to handle typos — here we intentionally misspell "swimming pool" as "swimin poul". Then rescore the top results by proximity to Raleigh, NC.

```
GET listings/_search
{
  "query": {
    "match": {
      "description": {
        "query": "swimin poul",
        "fuzziness": "AUTO"
      }
    }
  },
  "rescore": {
    "window_size": 50,
    "query": {
      "rescore_query": {
        "function_score": {
          "functions": [
            {
              "gauss": {
                "location": {
                  "origin": { "lat": 35.7796, "lon": -78.6382 },
                  "scale": "200km",
                  "decay": 0.5
                }
              }
            }
          ],
          "query": { "match_all": {} }
        }
      },
      "query_weight": 0.7,
      "rescore_query_weight": 1.3
    }
  }
}
```

> **What's happening:** `fuzziness: "AUTO"` corrects "swimin" to "swimming" and "poul" to "pool" using edit-distance matching. Then the `gauss` decay function rescores results by proximity to Raleigh — closer listings get boosted, farther ones get demoted.

---

## Part 2: Vector Search

Now we'll move beyond keyword matching to semantic vector search. This finds results based on *meaning*, not just exact words.

---

### Exercise 5: Generate an Embedding

Call the inference API to see what a text embedding looks like. This converts text into a 1024-dimensional vector.

```
POST _inference/text_embedding/.jina-embeddings-v5-text-small
{
  "input": "cozy family home with a big backyard"
}
```

> **Observe:** The response contains an array of 1024 floating-point numbers. This is how the model represents the *meaning* of your text. Similar meanings produce similar vectors.

---

### Exercise 6: See a Stored Embedding

Retrieve a document from `listings_vector` to see the actual embedding stored alongside the listing data.

```
GET listings_vector/_search
{
  "size": 1,
  "_source": ["title", "description_embedding"],
  "query": {
    "match": {
      "title": "waterfront"
    }
  }
}
```

> **Observe:** The `description_embedding` field contains 1024 numbers — this is the vector representation of that listing's description, generated at index time.

---

### Exercise 7: kNN Search with a Raw Vector

First, generate a query vector, then use it to find the most semantically similar listings. Run the inference call from Exercise 5 again, copy the embedding array from the response, and paste it in place of `<PASTE_VECTOR_HERE>` below.

```
GET listings_vector/_search
{
  "knn": {
    "field": "description_embedding",
    "query_vector": [<PASTE_VECTOR_HERE>],
    "k": 5,

  },
  "_source": ["title", "description", "price", "city"]
}
```

> **Note:** This is tedious — copying 1024 numbers! That's why `query_vector_builder` exists (next exercise).

---

### Exercise 8: kNN with query_vector_builder

Let Elasticsearch generate the query vector for you by specifying the model. No need to copy vectors manually.

```
GET listings_vector/_search
{
  "knn": {
    "field": "description_embedding",
    "query_vector_builder": {
      "text_embedding": {
        "model_id": ".jina-embeddings-v5-text-small",
        "model_text": "cozy family home with a big backyard"
      }
    },
    "k": 5,

  },
  "_source": ["title", "description", "price", "city"]
}
```

> **Compare:** The results should be semantically relevant — homes with yards, family-friendly features — even if they don't contain the exact words "cozy", "family", or "backyard".

---

### Exercise 9: Inspect the Semantic Text Mapping

The `listings_semantic` index uses two fields for the description: a `text` field for BM25 and a `semantic_text` field for vector search, connected via `copy_to`.

```
GET listings_semantic/_mapping
```

> **Observe:** The `description` field has type `text` with `copy_to: description_semantic`. The `description_semantic` field has type `semantic_text` with `inference_id: .jina-embeddings-v5-text-small`. When documents are indexed, the description text is automatically copied and embedded — no pipelines needed. This gives us two separate fields: one for keyword search, one for semantic search.

---

### Exercise 10: Semantic Query

The `semantic` query type works with `semantic_text` fields. You just provide natural language — Elasticsearch handles embedding and vector search behind the scenes. Note we target `description_semantic`, not `description`.

```
GET listings_semantic/_search
{
  "query": {
    "semantic": {
      "field": "description_semantic",
      "query": "quiet place in nature away from the city"
    }
  },
  "_source": ["title", "description", "price", "city"]
}
```

> **Compare with Exercise 1:** A keyword search for "quiet place in nature" would miss most results. Semantic search understands the *intent* and finds mountain retreats, lakefront cottages, and rural properties.

---

## Part 3: Hybrid Search

Combine text search and semantic search for the best of both worlds.

---

### Exercise 11: Naive Hybrid with Bool Query

Combine a `match` query (BM25) and a `semantic` query using `bool` + `should`. Both use the same search text so you can compare how each scoring method contributes to the final ranking.

```
GET listings_semantic/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "description": "modern kitchen granite surfaced cooking area"
          }
        },
        {
          "semantic": {
            "field": "description_semantic",
            "query": "modern kitchen granite surfaced cooking area"
          }
        }
      ]
    }
  },
  "_source": ["title", "description", "price", "city"]
}
```

> **The problem:** BM25 scores and vector similarity scores are on completely different scales, so combining them with `bool` can produce unpredictable rankings. The `match` targets the `description` text field (BM25 on individual words like "granite" and "kitchen"), while the `semantic` query targets `description_semantic` (vector search understanding the *concept* of a granite-surfaced cooking area). Because the scores are on different scales, the combined ranking may be skewed. RRF solves this.

---

### Exercise 12: Hybrid Search with RRF Retriever

Reciprocal Rank Fusion (RRF) combines results by *rank position* rather than raw scores, making it scale-independent. Each retriever produces its own ranked list, and RRF merges them.

```
GET listings_semantic/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": {
                "description": "modern kitchen granite surfaced cooking area"
              }
            }
          }
        },
        {
          "standard": {
            "query": {
              "semantic": {
                "field": "description_semantic",
                "query": "modern kitchen granite surfaced cooking area"
              }
            }
          }
        }
      ]
    }
  },
  "_source": ["title", "description", "price", "city"]
}
```

> **Compare with Exercise 11:** RRF combines the two ranked lists by position, not raw score. A document ranked highly by *both* the BM25 and semantic retrievers gets promoted, regardless of score scale differences. The same query text lets you see how each retriever ranks differently.

---

### Exercise 13: Multilingual Semantic Search

The Jina embedding model supports multiple languages. Here we search our English listings using a French query — semantic search matches by meaning, not language.

```
GET listings_semantic/_search
{
  "query": {
    "semantic": {
      "field": "description_semantic",
      "query": "une maison familiale avec un grand jardin et une cheminée"
    }
  },
  "_source": ["title", "description", "price", "city"]
}
```

> **Observe:** The query is in French ("a family home with a large garden and a fireplace") but it matches English listings — the Craftsman bungalow (fenced backyard, fireplace), the lakefront cottage (fireplaces), and similar properties. Semantic search works across languages because both the query and documents are mapped to the same vector space.

---

## Part 4: Reranking

Rerankers are cross-encoder models that look at the query and each document *together* for more accurate relevance scoring. They're more expensive, so we use them to refine a smaller set of candidates.

---

### Exercise 14: Call the Rerank API Directly

Send a query and a list of documents to the reranker to see how it scores them.

```
POST _inference/rerank/.jina-reranker-v2-base-multilingual
{
  "input": [
    "Stunning waterfront property on the Chesapeake Bay with a private dock and panoramic water views.",
    "Sleek open-concept loft in the heart of downtown with floor-to-ceiling windows.",
    "Beautifully restored 1920s Craftsman bungalow with a large fenced backyard and vegetable garden.",
    "Brand new energy-efficient smart home with solar panels and Tesla Powerwall.",
    "Charming lakefront cottage with a private sandy beach and screened-in porch.",
    "Una mansión de tres pisos con una factura de electricidad muy alta.",
    "एक असली जर्जर घर जिसकी छत टपकती है और यह एक अच्छा सर्च रिजल्ट नहीं है।"
  ],
  "query": "eco-friendly home with modern technology"
}
```

> **Observe:** The reranker assigns a relevance score to each document. The smart home with solar panels should score highest. The Spanish passage (a three-story mansion with a very high electricity bill) and Hindi passage (a dilapidated house with a leaking roof) should both score low — they're irrelevant to "eco-friendly home with modern technology" even though they mention houses. The reranker is *multilingual* and judges relevance by meaning, not language or keywords.

---

### Exercise 15: Reranker in a Retriever Pipeline

Wrap the RRF retriever from Exercise 12 inside a `text_similarity_reranker` — first retrieve candidates with hybrid search, then rerank for precision.

```
GET listings_semantic/_search
{
  "retriever": {
    "text_similarity_reranker": {
      "retriever": {
        "rrf": {
          "retrievers": [
            {
              "standard": {
                "query": {
                  "match": {
                    "description": "modern kitchen granite surfaced cooking area"
                  }
                }
              }
            },
            {
              "standard": {
                "query": {
                  "semantic": {
                    "field": "description_semantic",
                    "query": "modern kitchen granite surfaced cooking area"
                  }
                }
              }
            }
          ]
        }
      },
      "field": "description",
      "inference_id": ".jina-reranker-v2-base-multilingual",
      "inference_text": "modern kitchen granite surfaced cooking area"
    }
  },
  "_source": ["title", "description", "price", "city"]
}
```

> **This is the full pipeline:** Text search + Semantic search --> RRF fusion --> Reranker. Each stage refines the results.

---

## Part 5: ES|QL

ES|QL is Elasticsearch's pipe-based query language. It lets you search, filter, and transform data using a familiar SQL-like syntax — including full-text and semantic search.

---

### Exercise 16: Simple ES|QL Query

List affordable listings sorted by price.

```
POST _query?format=txt
{
  "query": """
FROM listings_semantic
| WHERE price < 500000
| SORT price ASC
| LIMIT 10
"""
}
```

> **Note:** ES|QL uses pipes (`|`) to chain operations, similar to Unix commands. Using triple-quoted strings and one pipe per line keeps queries readable.

---

### Exercise 17: ES|QL Text Search with MATCH

Use the `MATCH` function for full-text search within ES|QL.

```
POST _query?format=txt
{
  "query": """
FROM listings_semantic
| WHERE MATCH(description, "modern kitchen")
| KEEP title, description, price, city
| LIMIT 10
"""
}
```

> **Observe:** `MATCH` works like the match query — it uses BM25 scoring under the hood.

---

### Exercise 18: ES|QL Semantic Search

Use `MATCH` on a `semantic_text` field — ES|QL automatically performs a semantic query when the field type is `semantic_text`.

```
POST _query?format=txt
{
  "query": """
FROM listings_semantic
| WHERE MATCH(description_semantic, "family friendly neighborhood with good schools")
| KEEP title, description, price, city
| LIMIT 10
"""
}
```

> **Compare with Exercise 17:** `MATCH` on a `semantic_text` field finds results by meaning. It should surface the Craftsman bungalow ("family-friendly neighborhood"), the suburban townhome ("top-rated school district"), and similar listings — even though they don't contain the exact phrase.

---

### Exercise 19: ES|QL Hybrid Search

Use `FORK` to run BM25 and semantic search as separate branches, then `FUSE` to merge them with Reciprocal Rank Fusion (RRF) — the same technique we used in Exercise 12, but in ES|QL.

```
POST _query?format=txt
{
  "query": """
FROM listings_semantic METADATA _score, _id, _index
| FORK (WHERE MATCH(description, "fireplace") | SORT _score DESC | LIMIT 100)
       (WHERE MATCH(description_semantic, "cozy winter retreat") | SORT _score DESC | LIMIT 100)
| FUSE
| SORT _score DESC
| KEEP title, description, price, city, _score
| LIMIT 10
"""
}
```

> **What's happening:** `FORK` runs two independent retrieval branches — one does BM25 on the `description` text field, the other does semantic search on `description_semantic`. `FUSE` merges them using RRF by default, ranking documents by position across both branches. Documents that rank highly in *both* branches get promoted. This is the ES|QL equivalent of the RRF retriever from Exercise 12.

---

### Exercise 20: ES|QL Rerank + LLM Completion

The full pipeline in a single ES|QL query: semantic search to find candidates, `RERANK` to refine relevance, then `COMPLETION` to call an LLM and generate an answer per result — all in one pipe.

```
POST _query?format=txt
{
  "query": """
FROM listings_semantic METADATA _score, _id, _index
| WHERE MATCH(description_semantic, "good investment property for first-time landlord")
| SORT _score DESC
| LIMIT 5
| RERANK "good investment property for first-time landlord" ON description WITH { "inference_id": ".jina-reranker-v2-base-multilingual" }
| LIMIT 3
| EVAL prompt = CONCAT("Given this property listing in ", city, ", ", state, " priced at $", TO_STRING(price), ": ", description, " — explain in 5 words: why is this a good investment property?")
| COMPLETION answer = prompt WITH { "inference_id": ".anthropic-claude-4.5-haiku-completion" }
| DROP prompt
| KEEP title, price, city, answer
"""
}
```

> **What's happening:** Semantic search finds the top 5 investment-worthy listings. `RERANK` re-scores them using a cross-encoder for better precision. Then `COMPLETION` sends each listing's description to Claude Haiku with the prompt, and the LLM's response appears in the `answer` column. This is RAG built entirely inside ES|QL — no external application code needed.

---

## Congratulations!

You've completed the full search journey:

1. **Text Search** — BM25 keyword matching with boosting and rescoring
2. **Vector Search** — Semantic similarity using dense vector embeddings
3. **Hybrid Search** — Combining text + semantic with bool and RRF
4. **Reranking** — Cross-encoder refinement for precision
5. **ES|QL** — Pipe-based query language with search, reranking, and LLM completion

Each technique builds on the previous one. In production, a typical search pipeline uses **hybrid retrieval (RRF) + reranking** for the best balance of recall and precision.
