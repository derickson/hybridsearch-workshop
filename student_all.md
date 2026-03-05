// Exercise 1: Simple Match Query
GET listings/_search
{
  "query": {
    "match": {
      "description": "waterfront"
    }
  }
}

// Exercise 2: Multi-Match Query
GET listings/_search
{
  "query": {
    "multi_match": {
      "query": "mountain views",
      "fields": ["title", "description"]
    }
  }
}

// Exercise 3: Multi-Match with Field Boosting
GET listings/_search
{
  "query": {
    "multi_match": {
      "query": "mountain views",
      "fields": ["title^3", "description"]
    }
  }
}

// Exercise 4: Fuzziness + Rescore with Geo Distance Decay
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

// Exercise 5: Generate an Embedding
POST _inference/text_embedding/.jina-embeddings-v5-text-small
{
  "input": "cozy family home with a big backyard"
}

// Exercise 6: See a Stored Embedding
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

// Exercise 7: kNN Search with a Raw Vector
// Copy the embedding array from Exercise 5 and paste it in place of <PASTE_VECTOR_HERE>
GET listings_vector/_search
{
  "knn": {
    "field": "description_embedding",
    "query_vector": [<PASTE_VECTOR_HERE>],
    "k": 5,

  },
  "_source": ["title", "description", "price", "city"]
}

// Exercise 8: kNN with query_vector_builder
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

// Exercise 9: Inspect the Semantic Text Mapping
GET listings_semantic/_mapping

// Exercise 10: Semantic Query
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

// Exercise 11: Naive Hybrid with Bool Query
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

// Exercise 12: Hybrid Search with RRF Retriever
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

// Exercise 13: Multilingual Semantic Search
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

// Exercise 14: Call the Rerank API Directly
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

// Exercise 15: Reranker in a Retriever Pipeline
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

// Exercise 16: Simple ES|QL Query
POST _query?format=txt
{
  "query": """
FROM listings_semantic
| WHERE price < 500000
| SORT price ASC
| LIMIT 10
"""
}

// Exercise 17: ES|QL Text Search with MATCH
POST _query?format=txt
{
  "query": """
FROM listings_semantic
| WHERE MATCH(description, "modern kitchen")
| KEEP title, description, price, city
| LIMIT 10
"""
}

// Exercise 18: ES|QL Semantic Search
POST _query?format=txt
{
  "query": """
FROM listings_semantic
| WHERE MATCH(description_semantic, "family friendly neighborhood with good schools")
| KEEP title, description, price, city
| LIMIT 10
"""
}

// Exercise 19: ES|QL Hybrid Search with FORK/FUSE (RRF)
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

// Exercise 20: ES|QL Rerank + LLM Completion
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
