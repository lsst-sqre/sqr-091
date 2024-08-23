##########################################
Draft IVOA web service standards framework
##########################################

.. abstract::

   Proposes a framework for defining IVOA web standards that separates the service specification from the protocol encoding and supports JSON encoding of web service calls.

.. warning::

   This document is **not an IVOA standard** and is **not complete**.
   It is an experiment intended as input to ongoing IVOA discussions about future revisions to IVOA standards.
   The document is written as if it were an IVOA standard to save effort if parts of it can be cut and pasted into future documents, but it is not endorsed by the IVOA and is not part of any IVOA protocol.

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

Data types
==========

.. note::

   This should be reconciled with the standard data types used elsewhere in IVOA standards.
   This is almost certainly missing additional data types that will be needed.
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

integer
    An integer number with an optional sign.
    No exponent portion is permitted.
    By default, any integer between -9,007,199,254,740,991 and 9,007,199,254,740,991 (2\ :sup:`53` + 1 and 2\ :sup:`53` - 1) is permitted.
    The web service specification should state the valid range if it is different than this.
    Values outside this default range may create encoding problems for some network protocols.

float
    A floating point number.
    By default, any IEEE 754 binary64 (double precision) floating point number is supported.
    The web service specification should state any additional constraints.
    By default, the values positive infinity, negative infinity, and NaN are not permitted.
    The web service specification may explicitly allow them, but then must state their intended meanings in the context of the web service.

timestamp
    A specific point in UTC time.
    By default, the time is in UTC.
    By default, the time has second precision and a millisecond portion of all zeroes should not be interpreted as providing additional precision.
    Optionally, the web service specification may state that the milliseconds are significant.
    Precision greater than milliseconds is not supported in timestamp fields and should be represented using some other data type (generally integer or float) following a specification specific to that web service.

duration
    A time duration.
    By default, the duration has second precision and a millisecond portion of all zeroes should not be interpreted as providing additional precision.
    Optionally, the web service specification may state that the milliseconds are significant.

list
    A list of some other data type.
    The data type included in the list must be uniform.
    In other words, a list of strings, a list of floats, a list of objects of a specific type, or a list of lists of strings are all valid data types, but a list containing mixed integers and strings is not permitted.
    Lists may not contain null values.
