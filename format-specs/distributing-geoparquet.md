# Best Practices for Distributing GeoParquet

This guide aims to encapsulate a number of best practices that the community has
started to align on for making 'good' GeoParquet files, especially for distribution
of data. Parquet gives users lots of different options, and the defaults of various
libraries are different and usually not optimized for geospatial data.

## tl;dr Recommendations

Later sections will go deep into the reasoning and nuances behind these options, but if you're
just looking to be sure you get the basics right then this section may be sufficient.
And if you're building a tool or library then consider these as good defaults.

 * Use zstd for compression.
 * Be sure to include the [bbox covering](https://github.com/opengeospatial/geoparquet/blob/v1.1.0/format-specs/geoparquet.md#bbox-covering-encoding), and use GeoParquet version 1.1.
 * Spatially order the data within the file.
 * Set the maximum row group size between 50,000 and 150,000 per row.
 * If the data is larger than ~2 gigabytes consider spatially partitioning the files.
 * Use [STAC Metadata](https://stacspec.org/) metadata to describe the data.


### Compression

Parquet has built in compression, enabling users to directly use files that are similar in size to the zipped versions
of other formats. The cool thing is you can easily change the compression algorithm, and new ones continue to be added.
The default for most Parquet libraries is `snappy`, which excels at speed and gets good compression. More recently the
`zstd` library has been added to the Parquet ecosystem, and it achieves a better compression ration with similar speeds
to snappy. At this point most all Parquet libraries support `zstd`, and since better compression makes for faster downloads
and streaming it's the current recommendation.

One interesting option is `brotli`, which often compresses 20-30% smaller than `zstd` but is slower. It is reaching wider
adoption, so if you want to have the smallest possible files then it's worth considering. But for many access patterns
it will be slower overall than `zstd`. There is an option to do 'uncompressed' files, but this is not recommended.

### bbox covering

GeoParquet 1.1 included a couple new features that help with spatial indexing and querying. The easiest one to use is the
bbox covering, which adds a column called `bbox` that contains the bounding box of each geometry as a native Parquet 'struct'
of four values. This enables Parquet readers to quickly filter rows based on the bounding box, and thus greatly increasing
the performance of spatial queries. The bbox column by itself is not sufficient to speed up spatial queries - for that
you'll need to be sure to follow the next two recommendations. But be sure to include it. It is possible for some tools to
make use of the bbox column even if the GeoParquet version is not 1.1, but it's best to actually distribute the files with
GeoParquet 1.1 to ensure all tools know they can use the bbox column.

The other new feature is the Native geometry encodings, based on GeoArrow. Using these will enable the same types of speed
ups as the bbox covering, but will store the data more efficiently. Parquet readers will be able to use the min/max statistics
directly from the geometry column, instead of needing the bbox column. For points this will be a big win, since the bbox
for a point adds a good bit of overhead. But we do not yet recommend using the native encodings, since the tool support
isn't yet extensive enough to be sure that most clients will understand them. But as the ecosystem matures this will
be a great option.

### Spatial Ordering

It is essential to make sure that the data is spatially ordered in some way within the file, in order for the bbox column
to be used effectively. If the GeoParquet data was converted from a GIS format like GeoPackage or Shapefile then often
it will already by spatially ordered. One way to check this is to open the file in a GIS tool and see if the data loads
all the spatial data for an area in chunks, or if data for the whole are appears and continues to load everywhere.

<img alt="non-indexed load" height="300" src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*yugDd1ZjLG4lEwUZucRdmA.gif"> vs <img alt="indexed load" height="300" src="https://miro.medium.com/v2/resize:fit:1400/format:webp/1*-4wyoKgwFXpUnkLeziv5KA.gif"/>

GeoParquet itself does not have a specific spatial index like other formats (R-tree in GeoPackage, Packed Hilbert R-tree in
FlatGeobuf). Instead data can be indexed in any way, and then Parquet's Row Group statistics will be used to speed up spatial
queries (when using bbox covering or native arrow types). Most tools that provide GeoParquet writers have some ability to apply a spatial index, the examples below will show how to do this for a few common tools.

### Row Group Size

A row group in Parquet is 'a logical horizontal partitioning of the data into rows', and there's some good explanation
in [this article](https://medium.com/data-engineering-with-dremio/all-about-parquet-part-03-parquet-file-structure-pages-row-groups-and-columns-d7c7e54a8311). It ends up being important to
get this right, since it will impact the performance of spatial queries. If the row group size is too big then the GeoParquet
reader will not be able to 'skip' over large chunks of data, and if it's too small then the file metadata can get quite large,
which can really slow things down if there are a lot of files.

Unfortunately there's no single 'best' size for row groups, and it will depend on the size of the data and the access patterns.
And the community is still learning what works best, so there's no solid recommendations at this point - hopefully we'll learn
more and update this section in the future. But right now most of the larger global datasets are being distributed with
row group sizes of 50,000 to 200,000 rows, so that's what we recommend as a starting point.

Most geospatial tools give you the ability to set the maximum number of rows per row group, but some tools may let you set
the byte size for the row group. The core thing that really matters is the byte size for the row group, as that will be
the amount of data that needs to be read (and moved over the network in cloud-native geo access patterns). So if your data
rows are large then you'll want to set a smaller row group size, and if your rows are small it could make sense to go to the
larger end of the spectrum. If you can set the byte size for row groups a common recommendation is to aim for 128mb - 256mb
per row group.

### Spatial Partitioning

One of the coolest features of Parquet is the ability to partition a large dataset into multiple files, as most every reader
can be pointed at a folder of files and it will read them as a single dataset. The reader will use the row group statistics
to quickly figure out if a given file needs to be read, and multiple files can be read in parallel. So with spatial data,
where most every query contains a spatial filter, partioning the data spatially can greatly accelerate the performance.

Similar to the row group size, the community is still figuring out the best way to spatially partition the data, and the
overall query performance will depend on both row group size and the size of the partitioned files, along with the nature of
the data. Hopefully someone will do a set of robust testing to help inform more definitive recommendations.

For now the recommendation is to spatially partition your data 'in some way', at least if the dataset is larger than a couple
gigabytes. If it's smaller than that then the additional overhead of splitting it up is likely not worth it. There was some
[great discussion](https://github.com/opengeospatial/geoparquet/discussions/251) on the topic, and an nice
[blog post](https://dewey.dunnington.ca/post/2024/partitioning-strategies-for-bigger-than-memory-spatial-data/) with some
further experimentation. The leading approach at the moment is to use a K-dimensional tree (KD-tree), which will enable
nice balancing of the file sizes, but sorts based on S2, GeoHash or R-tree can all work. And partitioning [based on admin
boundaries](https://medium.com/radiant-earth-insights/the-admin-partitioned-geoparquet-distribution-59f0ca1c6d96) is another
approach that works, used in the [Google-Microsoft-OSM Buildings - combined by VIDA](https://source.coop/repositories/vida/google-microsoft-osm-open-buildings/description)
dataset.

### Use STAC metadata

If you're publishing GeoParquet files publicly or internally then it's a good idea to describe the data in a standard way.
The [STAC specification](https://stacspec.org/en)'s [Collection](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md#provider-object%20PROVIDERS%20=%20[) level metadata to describe what's in it. For single
GeoParquet files this should be very simple, just create a collection.json file in the same folder as the GeoParquet file and
use `application/vnd.apache.parquet` as the media type. If the GeoParquet is partitioned then you can create individual
STAC Items linked to from the collection, with each item describing the bounding box of the data in the file.

## Exemplar Datasets

At the time of writing there are a couple datasets that are fully following the recommendations above. They are provided
here as reference, and as a way to see what it looks like when all the recommendations are followed.

### Overture

[Overture Maps](https://overturemaps.org/) provides a number of different 'themes' of data in well-organized GeoParquet files, with larger datasets. See [their documentation](https://docs.overturemaps.org/getting-data/) for instructions on how to get
the data. Their buildings data is more than 2.2 billion rows, and follows all the core recommendations above. The row group
size seems to be around 150,000, and it's zstd compressed with the bbox column, ordered by a GeoHash. The data is partitioned
spatially, see [this discussion comment](https://github.com/opengeospatial/geoparquet/discussions/251#discussioncomment-11478379)
for more details.

### Almost Exemplar

These datasets are all 'good enough' to use, but don't quite follow all the recommendations above. Once they are updated we'll
move them up.

* The [Google-Microsoft-OSM Buildings - combined by VIDA](https://source.coop/repositories/vida/google-microsoft-osm-open-buildings/description) is a great example of a dataset that is almost following all the recommendations above. They did use snappy, and
their row group sizes are around 5,000 (which still gets reasonable performance). They distribute the data in 2 different
partition schemes. One is just by admin boundary, which leads to a few really large files (India, USA, etc). The other further
splits larger countries into smaller files, using S2 cells.

* [US Structures from Oak Ridge National Laboratory](https://source.coop/wherobots/usa-structures/geoparquet) formatted by
Wherobots.

* [Planet Ag Field Boundaries over EU](https://source.coop/repositories/planet/eu-field-boundaries/description) - needs to be
spatially partitioned, row group size is 25,000.

## Examples in common tools

This section will discuss what each tool does by default, and show any additional options
needed to follow the recommendations above. STAC metadata and spatial partitioning will
have their own sections, since there are fewer tools that can do it, but most any of
the other tools can be used to prep the data.

### GDAL/OGR

Out of the box:

```
ogr2ogr out.parquet in.geojson
```

Out of the box GDAL/OGR defaults to snappy compression, with max row group size of 65536.
Version 3.9 and later will write out the bbox column by default. And there is a built-in
option to spatially order the data that works by creating a temporary GeoPackage file and
using its r-tree spatial index. It defaults to false since it can be an intensive operation,
and GDAL is usually translating from formats that already have spatial indexes.

### GDAL/OGR with recommended settings

These examples are done with the `ogr2ogr command-line tool, but the layer creation options
will be the same calling from C or Python.

Without spatial ordering (use when source data already has spatial index (GeoPackage, FlatGeobuf, Shapefile, PostGIS, etc))
```
ogr2ogr out.parquet -lco "COMPRESSION=ZSTD" -lco "MAX_ROW_GROUP_SIZE=100000" in.fgb
```

With spatial ordering (use when source data does not have spatial index):
```
ogr2ogr out.parquet -lco SORT_BY_BBOX=YES  "COMPRESSION=ZSTD" in.geojson
```

### GeoPandas (Python)

### DuckDB

Out of the box:
```
load spatial;
COPY (SELECT * FROM geo_table) TO 'out.parquet' (FORMAT 'parquet');
```

DuckDB will automatically write GeoParquet as long as it's version 1.1 and above, the [spatial extension](https://duckdb.org/docs/extensions/spatial/overview.html)
is enabled and the table has geometries The default compression is snappy, and the max row group size is 122,880, and the bbox column is always written out. You can control the [compression](https://duckdb.org/docs/sql/statements/copy.html#parquet-options) and [row group size](https://duckdb.org/docs/data/parquet/tips.html#selecting-a-row_group_size):

```
COPY (SELECT * FROM geo_table) TO 'out.parquet' (FORMAT 'parquet', COMPRESSION 'zstd', ROW_GROUP_SIZE '100000');
```

Interestingly you can also set the row group size in bytes, which would likely be a better way to handle geospatial data since the
row size can vary so much.

```
COPY (SELECT * FROM geo_table) TO 'out.parquet' (FORMAT 'parquet', COMPRESSION 'zstd', ROW_GROUP_SIZE_BYTES '128mb');

```

DuckDB also has functionality to spatially order your data, with the `[ST_Hilbert](https://duckdb.org/docs/extensions/spatial/functions#st_hilbert)`
function. It is strongly recommended to pass in the bounds of your entire dataset to the function call or the hilbert curve
won't be built right. The following call will dynamically get the bounds of your dataset, and pass that into the ST_Hilbert function.

```
copy (SELECT *
  FROM geo_table
  ORDER BY ST_Hilbert(
      geom,
      (
          SELECT ST_Extent(ST_Extent_Agg(COLUMNS(geom)))::BOX_2D
          FROM fsq
      )
  )) TO 'out.parquet' (FORMAT 'parquet', COMPRESSION 'zstd');
```

### Sedona

### GPQ (Go)

TODO: add more tools.
