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

- `SODA 1.0 <http://www.ivoa.net/documents/SODA/20170604/REC-SODA-1.0.pdf>`_

- `ObsTAP <http://www.ivoa.net/documents/ObsCore/20170509/REC-ObsCore-v1.1-20170509.pdf>`_

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

Services
========

Database Service
----------------

In this section I will describe the database service, it's interface
and a suggested implementation.

Database Service Interface
--------------------------

TAP 1.1 & VOTable
^^^^^^^^^^^^^^^^^
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
^^^^^^^^^^^^^^^^^^^^
According to the standard, we need to provide endpoints to run queries
either sync or async.  For queries submitted to the /sync endpoint, the
service blocks and waits for the response to return to the caller in the
response.  For /async, we can return an ID that can be queried in the
future to determine the results.  This will be useful for long running
queries where the query may take hours to run.  For /async queries, the
spec requires us to implement the UWS standard.

While the UWS standard does not specify how to run the jobs, it provides
a RESTful way of accessing the state, checking results, and providing
control over jobs, such as cancelling.

TAP_SCHEMA
^^^^^^^^^^

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

Database Service Implementation
-------------------------------

Image Service
-------------

Further considerations
======================

Authentication and Authorization
--------------------------------

Load and Failure Characteristics
--------------------------------

Testing and Operations
----------------------

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
