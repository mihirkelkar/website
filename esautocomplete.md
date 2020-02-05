---
layout: default
---

# Building an Auto-complete API using ElasticSearch's completion suggester.

Fast auto-complete can be a pretty big factor in content discovery and a strong implicit funnel
to guide users and help them discover your catalog or content. While there are many ways you
can go about implementing auto-complete, I prefer using ES's inbuilt completion suggester
because it was built precisely for this use-case. The completion suggester is a `search-as-you-type`
field that is optimized for speed and returns results almost instantaneously. The added advantage is
the field can also have fuzziness and weights which means that it can adjust for typos and
also re-sort suggestions based on factors that we can decide are more relevant to our use-case.

In-order to demo this, I am going to build an auto-complete field mapping on Elastic Search
and then eventually build a golang web-service to act as an auto-complete API of sorts.

Let's start with spinning up elastic search. To do this I am going to use docker.
```
docker run -p 9200:9200 docker.elastic.co/elasticsearch/elasticsearch:6.0.1
```

We can verify that elastic search is running by running the following command
```
curl -XGET localhost:9200
```
The response should look similar to
```
{
  "name" : "R_iECK_",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "7QktTxXfSdSEhA5rnONpRw",
  "version" : {
    "number" : "6.0.1",
    "build_hash" : "601be4a",
    "build_date" : "2017-12-04T09:29:09.525Z",
    "build_snapshot" : false,
    "lucene_version" : "7.0.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
For this demo, I am going to build auto-complete suggestions on the ml-100k movie dataset. This data set is publicly
available for downloading [here](https://grouplens.org/datasets/movielens/latest/)

Within the dataset, I am going to be using the `u.item` file to seed my elastic search cluster with movie data.

## Defining a Mapping.
A mapping in Elastic Search is a definition of how a document and its fields are stored. Each index has one mapping type which determines how the document will be indexed.To use Completion Suggester, a special type of mapping field type called completion needs to be defined. Let's go ahead and define a new mapping for the `movies` index.
For now, we will only be using the name of the movie and the year in which the movie was released as a part of our index mapping.  

```
PUT localhost:9200/movies/

{
  "mappings": {
    "movies": {
      "properties": {
        "name": {
          "type": "completion"
        },
        "year": {
          "type": "keyword"
        }
      }
    }
  }
}
```

you can check if the index and mapping was created correctly by running the following command.
```
curl -XGET localhost:9200/movies/_mapping
```

Now that we created a new movies index and a mapping for the movies index, lets go ahead and add data to the index. For this, I am just going to loop through the `u.item` file from the dataset using a Python script. From a cursory look at the file, it looks like the second and third column in every row of the file are the name and the date of release.

Each line in the file looks something like
```
1682|Scream of Stone (Schrei aus Stein) (1991)|08-Mar-1996||....
```
Our script to parse this file looks like:
```
>>> import requests
>>> import json
>>> fp = open("u.item", "rb")
>>> movies = [str(movie).split("|") for movie in fp.readlines()]
>>> name_year_list = [(movie[1], movie[2].split("-")[-1]) for movie in movies]
>>> for index, movie in eval(name_year_list):
>>>   try:
>>>	    payload = json.dumps({"name": {"input": [movie[0]]},"year": int(movie[1])})
>>> 	  requests.put("http://localhost:9200/movies/movies/{id}".format(id=index), data=payload, headers={"Content-Type": "application/json"})
>>>   except Exception as err:
>>>     print(err)
```

In this python script, we loop through the file, parse out the name and year of the movie and then index the data in ES.

Let's go ahead and query something to see the result we get back:
```
curl -XGET localthost:9092/movies/movies/_search
{
  "suggest": {
    "movies-suggest": {
      "prefix": "American",
      "completion": {
        "field": "name"
      }
    }
  }
}
```

The result should look something like :
```
{
    "took": 6,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 0,
        "max_score": 0.0,
        "hits": []
    },
    "suggest": {
        "movies-suggest": [
            {
                "text": "American",
                "offset": 0,
                "length": 8,
                "options": [
                    {
                        "text": "American Buffalo (1996)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "h4juC3ABi9zg7g-gV_Gu",
                        "_score": 1.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Buffalo (1996)"
                                ]
                            },
                            "year": 1996
                        }
                    },
                    {
                        "text": "American Dream (1990)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "gYjuC3ABi9zg7g-gXfKV",
                        "_score": 1.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Dream (1990)"
                                ]
                            },
                            "year": 1990
                        }
                    },
                    {
                        "text": "American President, The (1995)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "_ojuC3ABi9zg7g-gR-5j",
                        "_score": 1.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American President, The (1995)"
                                ]
                            },
                            "year": 1995
                        }
                    },
                    {
                        "text": "American Strays (1996)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "oojuC3ABi9zg7g-gWPFL",
                        "_score": 1.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Strays (1996)"
                                ]
                            },
                            "year": 1996
                        }
                    },
                    {
                        "text": "American Werewolf in London, An (1981)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "_ojuC3ABi9zg7g-gQO27",
                        "_score": 1.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Werewolf in London, An (1981)"
                                ]
                            },
                            "year": 1981
                        }
                    }
                ]
            }
        ]
    }
}
```

We can clearly see that these search results have an ordering. We might want to alter the ordering in which they are returned. For our example, lets assume that we are building a movie search database for new movies, so we want to have a bias towards the more recent movies. In order to achieve this, we can define a weight field on each movie. This weight can help us in controlling the ranking of documents when querying.

Let's use the same python script we used previously and change the payload to include a weight field. Lets assign the year of release to be the weight for the document, so the more recent the release, the higher the weight of the document.

```
>>> import requests
>>> import json
>>> fp = open("u.item", "rb")
>>> movies = [str(movie).split("|") for movie in fp.readlines()]
>>> name_year_list = [(movie[1], movie[2].split("-")[-1]) for movie in movies]
>>> for index, movie in enumerate(name_year_list):
>>>   try:
>>>	    payload = json.dumps({"name": {"input": [movie[0]], "weight": int(movie[1])},"year": int(movie[1])})
>>> 	  requests.put("http://localhost:9200/movies/movies/{id}".format(id=index), data=payload, headers={"Content-Type": "application/json"})
>>>   except Exception as err:
>>>     print(err)
```

Let's run the same query that we did a while ago to see if the suggestion results have changed.
Let's go ahead and query something to see the result we get back:
```
curl -XGET localthost:9092/movies/movies/_search
{
  "suggest": {
    "movies-suggest": {
      "prefix": "American",
      "completion": {
        "field": "name"
      }
    }
  }
}
```

The response is ordered by the weights assigned to each movie.
```
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 0,
        "max_score": 0.0,
        "hits": []
    },
    "suggest": {
        "movies-suggest": [
            {
                "text": "American",
                "offset": 0,
                "length": 8,
                "options": [
                    {
                        "text": "American Buffalo (1996)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "1334",
                        "_score": 1996.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Buffalo (1996)"
                                ],
                                "weight": 1996
                            },
                            "year": 1996
                        }
                    },
                    {
                        "text": "American Strays (1996)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "1361",
                        "_score": 1996.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Strays (1996)"
                                ],
                                "weight": 1996
                            },
                            "year": 1996
                        }
                    },
                    {
                        "text": "American President, The (1995)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "691",
                        "_score": 1995.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American President, The (1995)"
                                ],
                                "weight": 1995
                            },
                            "year": 1995
                        }
                    },
                    {
                        "text": "American Dream (1990)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "1584",
                        "_score": 1990.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Dream (1990)"
                                ],
                                "weight": 1990
                            },
                            "year": 1990
                        }
                    },
                    {
                        "text": "American Werewolf in London, An (1981)",
                        "_index": "movies",
                        "_type": "movies",
                        "_id": "435",
                        "_score": 1981.0,
                        "_source": {
                            "name": {
                                "input": [
                                    "American Werewolf in London, An (1981)"
                                ],
                                "weight": 1981
                            },
                            "year": 1981
                        }
                    }
                ]
            }
        ]
    }
}
```

We can control the number of suggestions returned by adding a `size` parameter to the search query too.
```
curl -XGET localthost:9092/movies/movies/_search
{
  "suggest": {
    "movies-suggest": {
      "prefix": "American",
      "completion": {
        "field": "name",
				"size": 1
      }
    }
  }
}
```

We can also add fuzziness in completion suggester. This helps us in providing suggestions even when there is a typo. Let’s try searching for `amrica` the with fuzzy query

```
curl -XGET localthost:9092/movies/movies/_search
{
  "suggest": {
    "movie-suggest-fuzzy": {
      "prefix": "amerca",
      "completion": {
        "field": "name",
        "fuzzy": {
          "fuzziness": 1
        }
      }
    }
  }
}
```

We can use this strong in-built elastic search tool to build s strong auto-complete API for any
kind of data that we have indexed. We can also use context along with completion suggester to filter
auto-complete queries and boost some documents over others dynamically.  

In the next part, I will try to build a simple web-service that accepts plain text inputs and returns auto-complete responses
back to the client.
