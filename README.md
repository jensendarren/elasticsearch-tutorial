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
curl -H "Content-Type: application/json" -XPUT http://localhost:9200/movies -d
'{
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
curl -H "Content-Type: application/json" -XPUT http://localhost:9200/movies/movie/109487 -d'
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
curl -H "Content-Type: application/json" -XPOST http://localhost:9200/movies/movie/109487/_update -d'
{
	"doc": {
		"title": "Interstella Woo!"
	}
}
'
```

## Delete a movie

Run the following to delete the movie id: 109487

`curl -H "Content-Type: application/json" -XDELETE http://localhost:9200/movies/movie/58559/`

## Handling concurrency issues

If a document is open by client/user (A) and is updated by another client/user (B), which would automatically trigger an update to the version of the document, then when client/user(A) updates the document it will fail.

The way, around this issue is to provide a `retry_on_confilict` param stating the number of times to retry, togther with the update statement as follows:

``` 
	curl -H "Content-Type: application/json" -XPOST http://localhost:9200/movies/movie/109487/_update?retry_on_conflict=5 -d'
	{
	  "doc": {
		  "title": "Interstella Foo!"
	  }
	}'
```







