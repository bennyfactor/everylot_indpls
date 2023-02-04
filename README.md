# every lot indianapolis

This library supports a mastodon bot that posts Google Streetview pictures of every property in an SQLite database. 

## What you'll need

Set up will be easier with at least a basic familiarity with the command line. A knowledge of GIS will be helpful.

* mastodonk API stuff, tba
* A Google Streetview API token.
* A SQLite db with a row for every address you're interested in
* A place to run the bot (like a dedicated server or an Amazon AWS EC2 instance)

### mastodonk API

To be figured out

### Streetview key

Visit the [Google Street View Image API](https://developers.google.com/maps/documentation/streetview/) page and click get a key. Make sure that your account and key have both street view and geocoding enabled.

Once you have the key, save it on its own line in your `bots.yaml` file like so:
```yaml
streetview: 123ABC123ABC123ABC123ABC
apps: [etc]
users: [etc]
```

### Address database

First thing, we need a list from the Indianapolis open data portal of all the parcels in the city-county. There is likely a "correct" way to do this through the arcgis api; however, there is no reason to learn all of that when we can just hack together a custom filter and download a geojson file. 

A note: Marion County parcel IDs for regular property are a number; parcels for locations like railroad rights of way, common areas, etc, have an alphabetic prefix. The data portal gives us parcel IDs as a string `(PARCEL_C)` and an integer `(PARCEL_I)`. All those prefixed locations thus have PARCEL_I of zero due to the prefix letters not being converted correctly. Because we don't care about seeing streetview images of locations that might not have even have road access, we can filter by out all those `PARCEL_I = 0` entries. In order to do that, we have to figure out how the data portal filter urls work, which is fairly easy 

```
https://data.indy.gov/datasets/IndyGIS::parcels/explore?filters=eyJQQVJDRUxfSSI6WzAsNTgxNDM3MV19&location=39.779718%2C-86.133000%2C11.97&showTable=true
```

That `filters` querystring just turns out to be base64 encoded data, and after play around with the filter scales for PARCEL_I, we can figure out that the format is `{"PARCEL_I":[<lower_bound>,<upper_bound>]}`. For the bounds I chose 1 and 99999999 as, while the parcels IDs appear to be a 7 digit number, I wanted make sure I didn't miss anything. Thus, after base64encoding `{"PARCEL_I":[1,99999999]}` to `eyJQQVJDRUxfSSI6WzEsOTk5OTk5OTldfQ==` and then percent encoding the equals signs to make eyJQQVJDRUxfSSI6WzAsOTk5OTk5OTldfQ%3D%3D  we can form a url of `https://data.indy.gov/datasets/IndyGIS::parcels/explore?filters=eyJQQVJDRUxfSSI6WzAsOTk5OTk5OTldfQ%3D%3D&location=39.779718%2C-86.133000%2C11.97&showTable=true` and download a nice geoJSON file with all the valid parcels, their info, and their lat/lon coordinates.


Now, you'll need to transform that data into an SQLite database. The database should have a table named `lots` with these fields: `id`, `lat`, `lon`, `tweeted` (the last should be set to zero initially). You must also have some fields that represent the address, like `address`, `city` and `state`. Or, you might have `address_number`, `street_name` and `city`. Optionally, a `floors` field is useful for pointing the Streetview "camera". 

(In the commands below, note that you don't have to type the "$", it's just there to mark the prompt where you enter the command.)

#### Using GDAL/OGR to create the property database

One way to create a the SQLite database is with GDAL command line tools. If you're on OS X and don't have a GIS handy, install [Homebrew](http://brew.sh). Then, paying attention to the fields you noted, do something like this:

````
# this may take a while, you're installing a big software library
$ brew install gdal

# Convert the layer to Google's projection and remove unneeded columns.
# Check the file carefully, its field names will differ from this example.
$ ogr2ogr -f SQLite Parcels_2015_4326.db Parcels_2015.shp -t_srs EPSG:4326 -select taxid,addr,floors

$ ogr2ogr -f SQLite lots.db Parcels_2015_4326.db -nln lots \
    -sql "SELECT taxid AS id, addr AS address, floors, \
        ROUND(X(ST_Centroid(GeomFromWKB(Geometry))), 5) lon, \
        ROUND(Y(ST_Centroid(GeomFromWKB(Geometry))), 5) lat, \
        0 tweeted \
        FROM Parcels_2015_4326 ORDER BY taxid ASC"
````

If you don't want to install GDAL, you can use other command line tools (e.g. `mapshaper`) or a GIS like QGIS or ArcGIS to create a CSV and load it into SQLite:
````
# Convert a CSV (`lots.csv`) to SQLite with two steps
$ sqlite3 -separator , lots.db "import 'stdin' tmp" < lots.csv
$ sqlite lots.db "CREATE TABLE lots AS SELECT taxid AS id, CONVERT(lat, NUMERIC) AS lat,
    CONVERT(lon, NUMERIC) AS lon, address, city, state, 0 AS tweeted FROM tmp;"
````

However you create the SQLite db, add an index (skip this step if have a very small database or like waiting for commands to run):
````
$ sqlite3 lots.db "CREATE INDEX i ON lots (id);"
````

### Test the bot

Install this repository. First download or clone the repo, and open the folder up in terminal. Then run:
````
$ python setup.py install
````

You'll now have a command available called `everylot`. It works like this:
```
$ everylot SCREEN_NAME DATABASE.db --config bots.yaml
```

This will look in `DATABASE.db` for a table called lots, then sort that table by `id` and grab the first untweeted row. The bot then checks where Google thinks this address is, and make sure it's close to the coordinates in the table. Then it wil use the address (or the coordinates, if they seem more reliable) to find a Streetview image, then post a tweet with this image to `SCREEN_NAME`'s timeline. It will need the authorization keys in `bots.yaml` to do all this stuff. After posting, the bots saves the ID of the tweet it made to the `tweeted` field of that row.

`Everylot` will, by default, try to use `address`, `city` and `state` fields from the database to search Google, then post to Twitter just the `address` field.

You can customize this based on the layout of your database and the results you want. `everylot` has two options just for this:
* `--search-format` controls how address will be generated when searching Google
* `--print-format` controls how the address will be printed in the tweet

The format arguments are strings that refer to fields in the database in `{brackets}`. The default `search-format` is `{address}, {city}, {state}`, and the default `print-format` is `{address}`.

Search Google using the `address` field and the knowledge that all our data is in Kalamazoo, Michigan:
````
everylot everylotkalamazoo ./kalamazoo.db --config ./bots.yaml --search-format '{address}, Kalamazoo, MI'
````

Search Google using an address broken-up into several fields:
````
$ everylot everylotwallawalla walla2.db --config bots.yaml \
    --search-format '{address_number} {street_direction} {street_name} {street_suffix}, Walla Walla, WA'
````

You might leave off the city and state when tweeting because that's obvious to your followers:
````
$ everylot everylotwallawalla walla2.db --config bots.yaml \
    --search-format '{address_number} {street_direction} {street_name} {street_suffix}, Walla Walla, WA' \
    --print-format '{address_number} {street_direction} {street_name} {street_suffix}'
````

While testing, it might be helpful to use the `--verbose` and `--dry-run` options. Use the `--id` option to force `everylot` to post a particular property:
````
everylot everylotpoughkeepsie pkpse.db --config bots.json --verbose --dry-run --id 12345
````

### A place for your bot to live

Now, you just need a place for the bot to live. This needs to be a computer that's always connected to the internet, and that you can set up to run tasks for you. You could use a virtual server hosted at a vendor like Amazon AWS, Linode or DigitalOcean, or space on a web server.

Put the `bots.yaml` file and your database in the same folder on your server, then download this repository and install it as above.

Next, you want to set up the bot to tweet regularly. If this is a Linux machine, you can do this with crontab:
```
crontab -e
1,31 * * * * $HOME/.local/bin/everylot screen_name $HOME/path/to/lots.db -s '{address} Anytown USA'
```

(Note that you can omit the `bots.yaml` config file argument if it's located in the home directory.)

### Walkthrough for Baltimore

This walks through the steps of creating an example bot. It uses text-based command line commands, but most of these tasks could be done in programs with graphic interfaces.

First step is to find the data: google "Baltimore open data", search for parcels on [data.baltimorecity.gov](https://data.baltimorecity.gov).

````bash
# Also works to download through the browser and unzip with the GUI
$ curl -G https://data.baltimorecity.gov/api/geospatial/rb22-mgti \
    -d method=export -d format=Shapefile -o baltimore.zip
$ unzip baltimore.zip
Archive:  baltimore.zip
  inflating: geo_export_9f6b494d-b617-4065-a8e7-23adb09350bc.shp  
  inflating: geo_export_9f6b494d-b617-4065-a8e7-23adb09350bc.shx  
  inflating: geo_export_9f6b494d-b617-4065-a8e7-23adb09350bc.dbf  
  inflating: geo_export_9f6b494d-b617-4065-a8e7-23adb09350bc.prj

# Get a simpler name
$ mv geo_export_9f6b494d-b617-4065-a8e7-23adb09350bc.shp baltimore.shp
$ mv geo_export_9f6b494d-b617-4065-a8e7-23adb09350bc.shx baltimore.shx
$ mv geo_export_9f6b494d-b617-4065-a8e7-23adb09350bc.dbf baltimore.dbf

# Find the address and ID fields. It looks like we'll want to use a combination of
# blocknum and parcelnum to get a unique ID for each property
$ ogrinfo baltimore.shp baltimore -so
INFO: Open of 'baltimore.shp'
      using driver 'ESRI Shapefile' successful.
...
parcelnum: String (254.0)
...
blocknum: String (254.0)
fulladdr: String (254.0)
...

# Create an SQLite database, reprojecting the geometries to WGS84. Keep only the desired fields
$ ogr2ogr -f SQLite baltimore_raw.db baltimore.shp baltimore -t_srs EPSG:4326 
    -nln baltimore -select parcelnum,blocknum,fulladdr

# Convert feature centroid to integer latitude, longitude
# Pad the block number and parcel number so sorting works
# Result will have these columns: id, address, lon, lat, tweeted
$ ogr2ogr -f SQLite baltimore.db baltimore_raw.db -nln lots -nlt NONE -dialect sqlite
    -sql "WITH A as (
        SELECT blocknum,
        parcelnum,
        fulladdr AS address,
        ST_Centroid(GeomFromWKB(Geometry)) centroid
        FROM baltimore
        WHERE blocknum IS NOT NULL AND parcelnum IS NOT NULL
    ) SELECT (substr('00000' || blocknum, -5, 5)) || (substr('000000000' || parcelnum, -9, 9)) AS id,
    address,
    ROUND(X(centroid), 5) lon, ROUND(Y(centroid), 5) lat,
    0 tweeted
    FROM A;"

# Add an index
$ sqlite3 baltimore.db "CREATE INDEX i ON lots (id);"

$ everylot everylotbaltimore baltimore.db --search-format "{address}, Baltimore, MD"
````
