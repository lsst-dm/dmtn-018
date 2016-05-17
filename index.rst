
:tocdepth: 1

This Technical Note summarizes current state of understanding of various
issues involved into design of the L1 Database use by Alert Production
pipeline. This is not a final document and will likely be updated when more
information becomes available. Initial revision is compiled from
information in JIRA tickets
`DM-5316 <https://jira.lsstcorp.org/browse/DM-5316>`_,
`DM-5509 <https://jira.lsstcorp.org/browse/DM-5509>`_, and
`DM-5545 <https://jira.lsstcorp.org/browse/DM-5545>`_.

Following sections cover various aspects of the design in no particular
order.


L1 Database schema
==================

The schema of L1 database is outlined in
:abbr:`DPDD (Data Product Definition Document)`
(`LSE-163 <http://ldm-135.lsst.io/>`_). There are three tables which
contain most of the data of L1 database -- DiaSource, DiaForcedSource, and
DiaObject. Here are the approximate sizes in bytes of the rows in those
tables as given in Sizing Model Spreadsheet
(`LDM-141 <http://ls.st/LDM-141>`_) and estimated from DPDD.

.. table:: Row sizes of L1 tables.

   +-----------------+------------------+
   |                 | Row size, bytes  |
   +-----------------+----------+-------+
   | Table name      | LDM-141  | DPDD  |
   +=================+==========+=======+
   | DiaObject       |    1,124 | 1,536 |
   +-----------------+----------+-------+
   | DiaSource       |      333 |   232 |
   +-----------------+----------+-------+
   | DiaForcedSource |       41 |    33 |
   +-----------------+----------+-------+
   | SSObject        |      488 |   424 |
   +-----------------+----------+-------+

There is some discrepancy between two sets of numbers, calculations below
used estimates from LDM-141.


Current baseline assumptions
============================

Baseline design of L1 is explained in `LDM-135 <http://ldm-135.lsst.io/>`_
and it is based on input from DPDD and the figures from Sizing Model
Spreadsheet (LDM-141). Baseline design assumes that L1 database is replaced
every year by an improved version produced by DRP as a part of the Data
Release. It is expected that L1 data from Data Release does not contain the
full history of DiaObject but only the most recent version of it while
still keeping full set of DiaSource and DiaForcedSource tables. Because
DiaObject has the largest record size resetting DiaObject after each DR
helps to control the size of the DiaObject table.


:ref:`Figure 1 <fig-l1_row_count_old_baseline>` shows an estimated behaviour
of the count of the rows in three largest tables from L1 databases based on
the above assumptions. :ref:`Figure 2 <fig-l1_table_size_old_baseline>`
shows the sizes of the corresponding tables calculated for the numbers of
rows. Calculated sizes are based on the row sizes as defined in LDM-141
which do not quite agree with DPDD-defined schema and will likely need some
adjustment. Also these sizes do not take into account overhead due to data
organization and indexing which can be significant.

In the baseline design the total size of the tree major tables will likely
reach and exceed 100TB mark per single L1 database replica.


.. figure:: /_static/l1_row_count_old_baseline.png
   :scale: 66%
   :align: center
   :name: fig-l1_row_count_old_baseline
   :alt: L1 row count

   Estimate of the row count in L1 database for three major tables.

.. figure:: /_static/l1_table_size_old_baseline.png
   :scale: 66%
   :align: center
   :name: fig-l1_table_size_old_baseline
   :alt: L1 row count

   Estimate of the table sizes in L1 database for three major tables.


Revised L1 database design
==========================

L1 database switch-over after each DRP is seen as too disruptive for
scientific community so
`new approach <https://confluence.lsstcorp.org/pages/viewpage.action?pageId=45580703>`_
was proposed by Science Pipelines Definition Working Group. In this
proposal there is no annual switch-over to a DRP-produced L1 tables,
instead L1 database continues to accumulate its data indefinitely. This
means that the whole history of the DiaObject is kept forever which
increases significantly the row count and size of the DiaObject table.

:ref:`Figure 3 <fig-l1_row_count_new_proposal>` shows the estimated row
count for the same major tables in the new proposal (DiaSource and
DiaForcedSource counts do not change).
:ref:`Figure 4 <fig-l1_table_size_new_proposal>` shows the sizes of the
corresponding tables.

In this new proposal the total size is dominated by DiaObject table and
could reach 400TB (without accounting for overhead for indexing and
storage).

.. figure:: /_static/l1_row_count_new_proposal.png
   :scale: 66%
   :align: center
   :name: fig-l1_row_count_new_proposal
   :alt: L1 row count (new proposal)

   Estimate of the row count in L1 database in new proposal, baseline
   estimate of the DiaObject is given for comparison.

.. figure:: /_static/l1_table_size_new_proposal.png
   :scale: 66%
   :align: center
   :name: fig-l1_table_size_new_proposal
   :alt: L1 row count (new proposal)

   Estimate of the table sizes in L1 database in new proposal, baseline
   estimate of the DiaObject is given for comparison.


Queries on L1 database
======================

The most important client of L1 database is an Alert Production (AP)
pipeline, so L1 database will be optimized for queries generated by AP.
Pipeline will need following data from L1 database for its operation on a
single visit:

- Most recent version of all DIAObjects in the region covered by CCD (this
  may be limited to DIAObject produced in last 12 months only)

- Last 12 months of DIASources from the same region, matching the
  DIAObjects selected above

- History of DIAForcedSources from the same region, again the same 12
  months and matching selected DIAObjects

AP will produce more data to be stored in L1 database for each visit:

- set of DIASources discovered in a visit

- set of DIAObjects (either new or updated versions), for new objects also
  store association to L2 Objects.

- set of DIAForcedSources for forced measurements (DIAForcedSources will
  also be produced in precovery processing which happens during the day)

The queries to retrieve the data have both spatial and temporal constrains.
Given the baseline schema (LDM-135) for DIAObject table which includes
validity interval above translates in the following set of queries:

- Select all most recent versions of DIAObjects for specified region,
  additionally this may be limited to records not older than 12 months.
  Region may cover either a single CCD or the whole camera region depending
  on how Source-to-Object matching is performed.

- Select DIASource and DIAForcedSource history for the last 12 months
  which match the set of DIAObjects selected in previous query. It may be
  more efficient to select based on the region and then filtering unmatched
  sources on a client side.

- Insert new set of DIAObjects, this may also update validity interval of
  the latest version of the same object already stored in the database.

- For new DIAObjects store associations of the DIAobject to L2 Objects.

- Insert new set of DIASource and DIAForcedSource records.


Visit data dependency
=====================

There will be spatial overlaps between consecutive visits or there may be
multiple consecutive visits of the same region. This causes data dependency
between jobs processing these consecutive visits because each job will
require a set of the most recent DIAObjects which may overlap with the set
of the DIAObjects produced in the previous visit. This argument applies to
the DIASource and DIAForcedSource as well.

There are significant implications of this dependency for the L1 database
access pattern. Without data dependency it would be possible to start
pre-loading of all necessary L1 data as soon as pointing coordinates become
available (typically right before first exposure). With data dependency the
data can be pre-loaded but some of that data may need to be updated when
processing of the previous visit finishes (typically minute or less after
the second exposure).

Solving this problem may require additional synchronization and/or data
exchange between jobs processing different visits. Alternatively all
processing that needs L1 data (sources association and alert production)
could be moved to a separate processing to be done sequentially with the
results from one visit being immediately available to next visit, though
this may severely limit parallelism.


Partitioning
============

It is clear that the volume of data in L1 database will be extremely large.
Updating and indexing data volumes of that scale needs special care for
scalability issues. One standard tool to keep these issues under control is
horizontal partitioning of large tables. In case of L1 database
partitioning can be done based on time ranges with a range length related
to typical time periods of typical queries (e.g. 12 months or shorter).
With that approach only the latest shards will be update-able and queries
will typically only use limited number of recent shards.

Partitioning introduces additional complications for operations as new
partitions need to be created regularly, and for data access which requires
writing queries involving one or more shards. There may be additional
complications for DiaObject table whose records represent time intervals
which potentially can span more than one shard.

In addition to regular partitioning it may be reasonable to create
specialized partitions or views that contain data relevant for some
specific queries (e.g. query for most recent versions of DiaObjects). These
views will result in duplication of some information from the regular
database which requires special care to guarantee data consistency.

Indexing and Locality
=====================

All most important queries from the list above require geometry-based
selection of the small patch of the sky, typically of a single CCD or the
whole region of a single visit. It is extremely important to serve this
sort of queries in a most efficient way. One possible approach for fast
access is to use spatial indexing optimized for spherical geometry. There
may be several candidates for this sort of indexing scheme such as
Hierarchical Triangular Mesh (HTM) or Quad Tree Cube (Q3C). Both of these
two indices map small sky regions to a set of integer numbers, and for both
of them mapping of CCD regions to index ranges can be performed on client
side without special support on database server side. These two indexing
schemes will be implemented in LSST package ``sphgeom``.

In addition to spatial indexing other indices may be important, one example
of this is selection of the most recent DiaObject version. It could
possibly be achieved via special index on DiaObject validity interval
though this may create additional overhead in both space and CPU time.
Specialized structures could be used in some cases, e.g. already mentioned
above special partitions with a subset of the data.

Typical queries will return large number of records, this may translate into
a large number of I/O operations for database server. To reduce number of
disk seeks and resulting latency it will be important to think about data
locality and try to keep related data physically close on disk. How exactly
this can be controlled depends on particular technology, e.g. MySQL InnoDB
engine orders data according the primary index and MyISAM table format
stores records in order of insertion. Some storage technologies (SSD) can
help to avoid the issue by reducing seek time though exact gain is hard to
predict. Extensive prototyping at large scale will be need to understand
and evaluate possible approaches.


Replication and fail-over
=========================

L1 database needs a high-availability solution with a reasonably short
downtime in case when fail-over is needed. Additionally a read-only
instance of L1 database will be deployed at a base site and it will need to
be updated continuously from the master copy with a reasonably short delay.

Arguably simplest architecture to satisfy these requirements consists of the
two instances at the main site with master-master replication between them
and a slave instance at base site with master-slave replication from the
main site. Master-master replication allows quick fail-over in case of on
master failure, fail-over does not need re-configuration of the server
instances and happens entirely on client site. Currently MySQL C API does
not support transparent fail-over but it can be implemented in higher-level
API or with the help of a separate proxy layer. Master-slave replication
only supports single masters, in case of the master failure it will need
manual intervention to switch to a different master instance.

In addition to a basic MySQL replication solution there are other
third-party replication solutions (e.g. MariaDB Galera Cluster) which could
be used for the same purpose. PostgreSQL also supports master-master and
master-slave replication mechanisms and could be used in a similar
architectural approach.

.. figure:: /_static/L1replication-640.png
   :align: center
   :name: fig-l1_replication_architecture
   :alt: L1 database replication architecture

   Anticipated architecture for L1 replication.
