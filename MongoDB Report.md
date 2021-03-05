# Mongo DB 

## 2.1 Simple Queries

#### 2.1.1 
```javascript
db.dblp.find({type: "Book"}, {title: 1 , _id: 0})
```

#### 2.1.2
```javascript
db.dblp.find({"year":{$gte:2011}});

```

#### 2.1.3
```javascript
db.dblp.find({"type":"Book","year":{$gte:2014}})

```

#### 2.1.4
```javascript
db.dblp.find({"publisher":{$exists:true}},{title:1,_id:0});
```

#### 2.1.5
```javascript
db.dblp.find({"authors":{$in:["Jeffrey D. Ullman"]}},{title:1,_id:0});
```

#### 2.1.6
```javascript
db.dblp.find({"authors.0" : "Jeffrey D. Ullman"}, {"title":1,_id:0});
```

#### 2.1.7
```javascript
db.dblp.find({'authors': ['Jeffrey D. Ullman']}, {title: 1, authors: 1, _id: 0})
```

#### 2.1.8
```javascript
db.dblp.find({"title" : {$regex : /database/, $options : "i"},_id:0});
```

## 2.2 Distinct queries

#### 2.2.1
```javascript
db.dblp.distinct("publisher");
```

#### 2.2.2
```javascript
db.dblp.distinct("authors");
```

## 2.3 Aggregates
### 2.3.1 Complex queries

#### 2.3.1.1
```javascript
db.dblp.aggregate([{$match : {"authors": "Jeffrey D. Ullman"}}, 
{$project : {_id: 0, title: 1}}, {$sort : {"pages" : 1}}]);
```

#### 2.3.1.2
```javascript
db.dblp.aggregate([
  {$match:{"authors":"Jeffrey D. Ullman"}},
  {$group:{_id:"$year",count:{$sum:1}}},{$sort:  {"count":-1}}]);
```

#### 2.3.1.3
```javascript
db.dblp.aggregate([
  {$match:{"authors":"Jeffrey D. Ullman"}},
  {$group:{_id:1,count:{$sum:1}}},{$sort:{"count":1}}]);
```

### 2.3.2 Hard queries

#### 2.3.2.1

```javascript
db.dblp.aggregate([
    {"$unwind" : "$authors" },
    {"$group":{"_id":"$authors", "total": {"$sum": 1}}},
    {"$sort" : { "total" : -1} }]);
```

#### 2.3.2.2
```javascript
db.dblp.aggregate([
  {$group:{{"publisher", "year"},count:{$sum:1}}},{$sort:{"count":1}}]);
```

#### 2.3.2.3
```javascript
db.dblp.aggregate([ {$match:{publisher:{$exists:1}} },    {"$group": {"_id": ["$year","$publisher"], "total": {"$sum": 1}}}, {"$sort" : { "total" : -1}} ])
```

#### 2.3.2.4
```javascript
var groupAvgPub = {
$group:{"_id":"$_id.publisher",
"average" : { $avg : "$nb"}
}};
db.dblp.aggregate([
matchpublisher,
groupYearPub,
groupAvgPub,
{$sort:{average:-1}}
]);

```

### 3.1

Pour qu'il ne mette pas à jour que la première ligne qu'il voit.
```javascript
db.dblp.update ( { "authors" : "Jeffrey D. Ullman" },
{$set : { "label" : "Ullman" } },
{"multi" : true });
```

#### 3.2.1
```javascript
db.dblp.update({"type":"Book","title":/database/i},{ $set : {"label": "database"}},{multi:true})
```

#### .3.2.2
```javascript
db.dblp.update( { publisher: /\bACM\b/i }   ,{$unset : { pages : ""} } ,{multi:true}   );
```

#### 3.2.3
On utilise le find avant le update ou le remove pour verifier qu'on touche bien aux donnees souhaitees.
```javascript
db.dblp.find({ authors:{ $eq: [] } },{title:1,authors:1,_id:0});
```
Equivalent à : 
```javascript
db.dblp.find({ "authors" : {$size: 0} }   ,{title:1,authors:1,_id:0});
```

```javascript
db.dblp.remove({"authors": null});
```

#### 3.2.4

```javascript
db.dblp.updateMany({}, [{$addFields: {"pp": {$subtract: ["$pages.end", "$pages.start"]}}}])
```
ou
```javascript
db.dblp.updateMany({}, [{$set: {"pp2": {$subtract: ["$pages.end", "$pages.start"]}}}])
```

### 4.1 Indexing single attributes

#### 4.1.1 For the ”Jeffrey D. Ullman” query, generates the execution plan by adding “.explain()”
```javascript
db.dblp.find({"authors" : "Jeffrey D. Ullman"}, {"title" : 1}).explain("executionStats")
```


### 4.2

```shell
mongoimport --db TP --collection cities cities.json
```

```javascript
db.cities.find({"name": {"$in": ["Paris", "Lyon", "Bordeaux"]}}, {"_id": 0, "name": 1, "location.coordinates": 1})
```

#### 4.2.2
```javascript
var a =db.cities.findOne({"name": "Paris"}, {location:{coordinates:1}, _id:0}).location.coordinates
```
```javascript
db.cities.find( {"location.coordinates":{ $geoWithin :{$centerSphere : [ a,0.0156961]}}} ,{_id:0,name:1}).count()
```
la troisième coordonnée est pour avoir la surface de la terre etant donne que nous sommes dans des coordonnees spheriques.

Ou bien :

```javascript
var Paris = db.dblp.findOne({"name": "Paris"}, {_id: 0})
db.cities.find({"location": {$near: {$geometry: {type: "Point", coordinates: Paris.location.coordinates}, $maxDistance: 100000}}}).count()

```

#### 4.2.3


