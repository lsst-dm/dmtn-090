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

:tocdepth: 1

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

Purpose
=======

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

What standards to we want to support?
-------------------------------------

What clients do we want to ensure compatibility with?
-----------------------------------------------------

Architecture Diagram
====================

Call Flows
----------

Services
========

Database Service
----------------

Worker Service
^^^^^^^^^^^^^^

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
