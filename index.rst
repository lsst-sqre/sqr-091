##########################################
Draft IVOA web service standards framework
##########################################

.. abstract::

   Proposes a framework for defining IVOA web standards that separates the service specification from the protocol encoding and supports JSON encoding of web service calls.

.. warning::

   This document is **not an IVOA standard** and is **not complete**.
   It is an experiment intended as input to ongoing IVOA discussions about future revisions to IVOA standards.
   The document is written as if it were an IVOA standard to save effort if parts of it can be cut and pasted into future documents, but it is not endorsed by the IVOA and is not part of any IVOA protocol.

.. seealso::

   :sqr:`092`: Draft IVOA JSON encoding
       A network protocol encoding based on HTTP and JSON that follows this framework.

   :sqr:`093`: Draft IVOA SODA web service specification
       An example network protocol specification for a simple SODA cutout service using this framework.

Standards structure
===================

An IVOA web service definition consists of two components: a specification for the input, output, and semantics of the web service; and an encoding to a network transport protocol.

The web service specification defines the service in terms of data types, without regard to the network protocol used to make requests and responses.
It is then coupled with a specification for the network protocol, which describes how to serialize and deserialize those data types, send a request, and receive a response.
Those two documents together provide the information required to write an implementation.

This document specifies what information must be included in a web service specification and what information must be included in a network protocol specification.
It also specifies some generic components that are common to many or all web services, such as structured error responses and a processing framework called the Universal Worker Service.

Definitions
===========

The following terms are used consistently throughout IVOA web services.

web service
    A client/server protocol for performing some astronomy-related query or operation.
    In the context of standards documents, a web service generally refers to the specification for a web service, not to specific implementations of that specification.

network protocol
    A specification for how a client and server talk to each other over a network that is not specific to any specific web service.
    The specification of a network protocol includes instructions for encoding generic objects and types, sending a request, and receiving a response, but does not specify the precise contents of the requests and responses or the semantics of any request.

object
    A structured representation of some information.
    At the top level, an object is a mapping from labels to values.
    The values may be complex structures, such as objects or lists of objects.

request
    A single request to a web service, resulting in a single response.

operation
    A single thing a web service can do, such as a query or a state changing operation like a delete or a modification.
    In some cases, such as redirects, this may involve multiple client requests and server responses, but there is only one input object and only one response object or data stream.
    A complex action using a web service, such as creating a job, starting it, and then collecting its results, will therefore involve multiple operations.

parameters
    The top-level labels of the input object for an operation are called the parameters of that application.

.. _data-types:

Data types
==========

.. note::

   This should be reconciled with the standard data types used elsewhere in IVOA standards.
   This is almost certainly missing additional data types that will be needed.
   In particular, it should probably include all of the DALI types.

   Handling of milliseconds in timestamps and durations is probably not correct.
   Some more attention to UTC vs. TAI is probably required.

   We may wish to allow null values in lists with an accompanying specification for what they mean.

The following data types may be used for object values in web service specifications.
A web service specification should indicate the data type of every value in an object.
A network protocol specification must describe how to serialize and deserialize these data types when sending them as part of a request or response.

Primitive data types
--------------------

boolean
    A value that accepts only two options, true or false.

duration
    A time duration.
    By default, the duration has second precision and a millisecond portion of all zeroes should not be interpreted as providing additional precision.
    Optionally, the web service specification may state that the milliseconds are significant.

integer
    An integer number with an optional sign.
    No exponent portion is permitted.
    By default, any integer between -9,007,199,254,740,991 and 9,007,199,254,740,991 (-2\ :sup:`53` + 1 and 2\ :sup:`53` - 1) is permitted.
    The web service specification should state the valid range if it is different than this.
    Values outside the default range may create encoding problems for some network protocols.

null
    The null value, indicating no value is present.
    If the value of a label within an object is null, this is equivalent to omitting the label and its value entirely.
    Web service specifications will generally not define values as having the null type and instead specify that the corresponding label is optional, which means that it may either have a valid value of its specified type or may be null.
    Network protocol specifications must specify how to serialize and deserialize null values.

real
    A real number.
    By default, any IEEE 754 binary64 (double precision) floating point number is supported.
    The web service specification should state any additional constraints.
    By default, the values positive infinity, negative infinity, and NaN are not permitted.
    The web service specification may explicitly allow them, but then must state their intended meanings in the context of the web service.

string
    A sequence of Unicode code points.
    By default, this sequence is allowed to be empty.
    If this is not permitted for a given value, the web service must specify this.

timestamp
    A specific point in UTC time using the Gregorian calendar.
    By default, the time has second precision and a millisecond portion of all zeroes should not be interpreted as providing additional precision.
    Optionally, the web service specification may state that the milliseconds are significant.
    Precision greater than milliseconds is not supported in timestamp fields and should be represented using some other data type (generally integer or float) following a specification specific to that web service.
    Dates prior to 1582-10-15 should not use this data type since they predate the Gregorian calendar.

Derived data types
------------------

enum
    A value that must be chosen from an enumerated list of possibilities specified in the web service specification.
    Each possibility must be a string.

uri
    A Uniform Resource Identifier as specified in :rfc:`3986`.
    If there are any constraints on its contents, such as a specific schema or structure, this must be specified by the web service.

Composite data types
--------------------

list
    A list of some other data type.
    The data type included in the list must be uniform.
    In other words, a list of strings, a list of floats, a list of objects of a specific type, or a list of lists of strings are all valid data types, but a list containing mixed integers and strings is not permitted.
    Lists may not contain null values.
    By default, a list may be empty.
    If it must be non-empty, the web service specification must specify this.
    The specification for the label for a value of type list must include both the singular and plural form, since different network encodings will use either the singular or the plural or both in different contexts.

object
    A mapping of labels to values.
    Each object must have a specification for its acceptable labels and values, and each such specified object is considered a separate data type.
    Labels themselves are of type string.
    Every value corresponding to a label must have a specified data type.
    A given label and its value may be optional, in which case the absence of the label and value or the presence of the label with a null value are equivalent.

Operations
==========

A web service supports one or more operations.
Each is initiated by a client request and results in a server response.
The request consists of an input object and possibly some additional information specified by the network encoding.
The response consists of a single response object or a data response.
(See :ref:`responses`.)

A web service specification should describe every operation of that web service.
Those descriptions must include the input object specification, including all of its labels and value data types; a description of the possible responses; and the semantics of the operation.

Operation types
---------------

Operations must be classified as one of the following, since each may be encoded differently by the network protocol:

query
    Requests some data but does not modify it.

create
    Creates some new object on the server.
    This can be a control object such as a job to run, or a piece of data that the server should store.

modify
    Modifies some existing object stored on the server.

delete
    Deletes some object stored on the server.

action
    Requests that the server perform some action that does not directly correspond to creating, modifying, deleting, or querying an object.

Operations of type query and delete may have an empty request body.
All other types of operations should have a non-empty request body in the service specification.
This avoids some security issues with HTTP-based network encodings.

Operation paths
---------------

Operations have an associated path.
This is a relative-URL string that specifies the URL for this operation relative to the base URL for the deployed service.
It is only applicable to network protocols that use URLs.

Multiple operations may have the same path.
In this case, they must have different operation types.
Operations of type action may not share the same path with any other operation.

.. _responses:

Responses
=========

Web service responses fall into two general buckets: a structured protocol response that uses the data types specified here, and a data response consisting of output in some other format.
The web service specification must say, for each operation, what type of response to expect.
This may vary based on the nature of the response; for example, success may produce a data response, but a failure may produce a structured error.

Data responses must be associated with a MIME type that describes the format of the response.
Web service specifications should list the MIME types of the possible data responses that are standardized, but implementations may also allow the client to request non-standard data response types.

The network protocol encoding must describe how to receive a data response and determine its MIME type, but need not describe how it is encoded if the network protocol conveys the normal byte stream representation of that MIME type in a way that is understood by a normal client of that network protocol.
For example, a network protocol specification built on top of HTTP need not specify how HTTP conveys ``application/fits`` data, only how to label that the response is ``application/fits``.

Content negotiation for data responses
--------------------------------------

Operations that respond with data may support multiple formats for that data.
Network protocols may provide a protocol mechanism for the client to request which format should be used for the data response.
For example, HTTP-based protocols may use the ``Accept`` HTTP header and corresponding content negotiation protocol to determine the requested format for the data response.

Not all network protocols provide a protocol-level way of negotiating the format, so service specifications may also provide a way to ask for a specific data format as part of the request.

.. _errors:

Errors
======

When a web service returns an error in a context where it can include an object in the response, that object should be a list of error objects conforming to the following specification.

.. note::

   We should say something about localization.
   The URI scheme here is based on some mailing list discussion but needs more fleshing out.

error (uri)
    A unique idenifier for the class of error.
    This may be any URI, but preferrably is an ``http`` or ``https`` URL that points to a description of this class of error and any additional information about it that may be useful to the user.
    URIs with scheme ``http`` or ``https`` and a host of ``ivoa.net`` are reserved for IVOA-standardized errors and should point to the definition of that error in the relevant IVOA standard.

description (string)
    A human-readable description of the error.

details (string, optional)
    Additional information about the error that may be helpful for debugging.
    For example, the server may include a backtrace or execution trace, log output, or other verbose information about the failure.

reference (list of uri, optional, plural: references)
    Additional ``http`` or ``https`` URLs that provide additional information about this error or class of error.
    A common use of this field is to provide additional local documentation for IVOA-standardized errors.

input (object, optional)
    Indicates that the error was caused by a specific input value.
    This is an object with one or two keys.

    field (string)
        The portion of the input that caused the error.
        The syntax of this string is specific to the network protocol used and must be specified by the network protocol.
        For example, for a JSON-based protocol, it may be a JSONPath expression, and for an XML-based protocol, it may be an XPath.

    value (any, optional)
        The specific value that caused the error.
        This will have whatever type the input value that caused the error had.
        In cases where the value was missing or is not parsable or representable in the network protocol, this label may be omitted.

An error reply body always contains a list of these objects, even if there is only one error.
This allows a web service to return multiple errors in the same response, such as when input data contains more than one validation error.

This structured error message may be used outside of explicit error responses.
For example, it may be an appropriate data type for an error field in an object that provides the results of some previous operation.
It is referred to as the data type ``error``.

.. _uws:

Universal Worker Service
========================

Many IVOA services perform operations that take longer than the typical timeout on a network protocol request and response.
The Universal Worker Service (UWS) pattern is a standardized way to write such a service.
All services implementing UWS use the same data model and operations to create and manage potentially long-running jobs and retrieve their results.

.. note::

   It may make more sense to spin this off as a separate document.
   The formal specification should include all of the other semantic detail from the UWS specification, such as the state model.

Data types
----------

phase (enum)
    The current execution phase of the job.
    One of the following values:

    - ``PENDING``
    - ``QUEUED``
    - ``EXECUTING``
    - ``COMPLETED``
    - ``ERROR``
    - ``ABORTED``
    - ``UNKNOWN``
    - ``HELD``
    - ``SUSPENDED``
    - ``ARCHIVED``

job (object)
    A representation of a UWS job.
    It has the following fields:

    jobId (string)
        Unique identifier of the job.

    owner (string, optional)
        Identity of the owner of the job, if the job is owned by a specific user.

    phase (phase)
        Execution phase of the job.

    runId (string, optional)
        The run ID provided by the client when creating the job, if any.

    creationTime (timestamp)
        When the job was created.

    startTime (timestamp, optional)
        When the job started executing, if it has.

    endTime (timestamp, optional)
        When the job finished executing, if it has.

    destructionTime (timestamp, optional)
        When the job will be destroyed and any resources allocated to it will be freed.
        When this time is reached, the job will be stopped if it is still running, all stored results and other information will be freed, and the service will forget that the job existed.

    executionDuration (duration, optional)
        How long the job is allowd to run.
        If this is set and the job runs for longer than this period, it will be aborted.

    quote (timestamp, optional)
        Estimated time of completion of the job if it were started now.
        If this time is later than ``destructionTime``, job execution is not possible due to resource constraints.

    parameters (object)
        The parameters used to create the job.
        The data type of this object is defined by the service specification.

    error (list of error, plural: errors, optional)
        If the job failed with an error, the error messages corresponding to the failure.
        This is a list so that all of the errors can be recorded for jobs that failed for multiple reasons.

    result (list of object, plural: results, optional)
        The results from the job, if it executed successfully.
        The labels in each object are:

        url (uri)
            URL pointing to the job result.
            This URL can be requested by the client (with ``GET``) to obtain the results of the job.

        size (integer, optional)
            The size of the results returned by ``uri``, if known.

        mimeType (string, optional)
            The MIME type of the results.
            This should match the ``Content-Type`` header returned by ``GET`` on ``uri``.

        error (boolean, optional)
            If present and set to true, indicates that this result is an error.
            The service specification specifies the type and content of the error.
            Service specifications are encouraged to use an encoding of the object defined in :ref:`errors`.

Operations
----------

The paths of these operations are relative to the base path of the UWS API, as defined by the service specification.
This may not be the same as the base path of the service's API.

Create job
^^^^^^^^^^

Create a new job.
By default, the job is created in the ``PENDING`` phase.

Path
    ``./``

Operation type
    create

Parameters
""""""""""

parameters (object)
    The parameters to the job.
    The data model for this parameter must be defined by the service specification.

executionDuration (duration, optional)
    Set the execution duration for the job.
    Jobs will be aborted if they run for longer than this duration.
    The server may override the value requested by the client, so the client must check the ``executionDuration`` of the returned job record to see what value was set.

destructionTime (timestamp, optional)
    The time at which the job and all of its results are deleted.
    The job is aborted if it is still running, and all record of the job is removed.
    The server may override the value requested by the client, so the client must check the ``executionDuration`` of the returned job record to see what value was set.

runId (string, optional)
    A run ID to associate with the job.
    The server stores this as an opaque string and returns it in job records.
    It may be used by the client to associate several jobs together or make it easier to find specific jobs later.

start (boolean, optional)
    If included and set to true, attempt to immediately start the job.
    By default, a newly-created job stays in ``PENDING`` phase until it is explicitly started.

Response
""""""""

On successful creation of the job, returns a response of type job, containing the metadata about the newly-created job.
This may be done via a redirect to the :ref:`Get job <uws-get-job>` operation.

List jobs
^^^^^^^^^

Retrieve a list of jobs.

Path
    ``jobs``

Operation type
    query

``GET`` may be used in the JSON network encoding.

Parameters
""""""""""

phase (list of phase, optional, plural: phases)
    Limit returned jobs to those in one of the given phases.

after (timestamp, optional)
    Limit returned jobs to those created after the given timestamp.

last (integer, optional)
    Limit returned jobs to the given number of records, prefering the most recent.
    This limit is applied after other filtering.
    For example, if both ``phase`` and ``after`` are given, along with ``limit=50``, this is a request for the 50 most recent jobs that satisfy the phase and creation timestamp constraints.

Response
""""""""

On success, returns a list of objects representing jobs, sorted by descending creation time.
Each object has the following labels.

job (uri)
    URL to a job record.
    This should be the URL of the :ref:`uws-get-job` API that retrieves the job corresponding to this object.

owner (string, optional)
    Identity of the owner of the job, if the job is owned by a specific user.

phase (phase)
    Execution phase of the job.

runId (string, optional)
    The run ID provided by the client when creating the job, if any.

creationTime (timestamp)
    When the job was created.

.. _uws-get-job:

Get job
^^^^^^^

Retrieve the details of a job.

Path
    ``jobs/{jobId}``

Operation type
    query

``GET`` may be used in the JSON network encoding.

Parameters
""""""""""

None.

Response
""""""""

If the job with the ``jobId`` specified in the path is found and the user is authorized, returns a response of type job containing the metadata about that job.

Start job
^^^^^^^^^

Requests that the server start the execution of the job.

Path
    ``jobs/{jobId}/start``

Operation type
    action

Parameters
""""""""""

start (boolean)
    Must be set to true.

Response
""""""""

If the job with ID ``jobId`` can be started successfully, returns a response of type job, containing the metadata for that job.
This may be done via a redirect to the :ref:`Get job <uws-get-job>` operation.

Wait for job
^^^^^^^^^^^^

Wait for the phase of a job to change.

Path
    ``jobs/{jobId}/wait``

Operation type
    query

``GET`` may be used in the JSON network encoding.

Parameters
""""""""""

phase (phase)
    The phase that the client thinks the job is in.
    The server will return as soon as the job phase is different from this phase, including when the phase of the job is already different when the request was received.

timeout (duration, optional)
    How long to wait for a phase transition.
    If not given, the server will impose a maximum timeout.
    This should be chosen by the server to be consistent with the native low-level timeout of the network protocol in use, if that imposes any constraints.

Response
""""""""

When the job with ID ``jobId`` has a phase different from the provided phase, or when the timeout expires, returns a response of type job, containing the metadata for that job.
The server will also return the job immediately if the phase is anything other than ``PENDING``, ``QUEUED``, ``EXECUTING``, ``HELD``, or ``SUSPENDED``, even if it matches the provided phase, since the other phases are terminal phases and will generally not change within the timeout of a query.
This may be done via a redirect to the :ref:`Get job <uws-get-job>` operation.

Modify job
^^^^^^^^^^

Modify the parameters of an existing job.

Path
    ``jobs/{jobId}``

Operation type
    modify

Delete job
^^^^^^^^^^

Deletes a job.
The job is stopped if it is currently running and removed from the API.
It is not specified whether the underlying data associated with the job is deleted or merely made inaccessible.
Deletion may not happen immediately.

Path
    ``jobs/{jobId}``

Operation type
    delete

Parameters
""""""""""

None.

Response
""""""""

On success, none apart from the status code indicating success.

Specification contents
======================

Network protocols
-----------------

Network protocol specifications must include all of the following:

#. An encoding for all of the data types listed in :ref:`data-types`.
   For object labels whose values are list data types, this must include how to use the singular and plural forms of the label.

#. A specification for how a client should send a request.

#. A specification for how a server should send a response.

#. A specification for how a web service protocol can include a schema of its encoding in this network protocol.
   For example, an XML network protocol specification may include instructions for how to provide XML schemas for request and response objects, and a JSON-based network protocol may include instructions for how to provide an OpenAPI schema.

Web services
------------

Web service specifications must include all of the following:

#. A list of all supported operations.
   For each operation, this must include the type of operation, the input object and its data types, and the nature of the response.
   If the response is a structured object, a specification of that object must be included.

#. For each supported network protocol encoding, the information that the network protocol says should be provided.
   This will usually include some form of schema for the combination of the web service and the network protocol.
   Requiring a schema is highly recommended, since schemas allow for code generation, service validation, and automatic generation of documentation.

If the web service uses the UWS pattern, it can refer to :ref:`uws` for its supported operations and need only specify the data type for job parameters, the semantics of the job, the possible results of the job, and any additional service operations that fall outside the UWS pattern.

To do
=====

The following things should be part of this draft specification but have not yet been written:

.. rst-class:: compact

- Address the various notes scattered through the document.
- Descriptions of errors common to many web services, in a form suitable for creating URLs that serve as error identifiers.
- Add the VOSI availability API
- Add the VOSI capabilities API, which requires writing a data model for registry entries or continuing to return the current XML data model.
- Add the DALI examples API.
