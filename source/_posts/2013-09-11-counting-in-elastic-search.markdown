---
layout: post
title: "Counting in ElasticSearch"
date: 2013-09-11 15:48
comments: true
author: mbazydlo
categories: [ElasticSearch, Count, Search, PlainElastic.Net]
---

> Counting is the religion of this generation it is its hope and its salvation.
> Gertrude Stein

In our NeverEnding quest to provide better experience to the users we utilise user behaviour logs to influence future results. One particular case is restaurant popularity, which is indicated by many factors, for example how often it is searched and viewed.

In this blog post we will look into multiple ways of counting documents in Elastic Search which is crucial for this kind of activity. All examples here are provided using Elastic Search HTTP interface and code examples implemented with [PlainElastic.NET](https://github.com/Yegoroff/PlainElastic.Net) are [available here](https://gist.github.com/gondar/6320578)

Before we make a deep dive into Elastic Search Counting options let's define our expectations:
```
So that I can order restaurants by those that are most searched
As a potential diner
I want the most searched statistics from the logs to be part of the search database
```
Okay, that's not exactly how our story was defined but as we don't want to discuss the whole search infrastructure here, let's assume this is sufficient.

Because we are eager engineers, we will quickly build some mock data against which to test our assumptions. Our restaurant name search logs look something like this:
```
{
	"RestaurantId" : 2,
	"RestaurantName" : "Restaurant Brian",
	"DateTime" : "2013-08-16T15:13:47.4833748+01:00"
}
```
So we will populate our mock database with appropriate commands and check that all is in place:
```
curl http://localhost:9200/store/item/ -XPOST -d '{"RestaurantId":2,"RestaurantName":"Restaurant Brian","DateTime":"2013-08-16T15:13:47.4833748+01:00"}'
curl http://localhost:9200/store/item/ -XPOST -d '{"RestaurantId":1,"RestaurantName":"Restaurant Cecil","DateTime":"2013-08-16T15:13:47.4833748+01:00"}'
curl http://localhost:9200/store/item/ -XPOST -d '{"RestaurantId":1,"RestaurantName":"Restaurant Cecil","DateTime":"2013-08-16T15:13:47.4833748+01:00"}'
curl http://localhost:9200/store/item/_search?q=*\&pretty
```
Our expected output is a count of documents for each restaurant. For example:
```
{
	"Restaurant Brian" : 1
	"Restaurant Cecil" : 2
}
```
There are three ways this can be achieved in Elastic Search; using count API (which seems like the most obvious way), a search with type set to count, or using facets to generate counts of all objects grouped by given property. Let's compare them:
### Count API ###
([See documentation here](http://www.elasticsearch.org/guide/reference/api/count/))
```
curl -XPOST http://localhost:9200/store/item/_count -d '{
	"field": {
		"RestaurantName": {
			"query": "Restaurant Cecil",
			"default_operator": "AND"
		}
	}
}'
{
	"count":2,
	"_shards":{"total":5,"successful":5,"failed":0}
}
```
Count is nice little feature which solves our problem. However, if we need count for multiple restaurants we need to execute similar queries multiple times, which may hugely influence both performance of our query and usage of our ElasticSeach cluster.
### Search ###
([See documentation here](http://www.elasticsearch.org/guide/reference/api/search/search-type/))
```
curl -XPOST http://localhost:9200/store/item/_search?search_type=count -d ' {
	"query": {
		"field": {
			"RestaurantName": {
				"query": "Restaurant Cecil",
				"default_operator": "AND"
			}
		}
	}
}'
{
	"took":5,"timed_out":false,"_shards":{"total":5,"successful":5,"failed":0},
	"hits": {
		"total": 2,
		"max_score": 0.0,
		"hits":[]
	}
}
```
Using search type set to count is the same as executing a search request with size set to zero, but it's internally optimised for performance. The nice thing about search is that we can use multi_search interface to execute many count queries at once.

On the other hand, the query still will be executed multiple times, so it is only feasible if we want to get popularity for a small subset of all the restaurants we have.
 
Comparing two previous requests highlights that the query language is slightly different. The DSL for _count API_ is basically the same as for the _search API_, but you are immediately inside the 'query' part. That inconsistency on the ElasticSearch side is only a minor inconvenience.
### Facets ###
([See documentation here](http://www.elasticsearch.org/guide/reference/api/search/facets/))
 
```
curl -XPOST http://localhost:9200/store/item/_search?search_type=count -d '
{
	"query": {
		"match_all": {
			 
		}
	},
	"facets": {
		"ItemsPerCategoryCount": {
			"terms": {
				"field": "RestaurantId",
				"size": 100
			}
		}
	}
}'
{
	"took":1,"timed_out":false,"_shards":{"total":5,"successful":5,"failed":0},"hits":{"total":132,"max_score":0.0,"hits":[]},
	"facets": {
		"ItemsPerCategoryCount": {
			"_type": "terms",
			"missing":0,
			"total":3,
			"other":0,
			"terms": [
				{"term": 2, "count": 1},
				{"term": 1, "count":2}
			]
		}
	}
}
```
 
Facets is a means to obtain grouping by a given field together with count in a group. It is designed to ease creation of filters which are often naturally part of search results interface.

This is nice feature which grabs for us all counts grouped by given field. That's more then we need if we only care for a count of single type, but it's invaluable if you want to have counts for all terms in a field. Also note that we are using search type count again, but facets work equally well for all types of searches including those which actually return results.
 
In the example above we used 'RestaurantId' field instead of restaurant name, as this field is not analysed. If we used restaurant name it would give us facets for each term e.g. [{"term": "Restaurant", "count": 3}, {"term":"Cecil", "count":2},{"term":"Brian", "count":1}], which is not what we exactly want.

### Conclusion ###
It's hard to discuss which one is better. Count API is slightly faster then Search of type count. On the other hand search is more flexible, and its queries are consistent with normal search queries. Facets is a different beast altogether as it always grabs all the results. Still, it's fun that ElasticSearch is elastic in this aspect giving us variety of approaches.
 
We are really curious about your experiences in ElasticSearch. If you have any questions, proposals or comments feel free to [email me](mailto:mbazydlo@opentable.com).
 
### Acknowledgement ###
This blog post benefited thanks to invaluable comments from my team (Andrew Metcalfe, Michael Wallett and Tom Harvey), [PlainElastic.Net](https://github.com/Yegoroff/PlainElastic.Net) author (Yegoroff) and [my brother](https://github.com/pbazydlo).