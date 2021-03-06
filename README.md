# Data Wrangling with OpenStreetMap

* [exercise](https://github.com/LiChangNY/Udacity_MongoDB/tree/master/exercise): scripts for lesson 6 exercises
* [data](https://github.com/LiChangNY/Udacity_MongoDB/tree/master/data): original OpenStreetMap dataset and converted JSON file
* [report](https://github.com/LiChangNY/Udacity_MongoDB/tree/master/index.html): project summary HTML file
* [script](https://github.com/LiChangNY/Udacity_MongoDB/tree/master/script): scripts for data cleaning and MongoDB analysis
* [reference](https://github.com/LiChangNY/Udacity_MongoDB/tree/master/reference): resources used for this project

##Ho Chi Minh City
The reason why I choose Ho Chi Minh City is because it's one of my choices for destination wedding. I want to use OpenStreetMap to explore what hotels are available in the area and, if possible, to compare amenities and ratings of different hotels. Here are the general steps for preparing the dataset:
1. Download data from MapZen:
https://s3.amazonaws.com/metro-extracts.mapzen.com/ho-chi-minh-city_vietnam.osm.bz2 It returns a file of 64 MB (uncompressed).
2. Clean, convert XML data to JSON format using "data_clean.py"
3. Load JSON file to MongoDB for analysis.
mongoimport –d test –c ho ho-chi-minh-city_vietnam.osm.json
 
##Problems encountered
1. Missing addresses
When I was looking at some XML data, I found many records are missing or have incomplete addresses. After I loaded JSON file to MongoDB, it shows that the majorities, 332,214 records, don't have the address information at all.
```
> db.ho.find( { address: {$exists: false}}).count()
332214.
```

2. Street mixed with other address information
I also found that some streets were mixed with district or even city information. For example: 
```
"street" : "Dương B� Trạc, Phường 2, Quận 8, Tp. Hồ Ch� Minh"  <br />
"street" : "Pham Ngu Lao, District 1"  <br />
"street" : "Dương B� Trạc, Phường 2, Quận 8, Tp. Hồ Ch� Minh"  <br />
```

So during the data cleaning process, I decided to keep strings before the first coma even though I lost other information.  Here are the top street names.

```
> db.ho.aggregate( [ {$match: {"address.street": {$exists: true}}}, {$group: { _id: "$address.street", total: { $sum: 1}}}, { $sort: {total: -1}},  {$limit: 5} ])
{ "_id" : "V� Văn Ng�n", "total" : 56 } 
{ "_id" : "Trường Chinh", "total" : 34 }  
{ "_id" : "Nguyễn Thị Minh Khai", "total" : 24 }
{ "_id" : "Hai B� Trưng", "total" : 19 } 
{ "_id" : "Đ�o Duy Anh", "total" : 18 }
```

I've never been to Vietnam so I am interested in knowing what type of street names have been used in Vietnam: I first used MongoDB's MapReduce framework to parse out, aggregate individual words, then list the most used words and finally compared the results with common street names and abbreviations in Vietnam.  Interestingly, very limited street names such as "Đường" (Road, Street, Drive) appeared on the top list; the majority belongs to people's names since many streets, especially in the south, are named to honor communist heroes or folklore characters. 

##MapReduce
```
> var map = function() { if(this.address != null && this.address.street != null) { var words = this.address.street.split(" "); if(words == null){return;} for (var i = 0; i < words.length; i ++ ){ emit(words[i], {count: 1});}}}
> var reduce = function(key, values) {var total = 0; for (var i =0; i < values.length; i ++) { if(values[i] != null) total += values[i].count;} return {count: total};}
> var final = function(key, values) {return key, values;}
> db.ho.mapReduce(map, reduce, {out:"results", finalize: final})  

{
     "result" : "results",
     "timeMillis" : 3883,
     "counts" : {
         "input" : 333369,
         "emit" : 2518,
         "reduce" : 268,
         "output" : 409
     },
     "ok" : 1
}
```

```
> db.results.aggregate( {$sort: {"value.count": -1}}, {$limit:10})
{ "_id" : "Nguyễn", "value" : { "count" : 165 } }
{ "_id" : "Văn", "value" : { "count" : 129 } }
{ "_id" : "V�", "value" : { "count" : 71 } }
{ "_id" : "Ng�n", "value" : { "count" : 56 } }
{ "_id" : "L�", "value" : { "count" : 51 } }
{ "_id" : "Đường", "value" : { "count" : 41 } }
{ "_id" : "Thị", "value" : { "count" : 40 } }
{ "_id" : "Trường", "value" : { "count" : 38 } }
{ "_id" : "Chinh", "value" : { "count" : 36 } }
{ "_id" : "Minh", "value" : { "count" : 35 } }
```
3. Missing/inaccurate postcodes
I also took a look at the postcodes that should have at least 5 digits starting with either 70 or 76. However, as you can see from below, many have strings instead of numbers.
```
> db.ho.aggregate( [ {$match: {"address.postcode": {$exists: true}}}, {$group: { _id: "$address.postcode", total: { $sum: 1}}}, { $sort: {total: -1}},  {$limit: 5} ])
{ "_id" : "Quan Thu Duc", "total" : 84 }
{ "_id" : "70000", "total" : 13 }
{ "_id" : "7000", "total" : 4 }
{ "_id" : "08", "total" : 1 }
{ "_id" : "District 3", "total" : 1 }
```
Therefore I cleaned up inaccurate postcodes before exporting to JSON file and it turns out only 21 records have valid postcodes.
```
> db.ho.aggregate( [ {$match: {"address.postcode": {$exists: true}}}, {$group: { _id: "$address.postcode", total: { $sum: 1}}}, { $sort: {total: -1}},  {$limit: 10} ])
{ "_id" : "70000", "total" : 13 }
{ "_id" : "7000", "total" : 4 }
{ "_id" : "70000-989", "total" : 1 }
{ "_id" : "709999", "total" : 1 }
{ "_id" : "70000-1648", "total" : 1 }
{ "_id" : "70150", "total" : 1 }
```

##Overview of the data
The imported JSON file has about 91 MB and a total of 333,369 records comprised of 299,969 nodes and 33,400 ways.
```
> db.ho.stats()
{
     "ns" : "test.ho",
     "count" : 333369,
     "size" : 95796592,
     "avgObjSize" : 287,
     "storageSize" : 123936768,
     "numExtents" : 11,
     "nindexes" : 1,
     "lastExtentSize" : 37625856,
     "paddingFactor" : 1,
     "systemFlags" : 1,
     "userFlags" : 1,
     "totalIndexSize" : 10833200,
     "indexSizes" : {
         "_id_" : 10833200
     },
     "ok" : 1
}

> db.ho.find( {type: 'node'}).count()
299969
> db.ho.find( {type: 'way'}).count()
33400
```

A total of 329 users have been contributing to this area.
```
> db.ho.distinct('created.user').length
329
```
 
Top 10 users contributed about 90% of the total records. Kudos to those diligent folks!
```
> db.ho.aggregate( { $group: { _id: "$created.user", total: { $sum: 1}}}, { $sort: {total: -1}},  { $limit: 10} )
{ "_id" : "Tuan Ifan Vpy", "total" : 89563 }
{ "_id" : "Ivan Garcia", "total" : 84724 }
{ "_id" : "Dymo12", "total" : 42389 }
{ "_id" : "QuangDBui@TMA", "total" : 32285 }
{ "_id" : "528491", "total" : 16985 }
{ "_id" : "guenter", "total" : 10841 }
{ "_id" : "tandung", "total" : 9709 }
{ "_id" : "ZeitgeistGroup", "total" : 5772 }
{ "_id" : "ninomax", "total" : 5396 }
{ "_id" : "jaapsen", "total" : 3842 }
```

Interestingly, the top one amenity is fuel station. It actually makes sense given that Ho Chi Minh is a city on the motorbikes and scooters!
```
> db.ho.aggregate( [ {$match: {amenity: {$exists: true}}}, {$group: { _id: "$amenity", "total": { $sum: 1}}}, { $sort: { total: -1}}, { $limit: 10}])

{ "_id" : "fuel", "total" : 483 }
{ "_id" : "atm", "total" : 268 }
{ "_id" : "school", "total" : 253 }
{ "_id" : "restaurant", "total" : 218 }
{ "_id" : "cafe", "total" : 216 }
{ "_id" : "place_of_worship", "total" : 189 }
{ "_id" : "bank", "total" : 149 }
{ "_id" : "fast_food", "total" : 131 }
{ "_id" : "parking", "total" : 112 }
{ "_id" : "hospital", "total" : 71 }
```

##Other ideas about the datasets
For my purpose, I want to look at what hotels are available in Ho Chi Minh City. There are 142 records matching exiting tags for hotel.
```
> db.ho.find( {$or: [ { tourism: 'hotel'}, {building: 'hotel'}]}).count()
142
```
 
Of 7 hotels that have ratings, 4 hotels are rated five-star and 3 are three-star.
```
> db.ho.aggregate({$match: { $or: [{tourism: 'hotel'}, {building: 'hotel'}]}},{$group: { _id: "$stars", total: {$sum: 1}}},{ $sort: {_id: -1}})

{ "_id" : "5", "total" : 4 }
{ "_id" : "3", "total" : 3 }
{ "_id" : null, "total" : 135 }
```

Here are the five-star hotels:
```
> db.ho.aggregate({$match: {stars: "5"}},{$group: { _id: "$name", total: {$sum: 1}}},{ $sort: {_id: -1}})
{ "_id" : "Park Hyatt", "total" : 1 }
{ "_id" : "Nikko Saigon", "total" : 1 }
{ "_id" : "M�venpick Hotel Saigon", "total" : 1 }
{ "_id" : "Majestic Hotel", "total" : 1 }
```

Only one hotel has information on number of rooms.
```
> db.ho.aggregate({$match: { $or: [{tourism: 'hotel'}, {building: 'hotel'}]}},{$group: { _id: "$rooms", total: {$sum: 1}}},{ $sort: {_id: -1}})
{ "_id" : "247", "total" : 1 }
{ "_id" : null, "total" : 141 }
```

As discussed in the Problems Encountered section, missing information on addresses (and hotel information discussed in this section) is a big problem for this OSM dataset. To tackle such large number of missing data, I don�t think it is realistic to rely on individual users to insert information. Instead, leveraging an online survey may be a feasible solution and starting point. This survey can include basic fields such as street name, house number, and key information pursuant to specific business such as the rating for hotel, etc. However, distributing the survey to individual business owners/points of contact may be challenging. One idea is to reach out to the local associations, for example, the Vietnam National Administration of Tourism located in Ho Chi Minh City, because these associations typically maintain good contacts with local businesses and can help administrate the process, increase survey response rate, and monitor the quality of data collected as long as they can be convinced that populating information on OpenStreetMap can help the public retrieve information and promote local tourism. After collecting survey data, we can batch upload the information to OpenStreetMap.
 
 
##Conclusion
After using Python and MongoDB to examine the OpenStreetMap data, I realized that there is still much work needed to fill in the address information. Exploring a dataset in non-English language is certainly challenging for me. But throughout the research, I certainly learned quite a bit about Vietnamese culture like street naming convention. Also, I was able to find a few good leads as potential wedding venue. However, if the dataset can have more information on the ratings and address, I am sure it can expand my search results and help me find the ideal hotel.
