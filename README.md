# Stratum Building

Import NYC buildings spatial data into a [stratum](https://github.com/mattyschell/stratum)
deployment.

# Dependencies

1. Git Large File Storage https://git-lfs.github.com/
2. Terminal with zip and psql access  

# Import data

Externalize PostgreSQL connection details.

```shell
$ export PGDATABASE=bse
$ export PGUSER=stratum
$ export PGPORT=5433
$ export PGPASSWORD=BeMyDatabae!
$ export PGHOST=aws.dollar.dollar.bill
```

Run the import script to populate either bldg_blue or bldg_green.

```shell
    $ ./import.sh bldg_blue
```


# TMI: Where Did This Data Come From?

You shouldn't read this, it is radically transparent background on how the data
sausage is made.  But you're still reading for some reason.

The New York City Department of Information Technology and Telecommunications
(DOITT) Geographic Information Systems (GIS) outfit maintains buildings 
footprints.  The [metadata is here](https://github.com/CityOfNewYork/nyc-geo-metadata/blob/master/Metadata/Metadata_BuildingFootprints.md).

This data is currently maintained in a versioned [ESRI](https://www.esri.com/en-us/home)
Geodatabase.  The spatial data is stored in ESRI's proprietary SDE.ST_GEOMETRY
format.  Data stored in this format is essentially ransomwared, so the procedure
outlined below is driven by the need to extract the spatial data from the 
database where it is locked up.

Paths and file names below should be changed to protect the innocent.

1. Using ESRI ArcCatalog, export the buildings data to the dreaded, but 
interoperable, [shapefile](https://en.wikipedia.org/wiki/Shapefile) format.

2. Load the dreaded but interoperable shapefile into a scratch PostGIS database
using [shp2pgsql](https://postgis.net/docs/using_postgis_dbmanagement.html#shp2pgsql_usage)

```
shell
shp2pgsql -s 2263 -g shape /d/temp/building.shp buildingtemp > /d/temp/buildingtemp.sql
```

3. Run the sql produced to create a new table named buildingtemp. Column names 
will be lopped off because of the dreaded but interoperable shapefile format. 
We could produce a mapping file to avoid the messy column names hitting the
database.

```
shell
psql /d/temp/buildingtemp.sql
```

4. Insert the scratch data into a more tidy form.  Eliminate buildings that
are under construction, aka "million bins." Remove meaningless vertices and snap
the results to a grid.  The exact parameters below are, and probably will be 
forever, in flux. 

```sql
insert into building (
    bin         
   ,base_bbl         
   ,construction_year  
   ,geom_source       
   ,last_status_type  
   ,doitt_id
   ,height_roof   
   ,feature_code   
   ,ground_elevation
   ,last_modified_date  
   ,mappluto_bbl   
   ,shape           
) select 
     bin
    ,base_bbl::numeric
    ,constructi
    ,geom_sourc
    ,last_statu
    ,doitt_id
    ,height_roo
    ,feature_co
    ,ground_ele
    ,last_edi_1
    ,mappluto_b::numeric
    ,ST_SnapToGrid(ST_SimplifyVW(shape,.1), 0,0, 1,1) 
from buildingtemp
where bin not in (1000000,2000000,3000000,4000000,5000000);
```

5. Verify that all shapes are valid. If not, deal with them.

```sql
select 
    objectid
   ,ST_IsValidReason(shape) 
from 
    building 
where 
    st_isvalid(shape) <> true;
```

6. Dump it

```shell
pg_dump -a -f /d/temp/building.sql -n bldg_blue -O -S stratum -t bldg_blue.building -x
```

7. Zip it

```shell
gzip -k building.sql
```


