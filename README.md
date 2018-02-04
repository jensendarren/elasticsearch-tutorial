# Elasticsearch Course

Run Elasticsearch in development by first pulling down the Docker Image:

`docker pull docker.elastic.co/elasticsearch/elasticsearch:6.1.3`

Then fire up Elasticsearch using the following Docker command:

`docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.1.3`

Test that all is well:

`curl http://localhost:9200`

For more details chekc the [Elastic Online Guide](https://www.elastic.co/guide/en/elasticsearch/reference/6.1/docker.html)

## Mapping

We will use the [Movielens dataset](https://grouplens.org/datasets/movielens/). The file to download and use is `ml-latest-small.zip` (size: 1 MB).

To create the mapping for the year type in Elasticsearch run the following curl command:

```
curl -H "Content-Type: application/json" -XPUT http://localhost:9200/movies -d '
{
	"mappings": {
		"movie": {
			"properties": {
				"year": { "type": "date"}
			}
		}
	}
}'
```

Check that the mapping was correctly loaded by getting it back using the following command:

`curl -H "Content-Type: application/json" -XGET http://localhost:9200/movies/_mapping/movie?pretty`

## Insert a single movie

```
curl -H "Content-Type: application/json" -XPUT http://localhost:9200/movies/movie/109487 -d '
{
		"genre": ["Sci-Fi","IMAX"],
		"title": "Interstellar",
		"year": 2014
}'
```

Check to see if the new movie was successfully added by running:

`curl http://localhost:9200/movies/movie/_search?pretty`

## Insert multiple movies

First download the json file containing the movie set

`wget http://media.sundog-soft.com/es/movies.json`

Then run the following command to import as bulk:

`curl -H "Content-Type: application/json" -XPUT http://localhost:9200/_bulk?pretty --data-binary @movies.json`

## Update a movie

Run the following command to update the title of movie id: 109487

```
curl -H "Content-Type: application/json" -XPOST http://localhost:9200/movies/movie/109487/_update -d '
{
	"doc": {
		"title": "Interstella Woo!"
	}
}'
```

## Delete a movie

Run the following to delete the movie id: 109487

`curl -H "Content-Type: application/json" -XDELETE http://localhost:9200/movies/movie/58559/`

## Handling concurrency issues

If a document is open by client/user (A) and is updated by another client/user (B), which would automatically trigger an update to the version of the document, then when client/user(A) updates the document it will fail.

The way, around this issue is to provide a `retry_on_confilict` param stating the number of times to retry, togther with the update statement as follows:

``` 
curl -H "Content-Type: application/json" -XPOST http://localhost:9200/movies/movie/109487/_update?retry_on_conflict=5 -d '
{
  "doc": {
	  "title": "Interstella Foo!"
  }
}'
```

## Using Analyzers 

Firstly lets try some searches. First lets try a match search for 'Star Trek':

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/movies/movie/_search?pretty -d '
{
	"query": {
		"match": { 
			"title": "Star Trek" 
		}
	}
}'
```

You will notice that Star Wars is listed as well as Star Trek. This is becuase the analyser used is the default full text / partial match analyser. Additionally, Star Wars appears above (with more relevance) than Star Trek due to the small number of documents across many shards. A larger corpus of documents will fix/improve the relevancy.

Let's search for a genre of 'sci' as follows:

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/movies/movie/_search?pretty -d '
{
	"query": {
		"match_phrase": { 
			"genre": "sci" 
		}
	}
}'
```

You will notice that the results might not be as expected since the results are *all* films with 'sci' such as 'Sci-Fi'. The *expected* results for requesting a phrase match of 'sci' might be zero documents.

We can fix these issues by specifying analysers to use at the mapping. To do this we need to delete our entire index, update the mapping and reindex the documents again.

To delete the index run this command:

`curl -H "Content-Type: application/json" -XDELETE http://localhost:9200/movies`

Then update the mappings as follows:

```
curl -H "Content-Type: application/json" -XPUT http://localhost:9200/movies -d '
{
	"mappings": {
		"movie": {
			"properties": {
				"id": { "type": "integer"},
				"year": { "type": "date"},
				"genre": { "type": "keyword" },
				"title": { "type": "text", "analyzer": "english"}
			}
		}
	}
}'
```

So to recap what just happend here:

* For string/text fields that should be *exact matches* then define as a `keyword` mapping
* For string/text fields that should be *partial matches based on relevance* then define as a `text` mapping together with an optional anayzer like 'english'.

## Data Modelling in Elasticsearch

Let's start with a parent -> child relationship for franchised movie collections. First up let's create a mapping:

```
curl -H "Content-Type: application/json" -XPUT http://localhost:9200/series -d '
{
	"mappings": {
		"movie": {
			"properties": {
				"film_to_franchise": { "type": "join", "relations": { "franchise" : "film" } }
			}
		}
	}
}'
```

Now we can query for all the movies that belong to a particular franchise 'Star Wars':

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/series/movie/_search?pretty -d '
{
	"query": {
		"has_parent": { 
			"parent_type": "franchise",
			"query": {
				"match": { "title": "Star Wars" }
			} 
		}
	}
}'
```

Or query for the franchise of a particular film by title, like so:

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/series/movie/_search?pretty -d '
{
	"query": {
		"has_child": { 
			"type": "film",
			"query": {
				"match": { "title": "The Force Awakens" }
			} 
		}
	}
}'
```

# Searching in Elasticsearch

## Query Lite / URI Search

Short hand query syntax - useful for debugging and testing out search. Some examples (whcih can also be executed directly in a browser):

`curl http://localhost:9200/movies/movie/_search?q=title:star`
`curl http://localhost:9200/movies/movie/_search?q=title:trek&explain=true`

For more details, refer to the [Elasticsearch URI Search Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-uri-request.html)

**NOTE**: Do not use this approach in Production. It's easy to break, URL may need to be encoded, is a seurity risk, its very fragile.

## Request body JSON search

Things you can do within a query

* Query - need results in order relevance
* Filters - usually for binary type searches. More efficient and faster since they can be cached.

Below is an example using both a Query and a Filter. 

In the query below we have a `bool` query - which is essentially aloowing us to combine conditions together.
Within the `bool` query we have `must` which is essentially the equivalend of an `AND` logical query.
Additionaly within the `bool` query we have a `filter` which further applies a range against the year attribute. 

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/movies/movie/_search?pretty -d '
{
	"query": {
		"bool": { 
			"must": { "term": { "title": "trek" } },
			"filter": { "range": { "year": { "gte": 2010 } }}
		}
	}
}'
```

Types of filters:

* `term` filter by exact values `{ "term": { "year": 2014 } }`
* `terms` match if any exact values in a list match `{"terms": { "genre": ["Sci-Fi", "Adventure"] }}`
* `range` find numbers or dates in a given range `{ "range": { "year": { "gte": 2010 } }`
* `exists` find documents where the field exists `{ "exists": { "field": "tags" } }`
* `missing` find documents where the field is missing `{ "missing": { "field": "tags" } }`
* `bool` combine filters with Boolean logic (must (logical AND), must_not (logical NOT), should (logical OR))

Types of queries:

* `match` searches analysed results such as full-text `{ "match": { "title": "The Force Awakens" } }`
* `multi_match` run the same query on multiple fields `{ "multi_match": { "query": "star", "fields": ["title", "synopsis"] }}`
* `bool` same functionality as with filers but the results are scored by relevance


## Phrase search

Use `match_phrase` and, optionally, `slop`. In the following example the search would find the document containing 'quick brown fox' becuase the match_phrase query is 'quick fox' with a slop of 1.

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/movies/movie/_search?pretty -d '
{
	"query": {
		"match_phrase": { 
			"title": { "query": "quick fox", "slop": 1} 
		}
	}
}'
```

## Proximity query

Let's say you want to find documents and give them a higher relevance when certain words are close together, then set the slop to a high number - like 100, as in this example:

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/movies/movie/_search?pretty -d '
{
	"query": {
		"match_phrase": { 
			"title": { "query": "quick fox", "slop": 100} 
		}
	}
}'
```

## Pagination

Via a url

`http://localhost:9200/movies/movie/_search?size=4&from=2`

Via the query body

```
curl -H "Content-Type: application/json" -XGET http://localhost:9200/movies/movie/_search?pretty -d '
{
	"from": 2,
	"size": 2,
	"query": {
		"match": { "genre": "Sci-Fi" }
	}
}'
```










