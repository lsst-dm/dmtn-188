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

   This note describes how the IVOA UWS will be used within the Data Management System, outlines some implementation issues, and presents a near-term plan.

Introduction
============

The IVOA Universal Worker Service (UWS) allows clients to create asynchronous jobs that perform a computation and return results.
Such a service will be needed to implement the SODA service, which is defined in terms of UWS.
It can be used to implement the precovery service.
With some optimizations to minimize startup latency, it can also be used to implement the OCPS and Prompt Processing.
UWS acts as a hub, providing a single consistent interface to one or more back-end execution services.

.. figure:: /_static/UWS_role.png
   :name: fig-uws-role
   :alt: UWS is the execution hub.

   UWS is the execution hub.

UWS can provide a common interface to back-end execution services ranging from Kubernetes Jobs to SLURM to HTCondor to a custom Science Pipelines server.


Proxy versus Hand-off versus Framework
======================================

There are three ways to implement user-facing services using UWS.

1. The user-facing service can act as a proxy for an underlying generic UWS service.
   All calls to UWS endpoints for the user-facing service are translated into calls to the UWS service with appropriate defaults and specializations.

2. The user-facing service can hand off requests to an underlying generic UWS service.
   Only the original job creation goes to the user-facing service.
   All other calls to UWS endpoints go directly to the UWS service.

3. The generic UWS service provides a framework into which the custom user-facing service code can be inserted.


Job Database Location
=====================

Many back-end execution services maintain their own databases of jobs, job statuses, and results.
It is relatively simple for a generic UWS service to retrieve this information from the back-end, translating as necessary into the standard UWS form.
The only difficulty is with UWS job editing prior to submission.
Since the back-end has not seen the job, there's no way to obtain an appropriate back-end job id, but some permanent id is required to allow the editing functionality.
This requires at a minimum a small mapping database between UWS job ids and back-end job ids.
Alternatively, we could violate part of the standard and explicitly disallow job editing.


Job Result Location
===================

UWS returns a list of URLs to job results.
Those URLs could point to the user-facing service, to the generic UWS service, or directly to storage (with appropriate pre-signing to avoid the need for authentication).

Results are read-only and unavailable until the job has completed successfully.
Intermediate pipeline results do not need to be returned to the user.

Pipeline executions will leave results in a Butler output collection.
All datasets in that collection can be listed, including either object store direct access URLs or ``file://`` URLs.
The former can be pre-signed and the latter can be translated to pre-signed HTTPS URLs for a generic web server, either embedded within the generic UWS service or separate from it.

In the case of the OCPS, outputs can be filtered to particular dataset types, allowing only calculated metrics to be returned as result payload.


Startup Latency Optimization
============================

One of the primary concerns for the OCPS and Prompt Processing is to minimize the latency from job submission to job execution.
In both cases, sub-second latency is desirable: for the OCPS, immediate feedback to operators and Commissioning personnel is needed while Prompt Processing needs to minimize consumption of its 60 second latency budget.
There are several progressive steps that can be taken to optimize this latency:

1. Avoid container startup time by having a pool of pods running to service job requests; auto-scale this pool as needed to handle surges of requests.

2. Avoid process startup time (including pipeline code import overhead and Butler creation overhead) by having an activator service running in each pod that would accept job descriptions and execute them.

3. Avoid data loading time by having the activator service preload configured master calibration information.

A load balancer might be an appropriate starting point to use to distribute job requests to already-running pods.


UWS Implementations
===================

A prototype execution service has been written in lsst-dm/uws-api-server.
It relies on the back-end execution service to maintain job state information.
It is not yet fully standard-compliant, but it can be extended to fully support the UWS protocol (or a sufficiently large subset).

OPUS (https://github.com/ParisAstronomicalDataCentre/OPUS), produced by the Paris Astronomical Data Center, appears to be a standard-conforming UWS server written in Python.
It uses an internal SQLite database to track job information uniformly, no matter the actual back-end service.
Back-end jobs are expected to contact special internal service endpoints to indicate progress.
It contains mechanisms to allow end users to propose new job types and have them be approved by service administrators that Rubin is unlikely to require.

The Daiquiri framework (https://github.com/django-daiquiri/daiquiri/ and https://django-daiquiri.github.io/docs/) appears to include UWS endpoints along with its implementation of TAP, cone search, and other features.
Extracting only the UWS features may be more difficult than with OPUS.


Near-Term Tasks
===============

0. (KTL) Develop a job definition to execute a generic pipeline and extract URLs to its results.

1. Integrate the prototype service and generic pipeline job execution with user-facing services:
   A. Image service by KSK
   B. OCPS CSC by KTL

2. Investigate whether it makes sense to leverage the OPUS and/or Daiquiri code to help achieve full standard compliance.

3. If it does, write a custom Manager class to execute jobs using the Kubernetes Jobs service, based on the existing prototype.  If it does not, extend the prototype to be as standards-compliant as possible.

4. Minimize job startup time by progressively implementing the optimizations given above, eventually resulting in developing a pipeline activation server that can preload datasets.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
