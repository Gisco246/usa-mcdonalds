# usa-mcdonalds
USA McDonald in county

This map shows the obesity percentages and the amount of McDonald's stores in the United States.
The Main Question of the map is: Does the McDonald's make people obese? Draw your own conclusions with the map (please do not sue me Mcdonald's, i still <3 you).
On loading you can notice the obesity percentages by state. Zoom in once and you can see the amount of Mcdonald's per 100000 km hexagons. Zoom into a state and you can see the actuall stores. Hover on a store for store info, or click a state to see obesity percentages and amout of stores per state 

## How the map was made
The mcDonalds layer poi had no headers and the adress phone and services was concatonaded in a single column. notepad++ had to be used to separate these. 
this csv was imported into qgis using the x and y info as coordinates.

then postgis was used to create 100k km hex and count the amount of stores 
The following query was created to make the hex count
```sql
drop table if exists hexgrid_100k;
create table
	module10.hexgrid_100k
as

select
    /* Function requires geometry field and length of side.
	long diagonal = 2 * side
	side = (5 * 5280) / 2
	side = 2.5 * 5280 */
    CDB_HexagonGrid(st_transform(geom::geometry,102008), 50000)::geometry(polygon, 102008) as geom
from
   module10.cb_2017_us_state_500k;
	 
	 alter table
	module10.hexgrid_100k
add column
	id serial primary key;
	
	create index sidx_hexgrid_100k on module10.hexgrid_100k using gist (geom);
	
	vacuum analyze module10.hexgrid_100k;
	
	/*      */
	 
	drop table if exists module10.mcdonals_by_50k_km_hexagon;
	create table
	module10.mcdonals_by_50k_km_hexagon
as
select
    module10.hexgrid_100k.id as id,
	  count(mcdonalds_usa_102008.*) as count  -- count() function
from
    /* Target layer with enumeration units */
    module10.hexgrid_100k

join
    /* Source layer that will be counted/analyzed */
    mcdonalds_usa_102008
on
    /* Geometric predicate intersects */
    st_intersects(module10.mcdonalds_usa_102008.geom,module10.hexgrid_100k.geom)

group by
    /* The attribute that aggregates the intersecting points, the county geoid */
    module10.hexgrid_100k.id

order by
    count desc;
drop table if exists mcdonals_by_50k_km_hexagon_grid;
/* Convert null values to zero for summary statistics */
create table mcdonals_by_50k_km_hexagon_grid
as
select
	module10.mcdonals_by_50k_km_hexagon.id,
	module10.hexgrid_100k.geom,
	module10.mcdonals_by_50k_km_hexagon.count
from
	module10.mcdonals_by_50k_km_hexagon, module10.hexgrid_100k
where
	module10.mcdonals_by_50k_km_hexagon.id = module10.hexgrid_100k.id;

```
this was created using the 102008 coordinate then converted to 5070

The obesity per state was added and also created a new field and used the count points in polygon to count the amount of stores per state, just for info

## Sources

[Poi Factory, GPS and other interesting topics](http://www.poi-factory.com/node/11154)

[The home of the U.S. Government’s open data](https://catalog.data.gov/dataset/national-obesity-by-state-b181b)