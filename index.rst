..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 2

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Implementation guide for DAX Webservices.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

Abstract
========

This document outlines the purpose, reasoning, and suggested implementation
of the DAX Webservices.  The DAX Webservices are the way to get LSST data through
a set of IVOA compliant services that will allow any tool or client supporting
IVOA standards to query and retrieve data.

Having Standards
================

The purpose of the DAX Webservices is to provide an IVOA compliant interface
to access LSST data, both catalog data (and its metadata) as well as image
data.

In the DAX Webservices, we are taking the LSST tools and stack, and
providing a view that IVOA compliant tools can use.  The LSST way involves
technology like the Butler for finding FITS files on networked disks, and
querying our distributed database, QServ, which hosts our catalog.

The Butler and QServ are both very purpose built to support scientific
processing and analysis of the LSST dataset, and aren't intended
to be generic across different observatories or telescopes (though some parts
may be), whereas the DAX Webservices are intended to be a layer that
provides a generic interface on top.

Many times, the DAX Webservices will use the LSST specific toolchain to
back the processing of requests.  For example, to do image cutouts,
the image service will likely need to use the Butler to find the raw
images, as well as do cutouts and other operations on the raw images,
before sending them back in a compliant format.

For IVOA compliant catalog access, the standards revolve around TAP (Table
Access Protocol) and VOTable to represent the results.  The language
of TAP queries is ADQL, which is SQL-like, but has some differences between
the variant of SQL that QServ.  The DAX Webservices must rewrite the query
appropriately before dispatching it to QServ and awaiting the result. QServ
also has a specific way of doing async queries relating to shared scans.

In the following sections, I will outline the standards we intend to
implement, how we might implement them, and where the tricky parts may be.

Why are standards important?
----------------------------

Astronomers have been writing code for quite a long time.  In the era of
multi-messenger astronomy and cross-referencing datasets of multiple
instruments, one way needed to be created to standardize access to the
data.  Otherwise, every time a new telescope is created, owners and maintainers
of astronomy tools would have to add support for the new telescope, taking
into account the idiosyncratic nature of that particular instrument.

By providing one standard way of doing it, we allow for one tool to use
data from multiple instruments.  We can also use tools developed before LSST,
unchanged, to access LSST data if we want.  This allows for astronomers
to have working, tested tools before our telescope is even ready for this level
of integration testing.

There are many things about the standards that I find confusing, or disagree
with.  But like the law, the standard is the standard.  I'm no fan of XML, but
it is better to support the standard and not love it, than try and break bad.

Each time the standard is not followed, there may be features or unintended
consequences in 3rd party tools.  For example, if an unknown state was returned
from an API, or a required key was missing, each tool will respond and error
in its own way.  Some may stop working, some may ignore it, some may outright
crash or throw an exception.  Tools differ in their access patterns even
using a standard, so we also need to do compatibility testing with tools we
want to officially support for our users.

We can do things above and beyond the standard, but we should know that those
things cannot interfere with what is stipulated by the standard.  Some additions
may be nice, but these must be measured against the impact of implementing more
of the IVOA standards, which will be used by more tools rather than just our
custom LSST tools.

What standards to we want to support?
-------------------------------------

We will implement an interface that supports these standards.  Links
are provided to give a reference set of versions to allow people to
communicate about sections and page numbers of each particular
specification:

Catalog oriented standards:

- `TAP 1.1 <http://www.ivoa.net/documents/TAP/20170830/PR-TAP-1.1-20170830.pdf>`_

- `VOTable 1.3 <http://www.ivoa.net/documents/VOTable/20130920/REC-VOTable-1.3-20130920.pdf>`_

- `Universal Worker Service (UWS) 1.1 <http://www.ivoa.net/documents/UWS/20161024/REC-UWS-1.1-20161024.pdf>`_

- `ADQL 2.0 <http://www.ivoa.net/documents/REC/ADQL/ADQL-20081030.pdf>`_

Image oriented standards:

- `ObsTAP <http://www.ivoa.net/documents/ObsCore/20170509/REC-ObsCore-v1.1-20170509.pdf>`_

- `SODA 1.0 <http://www.ivoa.net/documents/SODA/20170604/REC-SODA-1.0.pdf>`_

- `SIA 2.0 <http://www.ivoa.net/documents/SIA/20151223/REC-SIA-2.0-20151223.pdf>`_


User storage standards:

- `VOSpace 2.1 <http://www.ivoa.net/documents/VOSpace/20180620/REC-VOSpace-2.1.pdf>`_

If you don't know what these standards are or how they fit in, don't worry!
In the :ref:`Services <services-label>` section, I will outline where each of
these come into play.

What clients do we want to ensure compatibility with?
-----------------------------------------------------

Some clients and tools are just part of the general ecosystem of astronomy tools.
We will need to support them.  We will also be building the SUIT (LSST Science
Platform) on top of many of these services.  The portal and the notebook aspects
will both be calling the services, and possibly passing IDs to async results
between them.

Here's a list of clients:

- Science Platform / SUIT

  - `Science Platform Design LDM-542 <https://ldm-542.lsst.io/LDM-542.pdf>`_

  - `Science Platform Requirements LDM-554 <https://docushare.lsst.org/docushare/dsweb/Get/LDM-554/LDM-554.pdf>`_

- `Tool for OPerations on Catalogues And Tables (TOPCAT) <http://www.star.bris.ac.uk/~mbt/topcat/>`_

Architecture
============

Diagram
-------

Call Flows
----------

Data Flows
----------

.. _services-label:

Database Service
================

TAP 1.1 & VOTable
-----------------

For querying the catalog that is hosted in QServ, we want to support
Table Access Protocol (TAP) v1.1.  As outlined in the spec, TAP is a
standard interface to provide a query (in ADQL) and return a table
(usually VOTable) with the results of that query.

The results are returned usually in VOTable format, which include
metadata about the columns and datatypes in the table, as well as the
data values.

In order to run queries, we use the /sync, and /async endpoints, which
are required parts of TAP 1.1.  There are other optional endpoints
in the spec, such as /tables, /examples, and /capabilities.  For a chart
that contains what is required reference page 10 of the TAP spec.

Sync, Async, and UWS
--------------------

According to the standard, we need to provide endpoints to run queries
either sync or async.  For queries submitted to the /sync endpoint, the
service blocks and waits for the response to return to the caller in the
response.  For /async, we can return an ID that can be queried in the
future to determine the results.  This will be useful for long running
queries where the query may take hours to run.  For /async queries, the
spec requires us to implement the UWS standard.

While the UWS standard does not specify how to run the jobs, it provides
a RESTful way of accessing the state, checking results, and providing
control over jobs, such as canceling.

TAP_SCHEMA
----------

The IVOA standards try to not only standardize access to data, but also
the discovery of that data.  Section 4 of the TAP 1.1 spec outlines
TAP_SCHEMA, which is required of TAP 1.1 implementations.  The idea is
for a caller to be able to discover the schema of what we are serving
(tables, columns, and data types) to craft their queries correctly.

The further parts of section 4 of the TAP 1.1 spec (4.1, 4.2, 4.3, 4.4)
outline the schema for database tables to be created that can hold
metadata about the data that is accessible through the endpoint.

To use this part of the service, you can submit a query through TAP,
and the names of the metadata tables and columns are well known.  The
results are returned in VOTable format like any other query.  In this
clever usage, we can have one transport to tell us about the metadata
as well as the data itself, using ADQL to query the metadata.

LSST Specific Requirements
--------------------------

While not covered generally by any IVOA specific standard, there are
a few things that we have as requirements that are more LSST specific.

Authentication and Authorization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

LSST data is not all public, and scientists may have their own private
datasets uploaded as well to do JOINS or other algorithmic analysis against.
We need to be able to authorize each user to use the LSST DAC resources
as well as protect their results from someone else trying to scoop their
research.  Many IVOA standards come from the era of public astronomy data,
so there may be some excitement here trying to add AAA to everything.


.. note::

    AAA needs a lot more work and deciding on hard requirements

Since we are using UNIX groups and other very posix level permission
schemes, we need to figure out how to respect these things in our Webservices,
which aren't always impersonating the user.  For example, to get a result file,
it'd be much easier to check the permissions rather than try to su to that
user, and see if they still have access (which brings in things like ACLs, and
UNIX group mechanics).  Depending on the level of auth required, we might be
able to restrict this to the creator of the query, rather than their group.
Either way, this will have to be determined.

History Database
^^^^^^^^^^^^^^^^

We want a history database of queries that can be looked through.  The
UWS spec defines that there is a way to get a list of jobs, both pending
and finished, so that is one way of accomplishing this goal.  Depending
on how long we want to persist this data for, we might want to back up
the queries, and index them in some other interesting way, probably through
some other kind of ancillary service.

Query text should be protected by auth to only allow a user to see their
own queries.

Large Result Sets
^^^^^^^^^^^^^^^^^

Since LSST queries may take a long time to run, and have large results
sets, we need to be able to cache large results sets (up to 5 GB of
results per query) for a reasonable period of time so they can be
retrieved.  This may be on the other of a few days or a week, since
some of the queries may be run overnight or over the weekend.

These results must also be protected so that only the user executing
the query can retrieve the results.  After the results are retrieved,
that user can obviously do what they will with the results (such as
share them).  While there are data rights implications here, once the
data is out of our control, it's out of our control.

Implementation
--------------

Now that we've established the particulars of what we want, let's 
dive into the implementation of this service now.

This service needs to:

1. Accept queries through a TAP compliant HTTP interface.
2. Record the query in the query history.
3. Determine what backend those queries should be dispatched to.
4. Rewrite original ADQL query to the SQL variant of the backend.
5. Dispatch the query, either locally or through a pool of workers.
6. Gather results from the query, and transform them into VOTable.
7. Put the results in a place that the user can download.

TAP Compliant Interface
^^^^^^^^^^^^^^^^^^^^^^^

There are many ways to write a webservice these days, including many
frameworks.  We know what URIs we want to serve, /sync and /async,
and that we want to serve results in XML.  We need to really reference
the TAP 1.1 spec for this part, implementing what we need to, such as
parameters (LANG, QUERY, MAXREC) as well as wrapping the results in a
VOTable format.

History Database
^^^^^^^^^^^^^^^^

.. note::
   We still need firm requirements on what the retention period and
   auth scheme should be for accessing the history database.

There are many data stores we could use for a history database.  Many
might even be tied to the execution of async jobs.  For example, the
distributed task framework celery uses RabbitMQ, Redis, MongoDB, to store
results and execution status.  This isn't just used to query the history
but to drive execution.  These databases can also be queried directly
by users, or we can add additional URIs to look through the history.

The UWS spec also mandates a way to list jobs, and get their results.
This is fairly analogous to the history database functionality we want,
as it lists the queries, their IDs, execution status, and result location.

Determine the Backend
^^^^^^^^^^^^^^^^^^^^^

Many specs use the TAP and VOTable standards as a way of transmitting
complex data.  For example, the TAP_SCHEMA table stores the metadata,
and could be on a different backend than the catalog itself, which is
hosted by QServ.  Some user generated (Level 3) data might also be
present in another database, such as Oracle or Postgres.  There are
also special tables for ObsTAP to look at image metadata.

The tricky part here is that if one database isn't hosting all the tables,
we need to inspect the query to determine what tables are being accessed,
and then route the query to the appropriate backend.  Different backends
might also have different load characteristics, such as the number of
running queries.

Query Rewriting
^^^^^^^^^^^^^^^

QServ doesn't speak ADQL.  Neither does Oracle.  We need to take the
ADQL query, inspect it, and rewrite it to work on the individual backends.

This may be to work around various quirks of different SQL variants and
implementations (such as how keywords work, or the way of limiting results,
or datatypes).

There are also some extensions to do very astronomical things, such as
cone and other spatial searches, as well as dealing with different
coordinate systems.

Query Dispatch
^^^^^^^^^^^^^^

Once we have the final query and we know where it's going, we are
ready to send the rewritten query to the backend and start getting the
results.  Since these results may be very large (GBs) or very small
(0 or 1 rows), we need to be able to support both cases in a performant
way.

For sync queries, the caller simply waits on the HTTP connection until
the results are available.  For async queries, since the caller will
make another request, we need to ensure that these requests will always
find the results, no matter how many TAP service copies we have.  This
means we can't really store results locally on the TAP service disk
(also this has the possibility of filling up the disk).  It is better
to have a central disk or shared place, so that results can be written
there, and then picked up by anyone handling getting the results.  This
also helps with keeping results through upgrades and transient failures.

It's also a good idea to separate out your front ends (things taking HTTP
requests) from your back end workers (which dispatch to the database).
This allows for a more even distribution of load across the workers, and
keeping the load on the backends (which don't scale as easily) in check.

As we gather these results, we need to put them also in the right format,
which is VOTable.  This may involve some coercing of data types to VOTable
data types, rather than the original backend.  Once the result is written
and in the correct format, we can record that the query is finished and
the results are available.

.. note::
   QServ also supports an async query mode.  We should investigate this
   to determine where it fits in with our plans.  Inevitably we will
   have to gather the results, and put them in a VO compliant format.

.. note::
   We need to figure out how to properly impersonate the user making
   the request.  Do we store their token, or use a service account and
   su to them?

Centralized Result Store
^^^^^^^^^^^^^^^^^^^^^^^^

After the user has completed their query, they will want their results,
which may be large.  They may be downloaded more than once, so we likely
want to keep the results sets around for at least a few days, to prevent
needing to rerun the same query on the database.

Because of the diversity of queries and their results sizes, and not
being able to know the size of the results from the query, we need to be
careful about local resources.  If the results were stored on the TAP
service nodes, we could easily fill up the disk of a kubernetes pod, which
might be 100GB (or 20 results).  The fragmentation of splitting the load
across multiple TAP service nodes might also be bad, since the sizes of
the results might be uneven, filling up some nodes and leaving others
empty.  We want to store all these in a central place, preferably with
URL access, so we can serve the results file directly off disk.

By having one place store the results, we eliminate the problem of the
client needing to contact multiple servers to find the results,
or the results not existing by the time the user checks for the results.

This could easily be an S3 like object store, or an NFS volume with
Apache or another web front end checking for auth on top.  Given that it
is simply serving up static files, this part should be relatively easy.

Image Service
=============

ObsTAP
------

ObsTAP is the way to query and determine metadata about image data.
By using the same TAP / VOTable infrastructure from the database service,
a user or client can craft a query against the available metadata to
discover what images exist that fulfill those critiera.

The types of queries that can be run are independent of the data being
served - the standard dictates what tables and columns must exist to
run queries against.  This helps general discoverability, as otherwise
those tables would have to be described first (probably through TAP_SCHEMA),
but by having a uniform data model, this allows one query to be run
against multiple ObsTAP endpoints and have it work everywhere.

In the ObsTAP spec, there are some great UML diagrams for the data model
on page 13-15.  Then the data model is expanded further with tables describing
the database metadata.  Table 1 has all the metadata that is absolutely
required, containing the usual suspects such as observation id, time, type
of data, ra, dec, are all there.  Section 4 on page 20 actually has the
TAP_SCHEMA minimal set of fields and their datatypes that can easily be
dumped right into TAP_SCHEMA.tables.

For some of these fields, we will have one identifier that is present
throughout, and mostly constant, such as instrument and type of data (image).
For fields that change, such as RA/DEC we will need to present that as a 
database table.  This can be the same backends that the Database Service
uses for TAP_SCHEMA and other associated metadata.

Two important basic fields are the access_url, and the access_format.  This
tells the client what URL it can go to to retrieve the image, and what
format (JPG, FITS) the image at that URL is encoded in.  The format is a
standard MIME-type.

Along with image metadata, ObsTAP also supports serving and querying
provenance data, although it is not required.

.. note::
  Are we going to use ObsTAP to serve provenance data?

SIA
---

SIA (Simple Image Access) is a simpler way than ObsTAP to discover
images based on parameters the caller provides.  This isn't done in
ADQL, but via a smaller list of parameter options. The SIA metadata
model is the same as the ObsCore data model, and if we have a database
of the ObsCore data model, it should be easy to field SIA queries
against it.

The types of query parameters of SIA are things like position, energy,
time, and wavelengths.  There is a list of parameters in Section 2.1
of the SIA spec, that outlines all the possible query parameters.

SIA, unlike TAP, ObsTap, and SODA, only provides a sync endpoint called
query, which takes a query string or post parameters, and returns a
VOTable consistent with that of ObsTAP responses (Section 3.1 SIA spec).

SODA
----

SODA (Server-side Operations for Data Access) is an IVOA standard
that covers the processing of server side image data before returning
it to the caller.  Since many of our image files are large, and the
portion of the file that the caller may care about is small, this makes
sense to be able to filter the data down on the server side to reduce
the amount of data transferred, along with the latency and cost of
such a transfer.

By allowing a user to select positional regions using the POS argument,
different regions can be selected, such as CIRCLE, RANGE, and user defined
shapes via POLYGON.  To find the image with the correct filter, the user
can use the BAND parameter, to provide a range of wavelengths to return.

Like the TAP service, SODA specifies a sync an async resource, of which you
need at least one.  Async behaves as a UWS service, just like TAP, and can
provide an ID that can be later retrieved for large result sets.

Depending on the arguments, one query can provide multiple image results,
for example looking at multiple bands, or drawing multiple CIRCLEs.

.. note::
   It looks like SODA allows for us to also do our own custom parameters,
   to allow for more operations to happen.  Other than the cutouts defined
   by the spec, what server side transformations do we need?


LSST Specific Requirements
--------------------------

Authentication and Authorization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Users will have to be authenticated, and authorized (with data rights)
to query these services and retrieve image data.  This security model
may be simpler to that of the TAP service, because people will likely
not be uploading their own images to be served by the SIA, SODA, or ObsTAP
interfaces.  This means that there is generally a consistent level of
protection needed that does not vary per user - everyone has the same
access to all the image data.

That being said, ObsTAP does support a field called data_rights, which
allows us to say that our dataset is either public, secure, or
proprietary (ObsCore B.4.4).  This will likely be one flag per data
release, which will either be proprietary, then public after it is
released.

History Database
^^^^^^^^^^^^^^^^

While it is not mentioned in the requirements, we might want to extend
the idea of the history database to encompass queries to the image service,
such as ObsTAP, SODA, and SIA queries.  Because of the authorization model
outlined above, the results are less likely to need to be secured between
users, allowing for caching and result reuse to be higher and easier to
accomplish in a secure manner.

Either way, we will want to audit the access logs to this service, and
attempt to determine usage patterns, to improve performance.

Large Result Sets
^^^^^^^^^^^^^^^^^

Because of the large size of the LSST data, including the images, we will
want to ensure that queries are limited to a resonable number of results,
to not put undue load onto the system.

Since we have to support async queries to SODA, and because those jobs
may take a while to run, it makes senes to use the same centralized results
backend to store the data and provide URLs to objects in that backend.


Implementation
--------------



Further considerations
======================

Load and Failure Characteristics
--------------------------------

Testing and Operations
----------------------

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
