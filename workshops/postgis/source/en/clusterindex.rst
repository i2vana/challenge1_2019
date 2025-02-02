.. _clusterindex:

Clustering on Indices
=====================

Databases can only retrieve information as fast as they can get it off of disk. Small databases will float up entirely into RAM cache, and get away from physical disk limitations, but for large databases, access to the physical disk will be a limiting stop in disk access speed.

Data is written to disk opportunistically, so there is not necessarily any correlation between the order data is stored on the disk and the way it will be accessed or organized by applications. With the advent of Solid State Disks (SSD) this is changing to improve performance, `in the postgis Data Management and Queries <https://postgis.net/docs/using_postgis_dbmanagement.html#idm2461>`_ is mentioned the configuration for an index on an SSD.

.. image:: ./screenshots/clustering1.jpg
  :class: inline

One way to speed up access to data is to ensure that records which is likely to be retrieved together in the same result set are located in similar physical locations on the hard disk platters. This is called "clustering". 

The right clustering scheme to use can be tricky, but a general rule applies: indexes define a natural ordering scheme for data which is similar to the access pattern that will be used in retrieving the data.

.. image:: ./screenshots/clustering2.jpg
  :class: inline

Because of this, ordering the data on the disk in the same order as the index can provide a speed advantage in some cases.

Clustering on the R-Tree
------------------------

Spatial data tends to be accessed in spatially correlated windows: think of the map window in a web or desktop application. All the data in the windows has similar location value (or it wouldn't be in the window!)

So, clustering based on a spatial index makes sense for spatial data that is going to be accessed with spatial queries: similar things tend to have similar locations.

Let's cluster our ``nyc_census_blocks`` based on their spatial index:

.. code-block:: sql

  -- Cluster the blocks based on their spatial index
  CLUSTER nyc_census_blocks USING sidx_nyc_census_blocks_geom;

The command re-writes the ``nyc_census_blocks`` in the order defined by the spatial index ``sidx_nyc_census_blocks_geom``. Can you perceive a speed difference? Probably not, because the table is quite small and easily fits into memory, so disk access overhead doesn't affect performance.

One of the surprises of the R-Tree is that an R-Tree built incrementally on spatial data might not have high spatial coherence of the leaves. For example, see this visualization of the spatial index leaves of an index on roads in the province of British Columbia.

.. image:: ./screenshots/clustering3.jpg
  :class: inline

We would prefer to cluster using a more spatially compact tree, like this balanced R-Tree.

.. image:: ./screenshots/clustering4.jpg
  :class: inline

We don't have a balanced R-Tree algorithm available in PostGIS, but we do have a useful proxy that puts spatial data into a spatially autocorrelated order, the **ST_GeoHash()** function.

Clustering on GeoHash
---------------------

To cluster on the ST_GeoHash() function, you first need to have a geohash index on your data. Fortunately, they are easy to build.

Geohash is a public domain geocode system invented in 2008 by Gustavo Niemeyer, it encodes a geographic location into a short string of letters and digits. The first step is decoding it from a variant of base 32 using all digits 0-9 and almost all lower case letters except a, i, l and o, here is an example following the `hash ezs42 <https://en.wikipedia.org/wiki/Geohash>`_:

::

  Decimal | Base 32 
  ------------------
      0   | 0	
      1   | 1 
      2   | 2 
      3   | 3 
      4   | 4	
      5   | 5	
      6   | 6	
      7   | 7	
      8   | 8	
      9   | 9	
      10  | b	
      11  | c  
      12  | d 
      13  | e	
      14  | f 
      15  | g 
      16  | h 
      17  | j 
      18  | k 
      19  | m 
      20  | n 
      21  | p 
      22  | q
      23  | r 
      24  | s 
      25  | t 
      26  | u 
      27  | v 
      28  | w 
      29  | y 
      30  | z 

This operation results in the bits 01101 11111 11000 00100 00010. Assuming that counting starts at 0 in the left side, the even bits are taken for the longitude code (0111110000000), while the odd bits are taken for the latitude code (101111001001).

Each binary code is then used in a series of divisions, considering one bit at a time, again from the left to the right side. For the latitude value, the interval -90 to +90 is divided by 2, producing two intervals: -90 to 0, and 0 to +90. Since the first bit is 1, the higher interval is chosen, and becomes the current interval. The procedure is repeated for all bits in the code. Finally, the latitude value is the center of the resulting interval. Longitudes are processed in an equivalent way, keeping in mind that the initial interval is -180 to +180.

The geohash algorithm only works on data in geographic (longitude/latitude) coordinates, so we need to transform the geometries (to EPSG:4326, which is longitude/latitude) at the same time as we hash them.

.. code-block:: sql

  CREATE INDEX geohash_nyc_census_blocks ON nyc_census_blocks (ST_GeoHash(ST_Transform(geom,4326)));

Once you have a geohash index, clustering on it uses the same syntax as the R-Tree clustering.

.. code-block:: sql

  CLUSTER nyc_census_blocks USING geohash_nyc_census_blocks;

Now your data is nicely arranged in spatially correlated order!


Function List
-------------

`ST_GeoHash(geometry A) <http://postgis.net/docs/ST_GeoHash.html>`_: Returns a text string representing the GeoHash of the bounds of the object. 
