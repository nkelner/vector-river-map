Map of American Rivers
======================

## A vector tile demonstration and tutorial

By [Nelson Minar](http://www.somebits.com/) <tt>&lt;nelson@monkey.org&gt;</tt><br>
May 2013<br>
See the [live map here](http://www.somebits.com/~nelson/tmp/rivers/)

<div style="background-color: #ffd"><b>Prerelease version</b>, not yet complete.</div>

-----------

This project contains everything you need from start to finish to make a
vector-based web map of American rivers. The demonstration map here is neither
particularly beautiful nor complex, but it is a complete example of how
to built a web map using tiled vector data into a web map. The source code is
open source and you are encouraged to read it and tinker with it.
The components integrated in this project are:

1. [NHDPlus](http://www.horizon-systems.com/nhdplus/), the source data for river flowlines.
2. [PostGIS](http://postgis.refractions.net/), a geographic database.
3. [TileStache](http://tilestache.org/), a vector tile [GeoJSON](http://www.geojson.org/) server
4. [Gunicorn](http://gunicorn.org/), a Python webapp server.
5. [Leaflet](http://leafletjs.com/) and [Polymaps](http://polymaps.org/), two Javascript libraries
for rendering maps.

It's a lot of pieces, but each one is pretty simple by itself. Combined
together you have a powerful open source mapping stack that can efficiently
serve vector data to web browsers. You're welcome to [see the map running
live](http://www.somebits.com/~nelson/tmp/rivers/) on my server, but the real
point of this project is to show developers all the pieces necessary to build
their own map using vector tiles. Read on for details of how the map is
constructed, and be sure to [check out the source code]().

## Quick start

* Install <a href="#required">required software</a>
* Run `downloadNhd.sh` to get data
* Run `importNhd.sh` to bring data into PostGIS
* Run `serve.sh` to start TileStache in Gunicorn
* Run `serverTest.py` to do a quick test on the server
* Load `rivers-leaflet.html` or `rivers-polymaps.html` to view the map

## About vector tiles

Vector tiles are an exciting, underutilized idea to make web maps that are
more efficient and flexible. Google Maps revolutioned online cartography by
popularizing the use of [map tiles](http://www.maptiler.org /google-maps-
coordinates-tile-bounds-projection/) to serve "slippy maps" with excellent
quality and interactivity. Most slippy maps are raster maps, essentially a
mosaïc of PNG or JPG images glued together. But a lot of geographic data is
intrinsically vector, lines and polygons. Pre-rendering geodata into raster
image tiles is a popular approach today. But serving data as vector tiles
can result in maps that are faster, smaller, and more flexible.

Vector tiles are starting to catch on in proprietary applications; for
instance most mobile maps are now rendered with vector data. But doing vector
mapping in the open source world is still a bit obscure. The
[Polymaps](http://polymaps.org/) Javascript library was an early pioneer in
rendering vector tiles but that capability has been seldom used, in part
because generating vector tiles was difficult. But vector tile servers are
starting to become more common. This tutorial relies on [TileStache's VecTiles
provider](http://tilestache.org/doc/TileStache.Goodies.VecTiles.html) to
serve our own prepared geodata. OpenStreetMap is also experimenting with
serving vector tiles of its data.

Tiling isn't necessary for all vector data. For example, our demonstration map
contains the US state boundaries as a single 88k GeoJSON file. If the full
dataset is small it is reasonable to serve an entire vector geometry as a
single file and let the client renderer take care of clipping. That works fine
for 100kB of data but is impractical with 10+MB of geometry. Cropping to tiles
optimizes sending only visible geometry. Scaling tiles enables data to be
simplified and down-sampled to match pixel visibility.

Vector tiles are ultimately quite simple: take a look at [this tile near near
Oakland](http://somebits.com:8001/riverst/13/1316/3169.json). The [URL naming
system](http://www.maptiler.org/google-maps-coordinates-tile-bounds-
projection/) is exactly like Google's convention for raster map tiles:
zoom/y/x. Only instead of serving a PNG image what comes back instead is a
[GeoJSON file](http://www.geojson.org/) describing the geometry inside that
region. This example has 6 features in it, each describing part of a river or
creek. Each feature contains a geometry, a name, and a [Strahler
number](http://en.wikipedia.org/wiki/Strahler_Stream_Order) encoding the
stream's approximate size or importance. The tricky thing about vector
tiles is what to do about features that cross tiles. In this tutorial we clip
the geometry to the tile boundary and rely on the overlapping lines being
drawn to make a seamless map. It's also possible to not clip, which results in
redundant data but keeps features intact. Not clipping is particularly important
for polygons.

<a name="required">## Server prerequisites</a>

The following is a partial list of software you need installed on your Unix
system to generate and serve these maps. (Sorry Windows users, Unix is a
better choice for this kind of work.) I've tested with both MacOS and Ubuntu.
On the Mac, most prerequisites are available via
[Homebrew](http://mxcl.github.io/homebrew/). On Ubuntu many are available via
`apt-get`, although the more recent versions from the [UbuntuGIS
PPA](https://wiki.ubuntu.com/UbuntuGIS) are  recommended. Other Linux
distributions can probably install the required software via their native
package system. If the code is available on
[PyPI](https://pypi.python.org/pypi) I prefer to install Python code with
[`pip`](http://www.pip- installer.org/en/latest/) rather than rely on the Mac
or Ubuntu package versions.

* [curl](http://curl.haxx.se/) for downloading NHDPlus data from the web.
* [p7zip](http://p7zip.sourceforge.net/) for unpacking NHDPlus data. Ubuntu users be sure to install `p7zip-full`.
* [PostgreSQL](http://www.postgresql.org/) and [PostGIS](http://postgis.refractions.net/) for a geospatial database.
* shp2pgsql, part of PostGIS, for importing ESRI shapefiles into PostGIS
* [pgdbf](https://github.com/kstrauser/pgdbf) for importing DBF databases into PostgreSQL. Unfortunately the Ubuntu/precise
version 0.5.5 does not have the `-s` flag needed for handling non-ASCII data. Install from
[sources](http://sourceforge.net/projects/pgdbf/files/pgdbf/) or insure you're
getting version 0.6.* from somewhere.
* [gunicorn](http://gunicorn.org/) for a Python web app server.
* [TileStache](http://tilestache.org/) for the Python web app that serves map tiles. TileStache has
undocumented dependencies on [Shapely](https://pypi.python.org/pypi/Shapely) and
[psycopg2](http://initd.org/psycopg/) that you can install by hand via `pip`.
* [requests](http://docs.python-requests.org/en/latest/) and [grequests](https://github.com/kennethreitz/grequests) for `serverTest.py`, a Python HTTP client test.
* [gdal](http://www.gdal.org/) is the low level library for open source geo. It will be installed
as dependencies by the tools above, listing it here for proper respect.


## Project components

Here are all of the pieces that go into building and serving a vector tile
map.

* `downloadNhd.sh` downloads data from [NHDPlus](http://www.horizon-
systems.com/nhdplus/), a nice repository of cleaned up National Hydrographic
Data distributed as ESRI shapefiles. This shell script takes care of
downloading the files and then extracting the specific shapefiles we're
interested in. NHDPlus is a fantastic resource if you're interested in mapping
water in the United States.

* `importNhd.sh` takes care of importing the NHDPlus shapefiles and auxiliary
data into a PostGIS database named `rivers`. The script doesn't do anything
particularly clever, but it's a lot of necessary glue. This script borrows
some ideas from [Seth Fitzsimmons' NHD
importer](https://gist.github.com/mojodna/b1f169b33db907f2b8dd)

* `processNhd.sql` prepares the imported data for fast
serving. The key thing is it makes a new table named `rivers` which contains
copies of the geometry from NHDFlowline and metadata such as river name and
[Strahler number](http://en.wikipedia.org/wiki/Strahler_number) from
PlusFlowlineVAA. This table is essentially a pre-prepared and indexed copy of
the NHD data for efficient serving by TileStache.

* `serve.sh` is a simple shell script to invoke Gunicorn to run the TileStache
webapp. In a real production deployment this should be replaced with a server
management framework. (It's also possible to serve TileStache via CGI, but
it's terribly slow.)

* `gunicorn.cfg.py` is the Gunicorn server configuration. There's very little
here in this example, Gunicorn has [many configuration
options](http://docs.gunicorn.org/en/latest/configure.html).

* `tilestache.cfg` sets up TileStache to serve a single layer named `riverst`
with a disk cache in `/tmp/stache`. It uses the [VecTiles
provider](http://tilestache.org/doc/TileStache.Goodies.VecTiles.html), the
magic in TileStache that takes care of doing PostGIS queries and serving back
nicely cropped GeoJSON tiles. At this layer we start making cartographic
decisions.

* `serverTest.py` is a very simple Python client test that inspects a few
vector tiles for basic correctness and reports load times.

* `rivers-leaflet.html` and `rivers-polymaps.html` are two alternate
Javascript map renderers. [Leaflet](http://leafletjs.com/) is an actively
maintained excellent Javascript map library; vector tile support is provided
by Glen Robertson's [leaflet-tilelayer-geojson
plugin](https://github.com/glenrobertson/leaflet-tilelayer-geojson).
[Polymaps](http://polymaps.org/) is an older Javascript map library that is no
longer actively maintained. Polymaps pioneered the vector tile idea and
renders vector maps very efficiently.

## Cartographic decisions

Most of the work in this project is plumbing, systems programming stuff
that we have to do to make the engines go. The demonstration map is deliberately
quite simple and unsophisticated. Even so, we've made a few cartographic
decisions.

Most of the actual cartography is being done in Javascript, in the Leaflet and
Polymaps drawing files. This project does very little, mostly just telling
the underlying library to draw blue lines in varying thicknesses. In addition
the Leaflet version has a simple popup when rivers are clicked. But with the
actual vector geometry and metadata available in Javascript a lot more could
be done in the presentation; highlighting rivers, interactive filtering by
Strahler number, combination with other vector data sources.

Some cartographic decisions are made on the server side. The TileStache
VecTiles configuration contains an array of queries that return results at
different zoom levels. At high zoom levels (say z=4) we only return rivers
which are relatively big, those with a [Strahler
number](http://en.wikipedia.org/wiki/Strahler_number) of 6 or higher. At finer
grained zoom levels we return more and smaller rivers. This per-zoom filtering
both limits the bandwidth used on zoomed out maps and prevents the display
from being overcluttered. On the other hand rendering zillions of tiny streams
can be [quite beautiful](http://nelsonslog.wordpress.com/2013/04/19
/california-rivers/).

VecTiles also does some extra simplification work for us. It simplifies the
geometry to serve only with the precision needed at the zoom level. You can
see this in action if you watch it re-render as you navigate; rivers will
start to grow more bends and detail as you zoom in. TileStache does that for
us automatically.



## Project ideas


The map provided here is very basic, just a tutorial demonstration. To make
this a better map, some possible directions:

* More beautiful river rendering. The rivers here are drawn as simple blue
lines with a static thickness based on the river's Strahler number, a
topological measure of its distance from headwaters. It'd be better to vary
thickness also based on the map's zoom level, or maybe change the color too,
or bring in extra information on river size such as flow rate or average
channel width.

* More thematic data. The ESRI relief tiles are nice because they show the
natural relationship between terrain and river flow, but it's a pretty minimal
base raster map. Why not add some ponds and lakes, or ground cover coloring,
or cities and major roads?

* Use a better HTTP server. Gunicorn is designed to run behind a proxy like
Nginx or Apache. Not only does the proxy handle slow clients better, it can
serve appropriate caching headers and gzip the JSON output. clients well.

* Merge the river data. The NHDPlus source data has rivers broken up into many
separate little LineStrings. The TileStache server naively sends them as
separate Features, needlessly duplicating properties like the name. It'd be
more efficient to merge all the data for a river into a single LineString,
although care has to be taken not to disrupt the efficiency of the Gist
spatial index.

* More efficient vector tiles. The code here downloads a new set of tiles for
every zoom level. But that's needlessly redundant; it's feasible to only
download new tiles every few zoom levels and trade off pixel-perfect accuracy
for smaller bandwidth.

* Convert to TopoJSON for smaller encoding. Even though there's no shared
topology, TopoJSON encoding can be roughly 2/3 the size of equivalent GeoJSON.

