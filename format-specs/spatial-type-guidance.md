# Spatial Type Implementation Review

The GeoParquet specification is an example of a **Spatial Type** implementation
that attempts to combine principles from both well established existing geospatial libraries
and cutting edge data engineering components like [Apache Parquet](https://parquet.apache.org),
[Apache Arrow](https://arrow.apache.org), [Apache Iceberg](https://iceberg.apache.org).
These components are used by major data warehouse vendors like
[BigQuery](https://cloud.google.com/bigquery), [Snowflake](https://snowflake.com),
and [Databricks](https://databricks.com). Broadly, GeoParquet originated from a desire
to provide a cloud-native vector geospatial data format that was capable of performing
at scale.

## Spatial Types?

By **Spatial Type**, we are referring to a library, specification, or file format
that intends to represent points, lines, polygons, or combinations thereof and relate
them to a position on the surface of the Earth[^1]. We will also constrain this
definition to only include attempts to *formalize* the representation of this
data within an existing type system or format that is otherwise *not* a dedicated
Spatial Type. Essentially, if you are storing data in a non-spatial format that you
intend to be put on a map, and you expect somebody else to interpret it in the same
way that you did, that data comes from a Spatial Type implementation.

[^1]: Technically this also applies if you are on the surface of another planet or
celestial body, and also applies if you are above or below the surface. We'll
discuss this later.

## If you don't read anything else

The simplest possible Spatial Type representation that can interoperate with most existing
Spatial Type implementations without loosing information is a
"Geometry" [and/or "Geography"](#geometry-and-geography) type, parameterized with `crs`
([Coordinate Reference System](#coordinate-reference-systems)) as a string, with values
encoded as [Well-known binary](#well-known-binary) (if your format has a native way
to store an array of bytes) or [Well-known text](#well-known-text) (otherwise).

The suggestions in this document are intended to ensure that
new Spatial Type implementations:

- **Capture producer intent**: Users of your Spatial Type should not have to discard
  information when converting spatial data from elsewhere. That means
  if your Spatial Type intends to interoperate with any other Spatial Type, you need
  to provide a mechanism to locate a full Coordinate Reference System definition.
  For example, [GeoPandas](https://geopandas.org) stores a coordinate reference system
  for each column of geometries.
- **Defer to existing standards**: Spatial data can be complicated and there is a lot
  of prior art to draw from. Deferring to existing standards for well-worn topics like
  Coordinate Reference System representation and serialized representation of geometry
  increases the probability that other tools will be able to easily interact with your
  convention. For example, [GeoParquet](https://geoparquet.org) stores the coordinate
  reference system as [PROJJSON](#projjson).
  and stores geometries as [well-known binary](#well-known-binary). It can be tempting
  to improve on existing spatial standards because many of them have limitations; however,
  such an improvement whose definition lives in a non-spatial format is more likely to
  cause confusion and/or misinterpretation than deferring to an existing standard.
- **Leverage the capabilities of the format**: Every format has a unique set of
  features and limitations and there is sometimes a tension between deferring to existing
  standards and ensuring that users of the format can leverage existing tools that work with
  the format. For example, [GeoJSON](https://geojson.org) encodes geometries as JavaScript
  objects instead of a text or binary-based format.

## Prior art

GeoParquet is far from the first attempt to formalize the representation of spatial
data to an existing type system/storage format. A non-exhaustive list of established
Spatial Type implementations (in no particular order) we will refer to in this document
includes:

- [PostGIS](https://postgis.net/) (PostgreSQL)
- [GeoParquet](https://geoparquet.org) (Parquet)
- [GeoPandas](https://geopandas.org) (pandas)
- [Apache Sedona](https://sedona.apache.org) (Apache Spark)
- [Esri Shapefile](https://doc.arcgis.com/en/arcgis-online/reference/shapefiles.htm)
  (DBase).
- [R/sf](https://r-spatial.org/sf) (R)
- [GeoJSON](https://geojson.org/) (JSON)
- [GeoPackage](https://www.geopackage.org/) (SQLite)
- [SpatiaLite](https://www.gaia-gis.it/fossil/libspatialite/index) (SQLite)
- [Apache Calcite Spatial](https://calcite.apache.org/docs/spatial.html) (Apache Calcite)
- [BigQuery Geography](https://cloud.google.com/bigquery/docs/reference/standard-sql/geography_functions) (BigQuery)
- [Snowflake Geospatial](https://docs.snowflake.com/en/sql-reference/data-types-geospatial) (Snowfake)
- [Redshift Geospatial](https://docs.aws.amazon.com/redshift/latest/dg/geospatial-functions.html) (Redshift)
- [FlatGeoBuf](http://flatgeobuf.org/) (Flatbuffers)
- [GeoProtobuf](https://github.com/geo-grpc/api) (gRPC/Protocol buffers)
- [cuspatial](https://docs.rapids.ai/api/cuspatial/stable/) (RAPIDS/cudf)
- [GeoZarr](https://github.com/zarr-developers/geozarr-spec) (Zarr)
- [Climate and Forecasting (CF) Conventions for Geometries](https://cfconventions.org/cf-conventions/cf-conventions.html#geometries)

In addition to established examples of spatial types, several newer libraries are
in the process of adding spatial type support that we will also refer to in this document:

- [GeoArrow](https://geoarrow.org) (Apache Arrow)
- [DuckDB Spatial](https://github.com/duckdb/duckdb_spatial) (DuckDB)
- [Ibis Spatial](https://ibis-project.org/reference/expression-geospatial) (Ibis)
- [Substrait](https://github.com/substrait-io/substrait/blob/67f93b654e4cc60340a214ba90fe15c5e9de941b/extensions/functions_geometry.yaml)

Some examples of projects that are not considered Spatial Type implementations for
the purposes of this document include:

- [Shapely](https://github.com/shapely/shapely)
- [GEOS](https://libgeos.org)
- [JTS](https://locationtech.github.io/jts/)
- [s2geometry](http://s2geometry.io)
- [Boost Geometry](https://www.boost.org/doc/libs/1_86_0/libs/geometry/doc/html/index.html)

These libraries are used by Spatial Type implementations to handle the details of
computational geometry; however, their scope does not include relating those geometries
to the surface of the Earth[^2].

[^2]: Or another celestial body. Again, details forthcoming.

## Components of a data type

Data types describe the user-facing meaning of each element in some collection of
objects (e.g., a column in a table or element in an array). For example,
a [pandas dtype](https://pandas.pydata.org/docs/user_guide/basics.html#basics-dtypes),
a [Parquet LogicalType](https://parquet.apache.org/docs/file-format/types/logicaltypes/),
or an [Arrow Data Type](https://arrow.apache.org/docs/format/Columnar.html#data-types).
Formats use the concept of a data type to separate the arrangement of physical bytes
from the meaning of those bytes. For example, many formats encode dates as the number
of days since January 1, 1970, storing only the type name at the top level and using it
to interpret some sequence of integers.

Most modern type systems allow types to have parameters (e.g., Arrow/Parquet/DuckDB
decimals can be parameterized with a precision and scale at the type level instead
of storing those values alongside each element); however, many older type systems
(e.g., PostgreSQL) only have the opportunity to store a type name or identifier.
This requires that all information required to interpret a value lives with each element
(which, depending on the type, can involve a lot of repetition).

If the format allows, spatial data types should be parameterized with a Coordinate
Reference System to avoid unnecessarily repeating this information. This also allows
better interopearability with other spatial data types that store the Coordinate
Reference System in this way (e.g.,
[GeoPandas](#),
[R/sf](https://r-spatial.github.io/sf/reference/st_crs.html),
[GeoArrow](https://geoarrow.org/extension-types.html#extension-metadata),
[GeoParquet](https://github.com/opengeospatial/geoparquet/blob/v1.1.0/format-specs/geoparquet.md#crs)
).

There are cases where a format does not allow a type to be parameterized (i.e., a name
or identifier is all that is possible). PostgreSQL is an example where this is the case:
as a consumer of PostgreSQL using a non-spatial client, all the information you have
is the name or ID of the type ("geometry" or "geography"). The Spatial Type implementation
for PostgreSQL, PostGIS, uses two mechanisms to make the CRS of a column known:

- It stores a CRS identifier (an integer) with every element that refers to a
  `spatial_ref_sys` table that contains an extended string definition of that CRS.
- It keeps a record of all
  [geometry columns as a VIEW](https://postgis.net/docs/using_postgis_dbmanagement.html#geometry_columns)
  and includes CRS identifier (an integer) that refers to the `spatial_ref_sys` table.

(These mechanisms are also formalized in the
[OGC Simple Feature Access – Part 2: SQL Option](https://www.ogc.org/publications/standard/sfs/))

For clients reading a table directly, it is possible to look up the type-level CRS
by querying the `geometry_columns` and `spatial_ref_sys` tables; however, for clients
executing a complex query, it is necessary to inspect the integer that is sent alongside
each encoded geometry in the result (which is almost always the exact same value for
every element in the result). An important consequence of this workaround is that
once the context of the original database connection is lost, there is no mechanism
to obtain the full CRS definition to pass on to another Spatial Type[^3].

[^3]: To mitigate this problem, PostGIS does use generally consistent identifiers
for common CRS values such that it is usually possible to guess the CRS based on the
integer identifier.

## Coordinate Reference Systems

A Coordinate Reference System (CRS) can be conceptualized as a "unit"
for the combinations of x, y, z, and/or m values of which geometries are
comprised. Just as a value of "5 meters" has physical meaning whereas
the value "5" does not, geometries with a coordinate reference system have
physical meaning.

Just as the value "5 meters" must be transformed to be plotted alongside
the value "5 centimeters", geometries must be transformed to be plotted
alongside each other (this is the act of making a map!). This document will
not go into the multitude of ways that have been devised

- Coordinate reference systems are important to keep alongside
  any geometry in a Spatial Type because after discarding this information
  their physical meaning is lost.
- Coordinate reference systems can be serialized to a string and can be constructed
  from a string.

Producers of Spatial data converting to your Spatial Type implementation
will fall into one of two categories:

- CRS aware producers (i.e., producers like GeoPandas that already
  link to PROJ, a binding to PROJ, or some other coordinate transformation
  library) will be able to choose which format they use to serialize a coordinate
  reference system when converting to your Spatial Type implementation.
- Naive producers (e.g., database drivers or file readers that do not and will
  never link to a third party spatial library of any kind) will have whatever
  sequence of characters that was stored alongside its Spatial Type implementation.

Because some producers can choose, it is best to provide a recommendation. A
slightly opinionated current best option is
[PROJJSON](https://proj.org/en/9.5/specifications/projjson.html), followed by
[WKT2:2019](https://www.ogc.org/publications/standard/wkt-crs/). Both serializations
can be converted losslessly to each other and can represent information that
cannot be represented by earlier versions of WKT. We reccomend PROJJSON because
accessing the contents is possible with any off-the-shelf JSON parser (whereas
to parse WKT2 a specialized parser is required).

Because some producers *cannot* choose, we also reccomend allowing those producers
to pass on whatever sequence of characters they have available because this is
significantly better than dropping the coordinate reference system entirely,
writing those (potentially out-of-spec) characters to your CRS field anyway,
or losing a potential library that otherwise would have supported you. Another
reason to allow this is that there may be future coordinate reference system
representations that solve some of the problems we will list below, and mandating
an existing specification may prevent a producer from writing a better value
later.

Similarly, there are two types of consumers:

- CRS aware producers (i.e., producers like GeoPandas that already
  link to PROJ, a binding to PROJ, or some other coordinate transformation
  library), which usually construct some internal "CRS" object using a string.
- Naive consumers (e.g., database drivers or file readers that do not and will
  never link to a third party spatial library of any kind) whose job it usually
  is to pass on whatever information is in your Spatial Type implementation
  to some other Spatial Type implementation.

### Why can't I just use an "identifier"

Multiple identifiers needed for xy + elevation

### Unsolved Coordinate Reference System issues

- Axis Order:
  https://macwright.com/lonlat/
  https://erouault.blogspot.com/2024/09/those-concepts-in-geospatial-field-that.html
- Representing a coordinate reference system in JSON

## Geometry and Geography

Before getting into the differences between Geometry and Geography, the
important takeaway is that elements of a Geography type cannot be
blindly labeled as Geometry without the potential for severely misinterpreting
those elements[^5]. Your Spatial Type implementation does not have to support
Geographies (many do not), but you should be careful not to silently
interoperate with another Spatial Type implementation that does support them.

[^5]: It is also not a good idea to blindly label a Geometry as a Geography;
however, this comes up much less frequently since Spatial Type implementations
that include Geography as an option are usually well aware of the problems
associated with this and provide explicit mechanisms to perform this conversion.

Some Spatial Type implementations have two data types: Geometry and Geography.
Both Geometry and Geography type implementations interpret coordinate values
(i.e., any combination of x, y, z, and/or m values) identically; however the
two types differ with respect to how adjacent coordinates should be connected
(i.e., edges). Among other things, this affects how area, length, and distance
between elements are defined. A precise definition of the edge interpretation
of all Geometry types of which we are aware can be found in the widely adopted
["OpenGIS Implementation Specification for Geographic information - Simple feature access - Part 1: Common architecture"](https://www.opengeospatial.org/standards/sfa):

> **simple feature** feature with all geometric attributes described piecewise
> by straight line or planar interpolation between sets of points (Section 4.19).

In contrast, Geography data types are newer and have slightly more variation
among implementations. Broadly, all of them define adjacent coordinates in
a line or polygon to be connected as a **geodesic**, or the shortest path
between them over the "surface of the Earth"[^6]. Some selected definitions
include:

> Conversely, the operations on the `GEOGRAPHY` data type treat the coordinates
> inside objects as spherical coordinates on a spheroid.
> ([Amazon Redshift](https://docs.aws.amazon.com/redshift/latest/dg/geospatial-overview.html))

> The `GEOGRAPHY` data type, which models Earth as though it were a perfect sphere...
> Line segments are interpreted as geodesic arcs on the Earth’s surface.
> ([Snowflake](https://docs.snowflake.com/en/sql-reference/data-types-geospatial))

> An edge is a spherical geodesic between two endpoints. (That is, edges are
> the shortest path on the surface of a sphere.)
> ([BigQuery](https://cloud.google.com/bigquery/docs/geospatial-data#coordinate_systems_and_edges))

> `geography` is a spatial data type used to represent a feature in geodetic
> coordinate systems. Geodetic coordinate systems model the earth using an ellipsoid.
> ([PostGIS](https://postgis.net/docs/geography.html))

> This type represents data in a round-earth coordinate system. The SQL Server
> `geography` data type stores ellipsoidal (round-earth) data, such as GPS
> latitude and longitude coordinates.
> ([Microsoft SQL Server](https://learn.microsoft.com/en-us/sql/t-sql/spatial-geography/spatial-types-geography))

The main inconsistency between definitions is whether or not the existing implementations
use spherical formulas to simplify/accellerate calculations or whether strict ellipsoidal
calculations are used. The definitions provided by vendors are not even explicit on this
point in some cases, and we do not reccomend attempting to separate these cases until
the language defining these cases is made explicit in some upstream standard. It is worth
noting that because this distinction only affects the interpretation between explicitly
defined x and y values and because most data sets define x and y values relatively close
together, the ambiguity of this distinction does not frequently cause problems.

Note that some Spatial Type implementations that include geography only support exactly
one coordinate reference system (BigQuery and Snowflake only allow longitude/latitude on
the WGS84 ellipsoid).

An excellent read on why databases implement Geography types (including a discussion
of the tradeoffs associated with using one or the other) can be found
[in the PostGIS documentation](http://postgis.net/workshops/postgis-intro/geography.html).

[^6]: Or Celestial body. Just saying.

### Outliers

Two Spatial Type implementations handle Geography types in unique ways.

- GeoParquet and GeoArrow do not define a separate type for Geography and Geometry.
  Instead, they include an `"edges"` parameter alongside `"crs"` that can take on
  values of `"planar"` (corresponding exactly to Geometry as defined above) and
  `"spherical"` (corresponding exactly to Geography as defined above).
- R's sf package interprets all spatial data in a geographic coordinate system as
  a Geography (which can be controlled by a global flag). For more information,
  see the [Spherical geometry in sf using s2geometry](https://r-spatial.github.io/sf/articles/sf7.html) article in the sf documentation.

## Storing geometries

- serialized type options
- types that leverage the host format
- unsolved issues (empties)
- M values
