# NoSQL Report: ElasticSearch


## Know your dataset

### Theme

Our Dataset is called "cities" and is 29467 rows long.
Each row corresponds to a city, and each cities are described by five attribute : 
- city: the name of the city
- loc: the location with the coordinates of the city
- pop: the population, number of people living in the city
- state: the abreviation of the city's state
- _id: the city's zip code
```json
{ "city" : "AGAWAM", "loc" : [ -72.622739, 42.070206 ],
  "pop" : 15338, "state" : "MA", "_id" : "01001" }
```

This dataset represent cities from the U.S.A. and is classified as an easy dataset due to the poor number of columns.

### Study of the dataset

An important detail about the data we have, is that some cities appear several times in the dataset:
```json
{ "city" : "SPRINGFIELD", "loc" : [ -72.588735, 42.1029 ],
  "pop" : 2323, "state" : "MA", "_id" : "01103" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.577769, 42.128848 ],
  "pop" : 22115, "state" : "MA", "_id" : "01104" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.578312, 42.099931 ],
  "pop" : 14970, "state" : "MA", "_id" : "01105" }
{ "city" : "LONGMEADOW", "loc" : [ -72.5676, 42.050658 ],
  "pop" : 15688, "state" : "MA", "_id" : "01106" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.606544, 42.117907 ],
  "pop" : 12739, "state" : "MA", "_id" : "01107" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.558432, 42.085314 ],
  "pop" : 25519, "state" : "MA", "_id" : "01108" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.554349, 42.114455 ],
  "pop" : 32635, "state" : "MA", "_id" : "01109" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.527445, 42.092937 ],
  "pop" : 14618, "state" : "MA", "_id" : "01118" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.51211000000001, 42.12473 ],
  "pop" : 13040, "state" : "MA", "_id" : "01119" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.48890299999999, 42.094397 ],
  "pop" : 3272, "state" : "MA", "_id" : "01128" }
{ "city" : "SPRINGFIELD", "loc" : [ -72.487622, 42.122263 ],
  "pop" : 6831, "state" : "MA", "_id" : "01129" }
```
But it is not a replication of the data, each district of big cities has its own zip_code, this is why we can have same cities name with different localisation and population.


## Import the data in ElasticSearch

```python=
def main():
    with open("zips.json", "r") as file:
        with open ("zips_elastic.json", "w") as output:
            count = 1
            for line in file:
                city = line.replace('_', '')
                index = '{"index": {"_index": "cities", "_type": "city", "_id": %i}}' %count
                count += 1
                output.write(index + "\n" + city)


if __name__ == "__main__":
    main()
```

```shell 
docker cp zips_elastic.json elastic-db:/
```

```shell
docker exec -it elastic-db bash
```

```shell
cd /
curl -XPUT localhost:9200/_bulk -H "Content-Type: application/json" --data-binary @zips_elastic.json
```
Now we can open a page and access our localhost:5601

## Query your dataset

### Simple queries

#### Query 1
```javascript
GET cities/_search
{
  "query": {
    "bool": {
      "must": {
        "range": {"pop": {"from": "80000", "to": "85000"}}
      }
    }
  }
}
```
Range cities or districts with population between 80 000 and 85 000 inhabitants.

![](https://i.imgur.com/FN3XV6a.png)

We have 9 results (cf "hits" at the top of screenshot)


#### Query 2
```javascript
GET cities/_search
{
  "query": {
    "bool": {
      "should": [
        {"match": {"state": "NC"}},
        {"match": {"state": "SC"}}
      ]
    }
  }
}
```
Returns cities of states North Carolina and South Carolina.

![](https://i.imgur.com/NTzNzEA.png)


#### Query 3
```javascript
GET cities/_search
{
  "query": {
    "bool": {
      "must": [
        {"range": {"pop": {"gt":15000 }}}]
    }
  }
}
```
Rank cities by population between those wtih more thant 15000 inhabitants.

![](https://i.imgur.com/CCGTLol.png)


#### Query 4
```javascript
GET cities/_search
{
  "query": {
    "bool": {
      "must": {
        "match_phrase": {"city": "CHARLOTTE"}
        },
      "must_not": {
        {"range": {"pop": {"lt":20000 }}}
      }
    }
  }
}
```
This query gives us districs of Charlotte wich contains more than 50000 inhabitants. It will probably returns only districs of the big city of Charlotte in North Carolina and not the smaller cities named Charlotte all around the country.

![](https://i.imgur.com/ypW8KK9.png)

There is 11 districts in Charlotte, NC with more than 20 000 inhabitants.

#### Query 5
```javascript
GET cities/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {"city": "CHARLOTTE"}
      }
    }
  }
}
```
It returns all cities named Charlotte. We can see how many they are in the whole country.

![](https://i.imgur.com/KOqUTZt.png)

We have 38 occurences of Charlotte in our dataset. In the first two results we have a little city of Vermont (3009 habitants) and a little district of Charlotte, NC. 

### Complex queries

#### Query 1
```javascript
GET cities/_search?size=0
{
  "query": {
    "match_phrase": {"city": "NEW YORK"}
  },
  "aggs": {
    "avg_rating": {
      "avg": {
        "field": "pop"
      } 
    }
  }
}
```
Returns average population in each district of New-York City. The result is only 35366.

![](https://i.imgur.com/TslchJ2.png)


#### Query 2 
```javascript
GET cities/_search?size=0
{
  "query": {
    "match_phrase": {"state": "CA"}
  },
  "aggs": {
    "city_count": {
      "cardinality": {
        "field": "city.keywords"
      }
    }
  }
}
```
Number of distinct city in California. The result is 1079 distinct places.

![](https://i.imgur.com/tWQZGFm.png)


### Hard queries

#### Query 1
```javascript
GET cities/_search?size=0
{
  "aggs":{
    "state" : {
      "terms": {"field" :"state.keyword",
      "size": 1000
      },
      "aggs": {
        "average_rating": { "avg": { "field": "pop" }
        },
          "min": { "min": { "field": "pop" }
        },
          "max": { "max": { "field": "pop" }
        }
      }
    }  
  }
}
```
Give the average of population per state and least/highest populous city or district in each state.

![](https://i.imgur.com/meafE9l.png)

Above we can see the results of 3 states: Ohio (OH), Montana (MO), Iowa (IA).

#### Query 2
```javascript
GET cities/_search?size=0
{
  "aggs":{
    "state":{
      "terms":{
        "field":"state.keyword",
        "size":1000
      },
      "aggs":{
        "city":{
          "terms":{
            "field":"city.keyword",
            "size":1000
          },
          "aggs":{
            "totalpop":{
              "sum":{
                "field":"pop"
              }
            }
          }
        }
      }
    }
  }
}
```
This query calculates the total pop of each city (and not district), and make the difference between the cities with the same name, but in different states.

![](https://i.imgur.com/fez03mC.png)
