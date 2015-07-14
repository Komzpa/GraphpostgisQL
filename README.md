# GraphQL + PostGIS

Experiment with GraphQL, PostGIS, and OSM data

Very much based on Jason Dusek's <a href="https://github.com/solidsnack/GraphpostgresQL">Graphpostgresql</a>

## Install Prerequisites

Install PostGIS and <a href="http://wiki.openstreetmap.org/wiki/Osm2pgsql">Osm2pgsql</a>. On my machine, I had to install osm2pgsql with
a special parameter to include protocol buffers.

Download an OSM PBF file (find some at <a href="https://mapzen.com/data/metro-extracts">Metro Extracts</a>)

## Set up Database

Import an OSM PBF extract file into PostGIS database named "osm":

```bash
initdb pg_data
postgres -D pg_data &
createdb osm
psql -d osm -c "CREATE EXTENSION postgis;"
psql -d osm -c "CREATE EXTENSION postgis_topology;"
osm2pgsql -s -d osm import.osm.pbf
```

You need to have primary keys in your data to use GraphQL. OSM2PGSQL doesn't do this automatically, because
a few items will have repeated invalid, negative osm_ids. Remove them before setting the primary key.

```bash
psql -d osm
DELETE FROM planet_osm_point WHERE osm_id < 0;
ALTER TABLE planet_osm_point ADD PRIMARY KEY (osm_id);
DELETE FROM planet_osm_line WHERE osm_id < 0;
ALTER TABLE planet_osm_line ADD PRIMARY KEY (osm_id);
DELETE FROM planet_osm_polygon WHERE osm_id < 0;
ALTER TABLE planet_osm_polygon ADD PRIMARY KEY (osm_id);
DELETE FROM planet_osm_roads WHERE osm_id < 0;
ALTER TABLE planet_osm_roads ADD PRIMARY KEY (osm_id);
```

Load the graphql schema file:

```bash
psql -d osm -c "\i graphql.sql"
```

## Make queries

Here's a sample query looking up the id of all points:

```sql
SELECT graphql.run($$
  planet_osm_point { id }
$$);
```

Now to show the usefulness of GraphQL:

When the user clicks on a restaurant and I know the OSM id is 560983277, then I can query the database
for the tags which are relevant to a restaurant in OSM data:

```sql
SELECT graphql.run($$
  planet_osm_point("560983277") {
    amenity,
    cuisine,
    internet_access,
    website,
    opening_hours
  }
$$);
```

If your OSM data extract didn't have some of these tags, it might not have the columns and fail. Just
remove them from the query!

Because you enabled PostGIS, you should be able to return the GeoJSON of a field:

SELECT graphql.run($$
  planet_osm_point("560983277") {
    name,
    ST_As_GeoJSON(way)
  }
$$);

## Debug queries

For any query, use to_sql to see the SQL which you would be running:

```sql
SELECT graphql.to_sql($$
  planet_osm_point {
    name,
    amenity
  }
$$);
```

Responds with:

```sql
to_sql | SELECT json_agg("sub/1") AS planet_osm_point
       |   FROM planet_osm_point,
       |        LATERAL (
       |          SELECT planet_osm_point.id
       |        ) AS "sub/1"
 ```

## License

GraphpostgresQL and this repo use the open source PostgresQL license.
