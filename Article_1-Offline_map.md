How to build an offline map system for an hybrid app ?
===================

When I was working for my first project at BAM, I had to implement an offline map system. The hybrid application I was working for was developed with [ionic](http://ionicframework.com/) (basically [Cordova](https://cordova.apache.org/) + [Angular.js](https://angularjs.org/) + graphic elements) and [parse.com](http://parse.com) (a [MBaaS](http://en.wikipedia.org/wiki/Mobile_Backend_as_a_service)) and was using [leaflet.js](http://leafletjs.com/) and [its directive in angular](https://github.com/tombatossals/angular-leaflet-directive) for the online map. So we are gonna learn how to build an offline map with these technologies. Warning : you can't see the whole world at any zoom level (the size on the storage space will be enormous), so you have to choose which area you want to cover and the zoom levels you want to offer.

## 1. How do map libraries like leaflet.js work ?

When you are viewing map on a device or on a computer, you are in fact watching several images that are put together side by side. The map is divided in these images that are called tiles (like a rooftop has tiles that join together to cover all the surface).

![An example of a tile](http://a.tile.openstreetmap.org/17/66381/45078.png)

_Example of a tile from [openstreetmap.org](http://a.tile.openstreetmap.org/17/66381/45078.png)_

The mapping library is configured with an URL, for instance :
"http://{s}.{base}.maps.cit.api.here.com/maptile/2.1/maptile/{map_id}/normal.day/{z}/{x}/{y}/256/png8?app_id={app_id}&app_code={app_code}"

This URL points toward a tile rendering server which role is sending tile pictures. You can see in this URL several parameters between {}, they replaced by leaflet when fetching the tiles. Among these parameters : 2 are for the position of the tile (often called x and y), and one is the zoom 'z' (usually between 1 and 20). So when you are scrolling on a map, your library will fetch the tiles for x and y that are near, and when you increase the zoom, the library will fetch the tiles for z+1. The other parameters are less interesting for this tutorial (they can be credentials or parameters for the rendering).

Therefore, to display an offline map. You have to store the tiles on your device and configure the URL used by leaflet as a local path on your filesystem. The first thing is to make available on a server the tiles you want for your offline map. Then your app downloads and imports them.

## 2. Fetching the tiles and packaging them for the offline

Our MbaaS parse.com offers background jobs which are js functions executed on their servers. We are going to use them to fetch all the tiles we need and store them on parse.com.

The first step is to choose a tile server : be careful of the terms of the server, some of them forbid offline usage or bulk downloading of tiles. I used tile.openstreetmap.org for that article, it allows reasonnable use (zoom up to 16), see [their policy page](http://wiki.openstreetmap.org/wiki/Tile_usage_policy).

### 2.1 Crawl the tiles

First, I created a table “MapTile” on parse.com to store the tiles with 5 column : the objectId, the x,y,z parameters, and a file. Then I defined my job to browse the area and download the corresponding tiles :

```coffeescript
Parse.Cloud.job "FetchMapTiles", (request, response) →
	# add some code to get the latitude and the longitude of the boundaries of the map and to
	# set zoomMin and zoomMax, according to the area and the levels of zooms you want
	for zoom in _.range(zoomMin, zoomMax+1) 
			tileNE = _getTileCoordinates(northEastBoundary.latitude, northEastBoundary.longitude, zoom) 
			tileSW = _getTileCoordinates(southWestBoundary.latitude, southWestBoundary.longitude, zoom) 
			for i in _.range(tileSW.xTile, tileNE.xTile + 1)
				for j in _.range(tileNE.yTile, tileSW.yTile + 1) 
					promises.push _getTile i, j, zoom, zone.id
			Parse.Promise.when(promises) 
				.then( 
					(res) -> response.success promises.length+" tiles downloaded" 
					(err) -> response.error "Error"
				)
```

*_getTileCoordinates* is a function to compute the coordinates (x,y) of tiles given a geopoint (latitude, longitude) and a level of zoom (see [some implementations on openstreetmap.org](http://wiki.openstreetmap.org/wiki/Slippy_map_tilenames#Implementations)).

*_getTile* is a function to download and save a tile on parse.com :

```coffeescript
_getTile = (x, y, z, zoneId, app_id, app_code) -> 
	if (x+y)%2 == 0 then serverId = 'a' else serverId = 'b' #load balancing between 2 servers
	url = "http://"+serverId+".tile.openstreetmap.org/"+z+"/"+x+"/"+y+".png" 
	Parse.Cloud.httpRequest 
		url: url 
	.done (httpResponse) -> 
		tileFile = new Parse.File("maptile.png", {base64: httpResponse.buffer.toString('base64', 0, httpResponse.buffer.length)}) 
		Parse.Promise.when(tileFile.save()) 
		.done () -> 
			tile = new Maptile() 
			tile.set "zoom", z 
			tile.set "x", x 
			tile.set "y", y 
			tile.set "tileFile", {"__type": "File", name: tileFile.name(), url: tileFile.url()} 
			Parse.Promise.when(tile.save()) 
```

See [the official Cloud Code guide]( https://www.parse.com/docs/cloud_code_guide) to upload your code and call your job.

This will works fine if you test on small area with a low-medium zoom (less than 14), but if you try on Paris and its suburbs (48.9250,2.3968 to 48.7759,2.214) with zoom 11 to 16, parse.com will return a error, because the job exceeds the number of requests per minute that it is allowed to. My solution is to write a shell script that will call several times the job with a delay of ~1 minute between each call and each job will cover a subpart of the map or/and only for a zoom level (divide and conquer strategy). 

### 2.2 Generate a package

Once you have all the tiles you want on parse.com, the next step is downloading them on your mobile application. If your app downloads them one by one, it will be slow as numerous connexions on the Internet will be necessary (download one file of 1 MB is way faster than downloading 1000 files of 1 KB). A better idea might be to package them into one big JSON, unfortunately it will probably weigths between 40 and 100 MB (for a city like Paris with zoom 11-16) and th us excees the limits at 10 MB per file on parse.com, so 2 solutions :
* split your big JSON file in 10 MB chunks and save these chunks to parse.com, then your app download the the chunks, merge them together and process the result.
* make several JSON valid files (<10 MB) and download and process files one by one.

As your hybrid application runs on mobile devices which don't have a lot of RAM, the second solution is better to avoid crash due to excessive memory consumption, but the tricky part is ensuring that files weights less than 10MB each. As a matter of fact, the size of a tile (from [OSM](http://wiki.openstreetmap.org/wiki/Tile_usage_policy)) is between 7 KB and 40 KB, so grouping up to 200 tiles should comply with this limit.

I created a table on parse.com to store URLs to the files. The table is called MapPackage and contains mainly a column of arrays of URLs. I created a Cloud Code job to generate the MapPackages :
1. get all the tiles (from the table “MapTile”)
2. for each tile, convert the picture file of tile to base64 and save it in the tile object
3. make groups (as JSON objects) with a fixed number of tiles for instance 200 and save the resulting objects on parse.com:
    ```coffeescript
    # mapPackage.mapTile is an array of mapTiles objects
    sizeChunk = 200
    chunkJson = mapPackage.maptile.slice(i*sizeChunk, Math.min(mapPackage.maptile.length, (i+1)*sizeChunk))
    processedPackageStr = JSON.stringify chunkJson 
    bufferChunk = new Buffer processedPackageStr, "utf8" 
    file = new Parse.File(i, base64: bufferChunk.toString('base64'), "application/octet-stream") 
    file.save()
    ```
4. save the URLs of the files in a array of MapPackage

### 3. Importing them in your application

To store data in an hybrid app, there are several solutions :
* the localStorage of the browser, limited to 50 MB, not enough to store our tiles
* an Sqlite database
* the filesystem

As leaflet is configured with an URL to get the tiles it needs, we can't easily use SQLite (we would have to add a server on our hybrid app to serve the tiles). But we can store the tiles on the filesystem and use their path as URL for leaflet.

#### 3.1. Importing

To import the tiles, you  can :
1. get the right row of MapPackage
2. for each file:
⋅⋅* download it
⋅⋅* parse it with [JSON.parse](https://developer.mozilla.org/fr/docs/Web/JavaScript/Reference/Objets_globaux/JSON/parse)
..* save each tile on your storage, be careful on the name you save it, the path must containt the x,y and zoom parameters, for instance : "tile_"+tile.zoom+"_"+tile.x+"_"+tile.y+".png", thus leaflet could easily find a tile given the triplet (x,y,z). For example, the tile for x = 502, y = 1245 and z = 14 is located at "tile_14_502_1245.png".

Sample code to save a tile:
```coffeescript
convertBase64ToBlob = (base64) -> 
    image_data = atob(base64.split(',')[1]) 
    arraybuffer = new ArrayBuffer(image_data.length)
    view = new Uint8Array(arraybuffer)
    for j,i in image_data 
        view[i] = image_data.charCodeAt(i) & 0xff
    new Blob([arraybuffer], {type: 'application/octet-stream'})

saveMapTile = (tile) -> 
    onResolveSuccess = (fileDir) -> fileDir.getFile("tile_"+tile.zoom+"_"+tile.x+"_"+tile.y+".png", {create: true, exclusive: false}, gotFileEntry, fail); 

    gotFileEntry = (fileEntry) -> fileEntry.createWriter(gotFileWriter, fail); 

    gotFileWriter = (writer) -> 
        writer.write(convertBase64ToBlob tile.tileFile.url); 
        deferredFS.resolve '' 

    fail = (error) -> deferredFS.reject '' 

    deferredFS = $q.defer() 
    $window.resolveLocalFileSystemURL(cordova.file.dataDirectory, onResolveSuccess, fail); 
    deferredFS.promise
```

#### 3.2. Configuring leaflet

The last step is to configure your leaflet with the URL pointing to the files on your storage :

```coffeescript
$scope.defaults = tileLayer: cordova.file.dataDirectory+'tile_{z}_{x}_{y}.png'
```

To prevent the user from scrolling or zooming outside the available map (he would see grey tiles), you can set the zoom extrema and the boundaries:

```coffeescript
$scope.defaults.minZoom: 12
$scope.defaults.maxZoom: 16
$scope.maxbounds = leafletBoundsHelpers.createBoundsFromArray [ 
    [ northEastBoundary.latitude, zonenorthEastBoundary.longitude ],
    [ southWestBoundary.latitude, zonesouthWestBoundary.longitude ]
]
```

#### 3.3. Bonus: switching betwen online and offline
If you want your app to act differently whether there is a data connection, for example: fetch the tile from here.com if you are online and use the tiles on your storage if you are offline, you can use the plugin [network-information](https://github.com/apache/cordova-plugin-network-information) to write a method isOnline(). Then you can do:
```coffeescript
if isOnline()
    $scope.defaults = tileLayer: 'http://{s}.{base}.maps.cit.api.here.com/maptile/2.1/maptile/{map_id}/normal.day/{z}/{x}/{y}/256/png8?app_id={app_id}&app_code={app_code}'
else
    $scope.defaults = tileLayer: cordova.file.dataDirectory+'tile_{z}_{x}_{y}.png'
```
