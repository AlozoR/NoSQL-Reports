# NoSQL Report: Mongodb


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
But it is not a replication of the data, each district of big cities has its own zip code, this is why we can have same cities name with different localisation and population.


## Import the data in Mongodb

We decided to use the container we used during practical works (named `mongo-db`).

So first, we must import our data in our container (we imported them in the folder named cities inside our container).
```shell
docker cp ./zips.json mongo-db:./cities
```

Then we start and enter our container to write our command.
```shell
docker start mongo-db
docker exec -it mongo-db bash
```

Then we create our database and our collection, using our json.
```shell
mongoimport --db cities --collection cities zips.json
```

Once it is done, we can check if it has been created.
```shell
mongo
```

```javascript
use cities
db.cities.find().count()
```
![](https://i.imgur.com/CGrhGry.png)

And we can compare with the original json file length : 
```bash
wc zips.json
```
![](https://i.imgur.com/ayKHp6Q.png)

The result is indeed our number of lines.

## Query your dataset

### Simple queries

#### Query 1
```javascript
db.cities.distinct("city").length
```
This query returns the amount of different cities in the United states.

![](https://i.imgur.com/y05MMyf.png)



#### Query 2
```javascript
db.cities.find({
  "state": {
    $in: [
      "NC",
      "SC"
    ]
  }
})
```
This one print cities from North and South Carolina.

![](https://i.imgur.com/XyQiitu.png)


#### Query 3
```javascript
db.cities.find({
  "pop":{
    $gt: 15000
  }
}, {
  "city": 1
})
```
Find all queries with 15.000 inhabitants and more.

![](https://i.imgur.com/INDjpLb.png)


#### Query 4
```javascript
db.cities.find({
  "city": "UNION"
})
```
This query print every city named "Union", we can see that there is 15 states containing such city.

![](https://i.imgur.com/sN0UQg5.png)

#### Query 5
```javascript
db.cities.aggregate([{
  $project:{
    _id:0,
    city:1,
    pop:1
  }
},
{
  $sort:{
    "pop":-1
  }
}])
```
This query sorts all the city (or districts) depending on their population, starting from the most populated.

![](https://i.imgur.com/vVNxC0d.png)


### Complex queries

#### Query 1
```javascript
db.cities.ensureIndex({loc: "2dsphere"});

var NewYork = db.cities.findOne({
  city: "NEW YORK"
}, {
  _id: 0
});

db.cities.find({
  "loc": {
    $near: {
      $geometry: {
        coordinates: NewYork.loc
      },
      $maxDistance: 10000
    }
  }
}).count();
```
In this query, we have 3 steps. The first one is to create an index on the loc. Then we create a variable that gets all the data of the New-York city. And finally, we count the number of city in an area of 10 km around New-York.

![](https://i.imgur.com/e5umcOF.png)

#### Query 2
```javascript
db.cities.aggregate([{
  $group:{
    _id:"$city",
    totalPop:{
      $sum:"$pop"
    }
  }
}])
```
This query groups cities depending on their name and add their population (to get the total population of cities with the same name).

![](https://i.imgur.com/e9sF5De.png)

Bustins Island is just a touristic place, nobody lives there.

### Hard queries

#### Query 1
```javascript
var newyork = db.cities.findOne({
  "city": "NEW YORK"
}, {
  loc:1,
  _id:0
}).loc;

var washington = db.cities.findOne({
  "city": "WASHINGTON"
}, {
  loc:1,
  _id:0
}).loc;

var cleveland = db.cities.findOne({
  "city": "CLEVELAND"
}, {
  loc:1,
  _id:0
}).loc;

var triangle = [[newyork, washington, cleveland, newyork]];

db.cities.find({
  loc: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: triangle
      }
    }
  }
}).count();
```
In this query, we get the locations of three cities : New-York, Washington and Cleveland. Then these coordinates are used to create a triangle and we count the number of cities within this triangle.

![](https://i.imgur.com/A5Przw4.png)

#### Query 2
```javascript
db.cities.aggregate([{
  $group: {
    _id: {
      city: "$city",
      state: "$state"
    },
    totalPop: {
      $sum: "$pop"
    }
  }
}, {
  $sort: {
    totalPop: -1
  }
}])
```
In this query, we group cities depending on two data : the city name and the state. That way, we can get the total pop of each city, no matter the different districts and the cities that have the same name, but are located in different states.

![](https://i.imgur.com/vXY3o3w.png)

