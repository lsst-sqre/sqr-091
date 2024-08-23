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

   :sqr:`92`: Draft IVOA JSON encoding
       A network protocol encoding based on HTTP and JSON that follows this framework.

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

object
    A mapping of labels to values.
    Each object must have a specification for its acceptable labels and values, and each such specified object is considered a separate data type.
    Labels themselves are of type string (defined below).
    Every value corresponding to a label must have a specified data type.
    Labels may be optional, in which case the absence of the label and value or the presence of the label with a null value are equivalent.

null
    The null value, indicating no value is present.
    If the value of a label within an object is null, this is equivalent to omitting the label and its value entirely.
    Web service specifications will generally not define values as having the null type and instead specify that the corresponding label is optional, which means that it may either have a valid value of its specified type or may be null.
    Network protocol specifications must specify how to serialize and deserialize null values.

string
    A sequence of Unicode code points.
    By default, this sequence is allowed to be empty.
    If this is not permitted for a given value, the web service must specify this.

uri
    A Uniform Resource Identifier as specified in :rfc:`3986`.
    If there are any constraints on its contents, such as a specific schema or structure, this must be specified by the web service.

integer
    An integer number with an optional sign.
    No exponent portion is permitted.
    By default, any integer between -9,007,199,254,740,991 and 9,007,199,254,740,991 (2\ :sup:`53` + 1 and 2\ :sup:`53` - 1) is permitted.
    The web service specification should state the valid range if it is different than this.
    Values outside the default range may create encoding problems for some network protocols.

float
    A floating point number.
    By default, any IEEE 754 binary64 (double precision) floating point number is supported.
    The web service specification should state any additional constraints.
    By default, the values positive infinity, negative infinity, and NaN are not permitted.
    The web service specification may explicitly allow them, but then must state their intended meanings in the context of the web service.

boolean
    A value that accepts only two options, true or false.

enum
    A value that must be chosen from an enumerated list of possibilities specified in the web service specification.
    Each possibility must be a string.

timestamp
    A specific point in UTC time using the Gregorian calendar.
    By default, the time has second precision and a millisecond portion of all zeroes should not be interpreted as providing additional precision.
    Optionally, the web service specification may state that the milliseconds are significant.
    Precision greater than milliseconds is not supported in timestamp fields and should be represented using some other data type (generally integer or float) following a specification specific to that web service.
    Dates prior to 1582-10-15 should not use this data type since they predate the Gregorian calendar.

duration
    A time duration.
    By default, the duration has second precision and a millisecond portion of all zeroes should not be interpreted as providing additional precision.
    Optionally, the web service specification may state that the milliseconds are significant.

list
    A list of some other data type.
    The data type included in the list must be uniform.
    In other words, a list of strings, a list of floats, a list of objects of a specific type, or a list of lists of strings are all valid data types, but a list containing mixed integers and strings is not permitted.
    Lists may not contain null values.
    By default, a list may be empty.
    If it must be non-empty, the web service specification must specify this.

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

references (list of uri, optional)
    Additional ``http`` or ``https`` URLs that provide additional information about this error or class of error.
    A common use of this field is to provide additional local documentation for IVOA-standardized errors.

input (object, optional)
    Indicates that the error was caused by a specific input value.
    This is an object with one or two keys.

    field (string)
        The portion of the input that caused the error.
        The syntax of this string is specific to the network protocol used and must be specified by the network protocol.
        For example, for a JSON-based protocol, it may be a JSONPath expression, and for an XML-based protocol, it may be an XPath.

    value (optional)
        The specific value that caused the error.
        This will have whatever type the input value that caused the error had.
        In cases where the value was missing or is not parsable or representable in the network protocol, this label may be omitted.

An error reply body always contains a list of these objects, even if there is only one error.
This allows a web service to return multiple errors in the same response, such as when input data contains more than one validation error.

This structured error message may be used outside of explicit error responses.
For example, it may be an appropriate data type for an error field in an object that provides the results of some previous operation.

Specification contents
======================

Network protocols
-----------------

Network protocol specifications must include all of the following:

#. An encoding for all of the data types listed in :ref:`data-types`.

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

To do
=====

The following things should be part of this specification but have not yet been written:

.. rst-class:: compact

- Address the various notes scattered through the document.
- A specification for the Universal Worker Service API.
- Descriptions of errors common to many web services, in a form suitable for creating URLs that serve as error identifiers.
