# Elasticsearch Course

Run Elasticsearch in development by first pulling down the Docker Image:

`docker pull docker.elastic.co/elasticsearch/elasticsearch:6.1.3`

Then fire up Elasticsearch using the following Docker command:

`docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.1.3`

Test that all is well:

`http://localhost:9200`

For more details chekc the [Elastic Online Gude](https://www.elastic.co/guide/en/elasticsearch/reference/6.1/docker.html)

## Mapping

We will use the [Movielens dataset](https://grouplens.org/datasets/movielens/). The file to download and use is `ml-latest-small.zip` (size: 1 MB).

To create the mapping for the year type in Elasticsearch run the following curl command:

`curl -H "Content-Type: application/json" -XPUT http://localhost:9200/movies -d'
{
	"mappings": {
		"movie": {
			"properties": {
				"year": { "type": "date"}
			}
		}
	}
}
`