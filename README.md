# spatial-search-solr
This project is a perfect example for spatial searchin Solr to locate near-by Starbucks Stores

#### ***Pre-requisite***
1. JDK 1.8.x or above
2. Apache Solr (this solution is build on Apache Solr v8.11.1)
3. Terminal

#### ***Solr Setup***
Download Solr from https://solr.apache.org/downloads.html.  
Extract the download tar file  
Launch Terminal  
Set SOLR_HOME=<path_extracted_directory>  
Start Solr 1 in Port 8983  using the command  
`$SOLR_HOME/bin/solr start -c -p 8983`  


#### ***Store Collection Setup***
1. Upload solr config to Zookeeper  
`/bin/solr zk upconfig -d <path_to_config_set_directory> -n stores -z localhost:9983`  
2. Create Solr Collection using below arguments  
name:stores
configset:stores
numShards:2
`curl --location --request GET 'http://localhost:8983/solr/admin/collections?action=CREATE&autoAddReplicas=false&collection.configName=stores&maxShardsPerNode=1&name=stores&numShards=2&replicationFactor=1&router.name=compositeId&wt=json&nrtReplicas=1'`
3. Upload [starbucks_locations.json](starbucks_locations.json) file to stores collection using the command  
`curl --location --request POST 'http://localhost:8983/solr/stores/update?commit=true' \
--header 'Content-Type: application/json' --data-binary '@/starbucks_locations.json'`  
4. Check to see if indexing is completed by running below command
`curl --location -g --request GET 'http://localhost:8983/solr/stores/select?q=*:*&rows=1'`

#### ***Custom Request Handler to find near by stores***
The config-set stores contains a requestHandler called, store_lookup, which is condifured default values like radius(d=100), sort(nearest to farthest),geo-coordinate field in the index(sfield). The request handler will look like below
```xml
  <requestHandler name="/store_lookup" class="solr.SearchHandler">
       <lst name="invariants">
         <str name="sfield">latlong</str> <!--spatial field in index-->
       </lst>
       <lst name="defaults">
         <str name="pt">41.748489,-88.186111</str><!--lat and long of Naperville Downtown, IL-->
         <str name="q">{!cache=false}{!geofilt sfield=$sfield}</str><!-- geo spatial search query-->
         <str name="sort">geodist() asc</str><!--order by shortest to farthest-->
         <str name="shards.preference">replica.type:PULL</str><!--shard preference to direct 90% queries to PULL replicas when 2+ shards are available, to reduce load on TLOG replicas-->
         <str name="d">160.9</str> <!-- 100 miles-->
         <str name="distanceOffset">0.621371</str><!-- multiplication unit to covert from KM to Mi unit for distance-->
         <str name="fl">storeName,streetAddress,city,state,country,phoneNumber,latlong,distance:concat(mul(def(geodist(),0),$distanceOffset)," Miles"),type:ownershipType</str>
       </lst>
  </requestHandler>
```
#### ***Testing Spatial Search***
1. Use https://www.latlong.net/ to find lat,long of any location, and form the string with the values in the pattern latitude,longitude
(OR) refer the file [us-zip-lat-long.csv](us-zip-lat-long.csv) which contains latitude and longitude for most postal codes in US, from the entry form the string with the values in the pattern latitude,longitude
2. Use the combined value of latitude and longitude as parameter
3. Run below command to find out close by stores for the given lat,long 
   `curl --location -g --request GET 'http://localhost:8983/solr/stores/store_lookup?rows=10&q={!cache=false}{!geofilt sfield=$sfield}&pt=41.794371, -87.972860&echoParams=all'`   


**Play with the index you just created for Spatial Search**

