# Geohash, Geospatial Indexes, and Proximity Search

## 1. Why this matters

Many applications need questions answered by location:

- Which images were captured within 50 meters of this map point?
- Which driver, store, or taxi is nearest to me?
- Which points lie inside this city boundary or delivery zone?
- Which captures are close to this road segment and face the right direction?

A plain latitude/longitude pair is data, not an efficient search structure. A naive radius query computes a distance from the requested point to every row, which becomes expensive as the dataset grows. A geospatial index prunes most rows and leaves a much smaller candidate set for exact checks.

This note builds from coordinates to geohash and other spatial cells, then compares database indexes and tools for proximity search.

## 2. Coordinate foundations

### Latitude and longitude

- **Latitude** measures north/south position, from -90° at the South Pole to +90° at the North Pole.
- **Longitude** measures east/west position, conventionally from -180° to +180°.
- GeoJSON and many spatial APIs use coordinate order **[longitude, latitude]**. Other interfaces sometimes name arguments `(latitude, longitude)`. Check each API and be consistent.
- Earth is not a flat plane. A degree of longitude covers less ground as latitude approaches a pole; longitude also wraps across the antimeridian (around ±180°).

For global distances, choose a spherical or ellipsoidal distance model. A sphere-based great-circle distance is often adequate for candidate ranking; ellipsoidal geodesic calculations are more accurate. A local projected coordinate system can be efficient for local engineering work if its area and units are chosen correctly. Straight-line distance is not driving distance: road-network distance or travel time requires routing over a road graph.

### Geometry types

Geospatial databases commonly model more than points:

| Geometry | Example |
| --- | --- |
| Point | A taxi image capture or restaurant |
| LineString | A road segment or route |
| Polygon | A service area, park, or privacy zone |
| MultiPolygon | A country or group of service areas |

Proximity queries are often point-to-point, but geographic search engines also support containment, intersection, and distance-to-shape queries.

## 3. What is a geospatial index?

A geospatial index is an index structure that organizes records by spatial location. It helps answer spatial predicates such as “inside this box,” “within this radius,” “intersects this polygon,” or “nearest to this point.”

Most spatial indexes use a **two-stage process**:

1. **Candidate filter:** use an indexable approximation (often bounding boxes or cells) to find a manageable set of possible matches.
2. **Exact predicate:** calculate the actual distance or geometry relationship and discard false positives.

This is why a spatial index can be fast without pretending its approximation is the exact answer. PostGIS documents this pattern for `ST_DWithin`: the index narrows candidates using an expanded bounding box, then the distance check confirms them. `ST_Distance(...) < radius` by itself may calculate distance for every row, while `ST_DWithin` is index-aware when a suitable index exists. See [PostGIS spatial query guidance](https://www.postgis.net/docs/manual-dev/en/using_postgis_query.html).

### Common index families

- **R-tree family (often GiST in PostgreSQL):** indexes bounding boxes of points, lines, and areas. Useful for general spatial predicates and nearest-neighbor searches.
- **Quadtrees / geohash-like grids:** recursively divide 2D space into cells. Useful for prefix/range lookup and partitioning; cell-boundary behavior must be handled.
- **Space-filling curves:** map a multidimensional location to a one-dimensional sortable key. Geohash uses interleaved coordinate bits (a Z-order/Morton-style encoding); S2 uses a Hilbert-curve-based cell identifier.
- **Discrete global grid systems:** assign a cell ID to each location at a chosen resolution, such as H3 or S2. Useful for grouping, sharding, aggregation, and candidate generation.

A cell ID such as a geohash/H3/S2 index is often an **application-level spatial key**, not a replacement for a database's exact spatial index or distance predicate.

### Concrete examples of geospatial indexes

| Example | What it indexes / how it helps |
| --- | --- |
| PostGIS GiST (R-tree-style) | Bounding boxes in a tree; prunes regions for radius, intersection, and containment queries |
| Quadtree | Recursively splits a 2D area into four smaller squares; useful for maps, games, and spatial partitioning |
| Geohash | Encodes a rectangular grid cell as a base-32 string; prefixes support coarse grouping and range lookup |
| H3 | Assigns hierarchical hexagon/pentagon cell IDs to Earth locations; useful for aggregation, joins, and cell-based routing |
| S2 | Assigns hierarchical quadrilateral cell IDs on a sphere; useful for global indexing and covering regions with cells |
| MongoDB `2dsphere` | Database-managed spherical geospatial index for GeoJSON locations and shapes |
| Elasticsearch `geo_point` / `geo_shape` | Search index fields for nearby points, bounding boxes, polygons, and shapes |
| Redis GEO | Geohash-encoded scores in a sorted set, queried by radius or box for a hot set of locations |

These examples are related but not interchangeable. PostGIS GiST, MongoDB `2dsphere`, and Elasticsearch geo fields are database-managed indexes. Geohash, H3, and S2 are spatial cell/indexing systems that applications can use as keys or as part of a database/search design. Redis GEO is a data structure that uses geohash encoding internally.

## 4. Geohash

### Intuition

Geohash turns a latitude/longitude point into a short string. A longer string means a smaller rectangular cell. Nearby points often share a prefix, so a prefix can represent a coarse region.

Think of it as repeatedly answering “which half of the current world rectangle contains this point?” First split longitude, then latitude, then longitude again, alternating. Record each half-choice as a bit. Group bits into fives and map each group to one character from a 32-character alphabet.

### Encoding a point

Conceptual algorithm:

1. Start with longitude range `[-180, 180]` and latitude range `[-90, 90]`.
2. Split longitude at its midpoint. If the point is east/right of the midpoint, write `1` and keep the eastern half; otherwise write `0` and keep the western half.
3. Split the remaining latitude interval. If the point is north/above the midpoint, write `1` and keep the northern half; otherwise write `0` and keep the southern half.
4. Continue alternating longitude and latitude until enough bits are generated.
5. Group the bit stream into groups of five. Map each group (0–31) to a base-32 character.

For a point west of Greenwich and north of the equator, the first longitude bit is `0`, and the next latitude bit is `1`. These two decisions already narrow the world to a quadrant. Each additional bit halves either the longitude span or latitude span.

Encoding is deterministic, so the same coordinate and precision produce the same cell string. Decoding a geohash returns the cell bounds (and often its center), not the exact original coordinate. Store the original coordinate when exact location matters.

### Prefix hierarchy and precision

Removing trailing characters makes a coarser ancestor cell; adding characters makes a finer cell. Precision increases in steps of one character, but a cell is rectangular and its width and height alternate in how quickly they shrink.

Approximate cell dimensions near the equator (values are rough; longitude width shrinks toward the poles):

| Characters | Approx. width × height |
| ---: | ---: |
| 1 | 5,000 km × 5,000 km |
| 2 | 1,250 km × 625 km |
| 3 | 156 km × 156 km |
| 4 | 39 km × 20 km |
| 5 | 4.9 km × 4.9 km |
| 6 | 1.2 km × 0.61 km |
| 7 | 153 m × 153 m |
| 8 | 38 m × 19 m |
| 9 | 4.8 m × 4.8 m |

These are not guaranteed distance radii. A 7-character cell does **not** mean a point is within 153 meters of every other point in that cell; two points at opposite corners can be farther apart. Choose precision based on candidate density and query needs, then apply an actual distance predicate.

### The cell-boundary problem

Points only a few meters apart can have different geohashes if they fall on opposite sides of a cell edge. Therefore, a query must not search only the center point's geohash. That can miss valid neighbors.

Safer approaches:

- Calculate the query circle's bounding box, enumerate every geohash cell that intersects it, fetch candidates from those cells, and filter by exact distance.
- Or use a database-native spatial index and exact radius predicate, letting the database handle candidate generation.

Special care is needed at the antimeridian and near the poles. A naive longitude bounding box can fail when it crosses ±180°.

### Where geohash fits

Geohash is useful for:

- A readable location bucket or coarse location prefix.
- Prefix-based grouping and range scans in key-value stores.
- Sharding or routing records by approximate region.
- Map tiling / aggregation when its rectangular grid behavior is acceptable.

Geohash alone is a poor fit for:

- Exact radius membership or exact nearest-neighbor answers.
- Searching one cell without considering neighboring cells.
- Treating a geohash as a distance or as the original coordinate.
- Uniform-area analytics: cells have latitude-dependent shapes and sizes.

## 5. Other cell systems: H3 and S2

### H3

H3 is a global hierarchical grid, primarily hexagonal with 12 pentagons at each resolution. It converts coordinates to cell IDs, supports cell boundaries, neighbors, and parent/child relationships. Hexagons have six neighbors in the common case, which is convenient for aggregation and flow analysis. H3's own documentation cautions that hierarchical geometric containment is approximate in places, even though logical index containment is exact. See [H3 introduction](https://h3geo.org/docs/) and [H3 indexing notes](https://h3geo.org/docs/highlights/indexing/).

Good for: analytics by neighborhood, grouping and joining datasets by cell, spatial partition keys, and cell-based candidate generation. Do not assume a ring of H3 neighbors exactly equals an arbitrary-radius circle; cover the query region and do a precise final distance check.

### S2

S2 projects a cube onto a sphere and recursively subdivides it into hierarchical quadrilateral cells. Its cell IDs support parent/child traversal and region coverings. S2 is a strong option for global spatial indexing, adaptive region coverings, and partitioning. See the [S2 cell hierarchy guide](https://s2geometry.io/devguide/s2cell_hierarchy) and [S2 overview](https://s2geometry.io/about/overview.html).

### S2 versus S3

In geospatial indexing, the name commonly paired with H3 is **S2** (Google's spherical cell system), not S3. S2 and H3 both convert locations/regions into hierarchical cell IDs, with different cell shapes and hierarchy rules. [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) is a separate product: cloud object storage for files such as Street View image bytes. It is not a geospatial index and does not answer “what images are near this coordinate?” In the Street View architecture, S3 can store image files while PostGIS, a search index, or an H3/S2-backed index locates their metadata.

### Quick comparison

| Property | Geohash | H3 | S2 |
| --- | --- | --- | --- |
| Cell shape | Rectangular | Mostly hexagonal, with pentagons | Quadrilateral cells on a sphere |
| Identifier | Base-32 string | Cell index | Hierarchical cell ID |
| Natural hierarchy | String prefix | Resolution/parent-child API | Cell levels/parent-child API |
| Good fit | Simple prefixes and rectangular grid buckets | Aggregation and neighborhood analysis | Global region coverings and hierarchical indexing |
| Exact proximity by itself? | No | No | No |

Choose based on the operations and ecosystem you need. Do not switch to H3 just because hexagons look nicer on a map; if a database already answers exact spatial queries efficiently, a cell system may be unnecessary.

## 6. Proximity search

### Query types

- **Radius search:** all records within distance `r` of a point.
- **K-nearest neighbors (k-NN):** the nearest `k` records, with no fixed radius.
- **Top K within a radius:** candidates must satisfy the radius, then rank and limit.
- **Nearest along a route:** points near a line or route geometry, often with direction or progress constraints.
- **Network proximity:** shortest road-network distance or travel time; requires routing data and is different from straight-line distance.

Be precise about what “near” means. A map image query may use straight-line distance as an initial filter, then require road-segment match, heading compatibility, recency, privacy approval, and image quality.

### Correct radius-search pipeline

1. Validate coordinates, radius, and coordinate order.
2. Use a spatial index or covering cells to generate candidates.
3. Apply exact radius membership using a geodesic-aware database operation or trusted geospatial library.
4. Apply non-spatial filters: capture is processed, privacy-approved, retained, and accessible; optionally filter by time, heading, camera, map version, and quality.
5. Rank eligible results using a defined score (distance, heading difference, quality, recency, and coverage).
6. Return capture/asset IDs and metadata. Resolve image bytes through the asset catalog and authorized read path.

The index is an optimization. It must not weaken correctness or privacy filtering.

### Bounding-box prefilter and exact distance

A circle can be enclosed by a latitude/longitude rectangle, which is fast to search. The rectangle includes corner points outside the circle, so calculate exact distances afterward. At global scale, correctly handle longitude wrap and latitude-dependent geometry; avoid hand-written bounding-box math when a spatial database already provides a robust indexed predicate.

### Distance and ranking

For spherical points, the haversine formula is a common great-circle calculation. An ellipsoidal geodesic algorithm is more accurate over long distances. In most proximity systems, index candidate retrieval is the performance issue; using a more accurate distance for the resulting few candidates is affordable.

For Street View selection, a simple score could be:

~~~text
score = w_distance * normalized_distance
      + w_heading  * heading_difference
      + w_age      * age_penalty
      - w_quality * image_quality
~~~

This is only an example. Decide whether each term is a hard filter or a ranking factor. A close image facing away from the requested direction may be less useful than a slightly farther image facing the road correctly.

## 7. Database and tool choices

### Postgres + PostGIS

Use when you want relational metadata, transactions, mature geometry types, exact spatial predicates, and SQL joins in one place. Add a GiST spatial index and use index-aware predicates such as `ST_DWithin`; for nearest-neighbor ordering, PostGIS supports the `<->` KNN operator in `ORDER BY`. See [PostGIS spatial index FAQ](https://postgis.net/documentation/faq/spatial-indexes/) and [spatial query documentation](https://www.postgis.net/docs/manual-dev/en/using_postgis_query.html).

### What Postgres/PostGIS does under the hood

When you create `USING GIST (location)`, PostgreSQL builds a GiST (Generalized Search Tree) index. GiST is an extensible index framework; PostGIS implements an R-tree-style spatial index on top of it. Spatial index entries summarize geometries with bounding boxes rather than storing a full searchable copy of every shape. This makes the index quick to traverse, but the box is an approximation of the actual spatial object. See [PostGIS index internals](https://postgis.net/docs/manual-dev/en/using_postgis_dbmanagement.html) and [PostgreSQL GiST behavior](https://www.postgresql.org/docs/17/gist.html).

Conceptually, the tree looks like this:

~~~text
root bounding region
  ├─ bounding region for a group of index pages
  │    ├─ bounding box → table row pointer
  │    └─ bounding box → table row pointer
  └─ another bounding region
       ├─ bounding box → table row pointer
       └─ ...
~~~

For a radius query, the path is approximately:

1. **Plan the query.** PostgreSQL estimates whether using GiST is cheaper than scanning the table. A tiny table or a query covering most of the world may use a sequential scan, and that can be the right plan.
2. **Search the tree.** PostGIS derives an indexable spatial region that safely covers possible matches. GiST tests bounding regions and skips branches that cannot overlap the query region.
3. **Fetch candidate rows.** Leaf entries point back to table rows. A bounding box can overlap the search area even when the actual point is outside the radius, so these are candidates rather than final answers.
4. **Recheck the exact predicate.** `ST_DWithin` checks actual distance for candidates and removes false positives. For `geography`, the distance argument is in meters and the default uses spheroidal distance; the function includes an index-usable bounding-box comparison and a short-circuit distance calculation.
5. **Sort and limit if requested.** In a radius query ordered by `ST_Distance`, distances are calculated for the survivors, then sorted and limited. A KNN `ORDER BY location <-> query_point LIMIT k` can use GiST-assisted nearest-neighbor ordering; verify distance semantics and the plan for the chosen spatial type.

The key distinction: **GiST cheaply prunes regions; `ST_DWithin` supplies the correct radius result.** A spatial index alone does not skip every exact calculation. PostGIS documents that `ST_DWithin` is index-aware, while `ST_Distance(...) < radius` alone is not the preferred indexed radius filter. See [ST_DWithin](https://postgis.net/docs/ST_DWithin.html) and [PostGIS radius-search guidance](https://postgis.net/documentation/tips/st-dwithin/).

### What is physically stored?

There are two persistent PostgreSQL structures involved:

- **The table (heap):** stores the actual row, including the full `geography(Point, 4326)` value, `capture_id`, status, and other metadata. For a point, the spatial value is compact binary data inside the row. The image bytes remain in object storage, not in this row.
- **The GiST index:** is a separate PostgreSQL index relation, stored in PostgreSQL's normal database files. Its tree pages contain spatial summaries (bounding boxes) and references to child pages or table rows. It does not hold the image, nor is it a second full copy of each metadata row.

At query time, PostgreSQL reads index and table pages from its shared buffer cache when available, otherwise from storage managed by the operating system and database. Updates to the table and index are protected by PostgreSQL's write-ahead log (WAL) for crash recovery and replication. These are PostgreSQL storage internals; application code still interacts with tables, indexes, and SQL rather than managing the files directly.

In short: **the table stores the exact coordinates and metadata; the spatial index stores bounding-box summaries plus pointers that lead back to table rows.**

To inspect what the database actually did:

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT capture_id
FROM capture
WHERE ST_DWithin(location, :query_geography, :radius_meters);
~~~

Look for an index or bitmap index scan involving `capture_location_gix`, and compare rows removed by the exact filter with rows returned. A sequential scan is not automatically a problem: test with realistic table sizes, radii, and geographic density. Keep planner statistics current so its estimates reflect the data.

Illustrative table and radius query (longitude first in `ST_MakePoint`):

~~~sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE capture (
    capture_id text PRIMARY KEY,
    captured_at timestamptz NOT NULL,
    location geography(Point, 4326) NOT NULL,
    heading_degrees real,
    privacy_status text NOT NULL,
    quality_score real
);

CREATE INDEX capture_location_gix ON capture USING GIST (location);

-- All approved captures within 50 meters, nearest first.
WITH query_point AS (
    SELECT ST_SetSRID(ST_MakePoint(:longitude, :latitude), 4326)::geography AS p
)
SELECT c.capture_id,
       ST_Distance(c.location, q.p) AS distance_meters
FROM capture AS c
CROSS JOIN query_point AS q
WHERE c.privacy_status = 'APPROVED'
  AND ST_DWithin(c.location, q.p, 50)
ORDER BY distance_meters
LIMIT 100;
~~~

This is a good default for learning and for an initial product. Confirm the actual query plan with `EXPLAIN (ANALYZE, BUFFERS)` and representative data. Partitioning and sharding add complexity; introduce them only when measured write volume, dataset size, or latency requires it.

### Practical Java application stack

For a Java service, a concrete starter stack is:

- **PostgreSQL + PostGIS** for durable metadata, spatial storage, and indexed proximity queries.
- **pgJDBC** to connect from Java.
- **JDBC or Spring `JdbcTemplate`** to run the spatial SQL. Keep the spatial predicate in the database so the GiST index can reduce candidates before rows cross into the JVM.
- **Flyway or Liquibase** to version the extension, table, and index migrations.
- **JTS** only when Java code also needs geometry construction or planar operations such as intersections and buffers. JTS describes its geometry model as planar; do not calculate global meter distances by applying `Geometry.distance()` directly to raw longitude/latitude degrees. PostGIS `geography` is a better default for global point proximity.

Example with Spring `JdbcTemplate` (assumes `location` is `geography(Point, 4326)`):

~~~java
String sql = """
    SELECT capture_id,
           ST_Distance(location,
               ST_SetSRID(ST_MakePoint(?, ?), 4326)::geography) AS distance_meters
    FROM capture
    WHERE privacy_status = 'APPROVED'
      AND ST_DWithin(location,
          ST_SetSRID(ST_MakePoint(?, ?), 4326)::geography, ?)
    ORDER BY distance_meters
    LIMIT ?
    """;

List<NearbyCapture> matches = jdbcTemplate.query(
    sql,
    (rs, rowNum) -> new NearbyCapture(
        rs.getString("capture_id"),
        rs.getDouble("distance_meters")),
    longitude, latitude,   // MakePoint uses X=longitude, Y=latitude
    longitude, latitude,
    radiusMeters,
    resultLimit);
~~~

Parameterize values rather than concatenating user input into SQL. Validate latitude, longitude, radius, and result limit at the API boundary. If using JPA/Hibernate, Hibernate Spatial can map spatial types, but native SQL is often the clearest way to begin with PostGIS-specific `geography`, `ST_DWithin`, and KNN behavior. See [Hibernate Spatial documentation](https://docs.hibernate.org/orm/8.0/userguide/html_single/) and the [JTS documentation](https://locationtech.github.io/jts/).

### Elasticsearch / OpenSearch

Use when geo filters need to combine with text search, faceting, and relevance ranking, or when a dedicated search tier is needed. Elasticsearch supports `geo_point`, `geo_shape`, distance, bounding-box, polygon, and grid queries; see [geo-point mappings](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/geo-point) and [geo queries](https://www.elastic.co/docs/reference/query-languages/query-dsl/geo-queries). OpenSearch has a similar geo field/query model; check its documentation and client compatibility for the deployed version.

Implementation outline:

1. Create an index mapping with a `geo_point` field for a capture point; use `geo_shape` when the indexed value is a road, region, or other geometry.
2. Index only query metadata needed for search: `capture_id`, coordinates, status, time, heading, quality, and perhaps road segment. Keep full image bytes and the authoritative asset catalog elsewhere.
3. Use a `geo_distance` filter for radius membership and `_geo_distance` sort for nearest-first results. Combine with status/time filters in the same query.
4. Have an outbox/event consumer update the search index after the source-of-truth catalog changes. Make updates idempotent and support rebuilding the index.

Example mapping and query:

~~~json
PUT captures
{
  "mappings": {
    "properties": {
      "capture_id": { "type": "keyword" },
      "captured_at": { "type": "date" },
      "privacy_status": { "type": "keyword" },
      "heading_degrees": { "type": "float" },
      "location": { "type": "geo_point" }
    }
  }
}

GET captures/_search
{
  "size": 100,
  "query": {
    "bool": {
      "filter": [
        { "term": { "privacy_status": "APPROVED" } },
        { "range": { "captured_at": { "gte": "now-90d" } } },
        { "geo_distance": {
            "distance": "50m",
            "location": { "lat": 37.7749, "lon": -122.4194 }
        } }
      ]
    }
  },
  "sort": [
    { "_geo_distance": {
        "location": { "lat": 37.7749, "lon": -122.4194 },
        "order": "asc",
        "unit": "m"
    } }
  ]
}
~~~

Treat the search index as a derived projection when another database is authoritative, and plan for refresh lag, reindexing, and reconciliation. Avoid storing a signed image URL in the search document; generate authorized read access when serving the result.

### MongoDB

Use when the application is already document-centric and proximity queries belong naturally beside document records. MongoDB's `2dsphere` index supports spherical proximity and geometry operations; see [MongoDB geospatial queries](https://www.mongodb.com/docs/manual/geospatial-queries/) and [2dsphere indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/geospatial/2dsphere/).

Store the location as GeoJSON, with coordinates ordered `[longitude, latitude]`, then create a `2dsphere` index. A `$near` query with GeoJSON and a `2dsphere` index sorts results nearest to farthest; `$maxDistance` is measured in meters for GeoJSON.

~~~javascript
db.captures.createIndex({ location: "2dsphere" });

db.captures.find({
  privacy_status: "APPROVED",
  location: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-122.4194, 37.7749]
      },
      $maxDistance: 50
    }
  }
}).limit(100);
~~~

Use a compound index or query filters for frequently selective non-spatial fields only after checking the query plan and MongoDB's index rules. A geospatial operator has special-index constraints, so validate whether the desired additional filtering/sorting can be done in the same query or in a subsequent aggregation step.

### Redis GEO

Use for a hot, often-changing set of live locations where very fast nearby lookup is useful, such as currently available vehicles. Redis provides `GEOADD` and `GEOSEARCH` for radius or box lookups; see [Redis geospatial commands](https://redis.io/docs/latest/develop/data-types/geospatial/). The native GEO data type is designed for point/radius-style needs, not the broader shape and query features of a full spatial database.

For example, maintain a set of currently available taxis. The member is a stable taxi ID; `GEOADD` updates that member's coordinates as it moves. `GEOSEARCH` returns IDs and optionally distance/coordinates:

~~~text
GEOADD taxis:available -122.4194 37.7749 taxi-481
GEOADD taxis:available -122.4140 37.7790 taxi-782
GEOSEARCH taxis:available FROMLONLAT -122.4194 37.7749 BYRADIUS 500 m ASC COUNT 100 WITHDIST WITHCOORD
~~~

Redis stores GEO members in a sorted set using a 52-bit geohash score; the sorted-set structure lets it scan score ranges corresponding to nearby grid cells, then discard points outside the requested shape. Its distance calculation assumes a sphere (Haversine), and the coordinate range is limited to roughly ±85.05° latitude. Those tradeoffs are often fine for nearby active vehicles, but verify accuracy and coverage requirements. The application can use returned IDs to fetch richer data from a hash/cache or the canonical database. Remove members when taxis go offline or become unavailable; do not let stale locations accumulate indefinitely. Redis is not the right sole archive for a durable multi-year image catalog. Persist canonical records elsewhere and treat Redis as a serving/cache layer when appropriate. See [Redis GEOADD internals and limits](https://redis.io/docs/latest/commands/GEOADD/).

### H3 / S2 libraries

Use these libraries when the application needs explicit hierarchical cell IDs for aggregation, routing to regional partitions, or approximate candidate generation. They complement a metadata database; they do not store image bytes and do not replace exact distance validation.

### Analytics engines

For large historical aggregations (coverage by city/day, capture density, or fleet planning), a warehouse or columnar analytics engine may be better than the latency-sensitive serving database. Keep the interactive nearby-image read path separate from batch analytics if they create conflicting workloads.

### Selection guide

| Main need | Starting choice |
| --- | --- |
| General application with relational metadata and nearby queries | PostgreSQL + PostGIS |
| Geo plus full-text relevance and facets | Elasticsearch/OpenSearch as search projection |
| Existing document application with native geo predicates | MongoDB `2dsphere` |
| Small hot set of live moving entities | Redis GEO as a serving layer |
| Cell aggregation, geographic partitioning, or spatial joins | H3 or S2 library, paired with storage/search |
| Large historical scans and summaries | Spatial analytics/warehouse engine |

### Java client libraries

The database owns the index and executes the proximity query; the Java client sends the request and maps results. For this Java-learning project, use SQL/native geo queries first, then add an ORM only if it helps the rest of the application.

| Backend | Java client / integration |
| --- | --- |
| PostgreSQL + PostGIS | pgJDBC; JDBC or Spring `JdbcTemplate` for `ST_DWithin` and KNN SQL |
| MongoDB | Official Java Sync Driver; its `Filters` builder supports `near`, `nearSphere`, `geoWithin`, and `geoIntersects` |
| Elasticsearch | Official Elasticsearch Java API Client |
| OpenSearch | OpenSearch Java client; choose this client for OpenSearch deployments |
| Redis | Jedis (synchronous) or Lettuce (async/reactive); call `GEOADD` / `GEOSEARCH` |
| H3 | H3 Java bindings when the application needs H3 cell IDs in addition to a database |

Keep client/server versions compatible and use the target engine's own client where the ecosystems differ. References: [MongoDB Java geo queries](https://www.mongodb.com/docs/drivers/java/sync/current/crud/query-documents/geo/), [Elasticsearch Java client](https://www.elastic.co/docs/reference/elasticsearch/clients/java), [OpenSearch Java client](https://docs.opensearch.org/latest/clients/java/), and [Redis Java clients](https://redis.io/docs/latest/develop/clients/jedis/).

## 8. Correctness, performance, and operational pitfalls

- **Coordinate order:** standardize API field names and test with known points; `[longitude, latitude]` is common in GeoJSON.
- **Units:** make radius units explicit (meters, kilometers, miles). Do not mix angular degrees and meters.
- **Index usage:** use index-aware operators/functions, and verify plans. A spatial column without a spatial index may still scan the full table.
- **Boundary coverage:** geohash/H3/S2 candidate cells must cover the full query region, including adjacent cells. Exact filtering removes false positives; incomplete covering causes false negatives.
- **CRS and distance model:** ensure the data type and SRID support the intended units and accuracy. Do not treat degrees as meters.
- **Antimeridian and poles:** test global edge cases; a simple longitude min/max box may be wrong across ±180°.
- **Skew and hot partitions:** city centers produce much denser data than rural areas. Use partitioning and resolution based on observed density.
- **Filter selectivity:** a geo query may return many candidates. Apply privacy/status/time filters early where the database can combine them efficiently.
- **Stale projections:** if search is asynchronous, document index lag and provide reconciliation/rebuild tools.
- **Pagination:** use stable cursor/continuation semantics for large result sets; offset pagination can become expensive and results can shift as new captures arrive.
- **Privacy:** precise location history is sensitive. Restrict access, minimize retention, audit queries, and keep raw image access separate from public derivatives.

## 9. Learning path and practical exercises

1. Manually encode two nearby coordinates into geohash prefixes and observe that shared prefixes indicate a shared coarse cell.
2. Find two nearby points on opposite sides of a geohash boundary and show why neighbor cells must be searched.
3. Implement radius search over a small in-memory set using haversine distance; compare it with a full scan.
4. Load sample points into PostGIS, create a GiST index, compare `ST_Distance(...) < r` with `ST_DWithin(...)`, and inspect `EXPLAIN ANALYZE`.
5. Compare an H3 cell cover with a true circular radius: count false-positive candidates, then apply exact distance.
6. Try the same query with the number of points multiplied by 10, then measure index size, ingest throughput, and p95 query latency.

## 10. Interview summary

A geospatial index quickly narrows a large dataset to candidate locations; an exact geometric calculation determines which candidates truly match. Geohash, H3, and S2 encode locations into hierarchical cells and are useful for bucketing, routing, and aggregation, but cell membership is not the same as exact proximity. PostgreSQL + PostGIS is a strong starting point when relational metadata and accurate spatial predicates belong together. Add a dedicated search tier or cell-based partitioning only when measured scale or query needs justify the extra complexity.

## References

- [PostGIS: spatial query and index-aware predicates](https://www.postgis.net/docs/manual-dev/en/using_postgis_query.html)
- [PostGIS: how to use spatial indexes](https://postgis.net/documentation/faq/spatial-indexes/)
- [H3 documentation](https://h3geo.org/docs/)
- [H3 indexing model](https://h3geo.org/docs/highlights/indexing/)
- [S2 cell hierarchy](https://s2geometry.io/devguide/s2cell_hierarchy)
- [S2 overview](https://s2geometry.io/about/overview.html)
- [Amazon S3 overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Elasticsearch geo queries](https://www.elastic.co/docs/reference/query-languages/query-dsl/geo-queries)
- [Elasticsearch geo_point mapping](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/geo-point)
- [MongoDB geospatial queries](https://www.mongodb.com/docs/manual/geospatial-queries/)
- [MongoDB Java driver geospatial queries](https://www.mongodb.com/docs/drivers/java/sync/current/crud/query-documents/geo/)
- [Redis geospatial commands](https://redis.io/docs/latest/develop/data-types/geospatial/)
- [Redis GEOADD internals and coordinate limits](https://redis.io/docs/latest/commands/GEOADD/)
- [Elasticsearch Java API client](https://www.elastic.co/docs/reference/elasticsearch/clients/java)
- [OpenSearch Java client](https://docs.opensearch.org/latest/clients/java/)
