=======
Methods
=======

Standard Methods
================

We already discussed that resources are the fundamental concept in a RESTful
API, and that each resource has its own unique URL. Methods can be executed on
resources via their URL.

The table below lists the standard methods that have a well-defind meaning for
all resources and collections.

=======  ==========  ==================================================
Method     Scope                         Semantics
=======  ==========  ==================================================
GET      collection  Retrieve all resources in a collection
GET      resource    Retrieve a single resource
HEAD     collection  Return all resources in a collection (header only)
HEAD     resource    Retrieve a single resource (header only)
POST     collection  Create a new resource in a collection
PUT      resource    Update a resource
DELETE   resource    Delete a resource
OPTIONS  any         Return available HTTP methods and other options
=======  ==========  ==================================================

Normally, not all resources and collections implement all methods. There are
two ways to find out which methods are accepted by a resource or collection.

1. Use the OPTIONS method on the URL, and look at the "Allow" header that is
   returned. This header contains a comma-separated list of methods are are
   supported for the resource or collection.
2. Just issue the method you want to issue, but be prepared for a "405 Method
   Not Allowed" response that indicates the method is not accepted for this
   resource or collection.

Actions
=======

Sometimes, it is required to expose an operation in the API that inherently is
non RESTful. One example of such an operation is where you want to introduce a
state change for a resource, but there are multiple ways in which the same
final state can be achieved, and those ways actually differ in a significant
but non-observable side-effect. Some may say such transitions are bad API
design, but not having to model all state can greatly simplify an API. A great
example of this is the difference between a "power off" and a "shutdown" of a
virtual machine. Both will lead to a vm resource in the "DOWN" state.
However, these operations are quite different.

As a solutions to such non-RESTful operations, an "actions" sub-collection can
be used on a resource. Actions are basically RPC-like messages to a resource
to perform a certain operation. The "actions" sub-collection can be seen as a
command queue to which new action can be POSTed, that are then executed by the
API. Each action resource that is POSTed, should have a "type" attribute that
indicates the type of action to be performed, and can have arbitrary other
attributes that parametrize the operation.

It should be noted that actions should only be used as an exception, when
there's a good reason that an operation cannot be mapped to one of the
standard RESTful methods. If an API has too many actions, then that's an
indication that either it was designed with an RPC viewpoint rather than using
RESTful principles, or that the API in question is naturally a better fit for
an RPC type model.

PATCH vs PUT
============

In the HTTP RFC, PUT is defined to take a full new resource representation as
the request entity. If only certain attributes are provided, it means that
non-provided attributes should be removed (i.e. set to null).

An alternative method called PATCH `has been proposed recently
<http://tools.ietf.org/html/rfc5789>`_. The semantics of this call are that
like PUT it updates a resource, but unlike PUT, it applies a delta rather than
replacing the resource with the new representation. At the time of writing,
the PATCH was still a proposed standard waiting final approval.

Many current RESTful APIs use PUT but implement the PATCH semantics. Since
this behavior seems wide spread, and since requiring PUT to accept a full
representation is adds a high client overhread, my recommendation would be to
implement PUT as PATCH for now, until PATCH becomes widespread, at which point
it can replace PUT and PUT can get its original meaning.

Link Headers
============

An `Internet draft
<http://tools.ietf.org/html/draft-nottingham-http-link-header-10>`_ exists
defining the "Link:" header type.  This header takes the "link" attributes of
the response entity, and formats them as an HTTP header. It is argued that the
Link header is useful because it allows a client to quickly get links from a
response without having to parse the response entity, or even without
retrieving the response entity at all using the HEAD HTTP method.

In my view, the usefulness of this feature is dubious. First of all, it can
increase response sizes quite significantly. Second, it can only be used when
a resource is being returned, it does not make sense to be used with
collections. Because I have not yet seen any good use of this header in the
context of a RESTful API, I recommend not to implement Link headers.

Asynchronous Requests
=====================

Sometimes an action takes too long to be completed in the context of a single
HTTP request. In that case, a "202 Accepted" status can be returned to the
client. Such a response should only be returned for POST, PUT or DELETE.

The response entity of a 202 Accepted response should be a regular resource
with only the information filled in that was available at the time the request
was accepted. The resource should contain a "link" attribute that points to a
status monitor that can be polled to get updated status information.

When polling the status monitor, it should return a "response" object with
information on the current status of the asynchronous request. If the request
is still in progress, such a response could look like this (in YAML)::

  !response
  status: 202 Accepted
  progress: 50%

If the call has finished, the response should include the same headers and
response body had the request been fulfilled synchronously::

  !response
  status: 201 Created
  headers:
   - name: content-type
     value: applicaton/x-resource+yaml
  response: !!str
    Response goes here

After the response has been retrieved once with a status that is not equal to
"202 Accepted", the API code may garbage collect it and therefore clients
should not assume it will continue to be available.

A client may request the server to modify its asynchronous behavior with the
following "Except" headers:

* "Expect: 200-ok/201-created/204-no-content" disables all asynchronous
  functionality. The server may return a "417 Expectation Failed" 
  if it is not willing to wait for an operation to complete.
* "Expect: 202-accepted" explicitly request an asynchronous response. The
  server may return a "417 Expectation Failed" if it is not willing to perform
  the request asynchronously.

If no expectation is provided, client must be prepared to accept a 202
Accepted status for any request other than GET.

Ranges / Pagination
===================

In case collections contain a lot of resource, it is quite a common
requirement for a client to retrieve only a subset of the available resources.
This can be implemented using the Range header with a "resource" range unit:

.. code-block:: none

  GET /api/collection
  Range: resources=100-199

The above example would return resources 100 through 199 (inclusive).

Note that it is the responsibility of the API implementer to ensure a proper
and preferably meaningful ordering can be guaranteed for the resources.

Servers should provide an "Accept-Ranges: resource" header to indicate to a
client that they support resource-based range queries. This header should be
provided in an OPTIONS response:

.. code-block:: none

  OPTIONS /api/collection HTTP/1.1

  HTTP/1.1 200 OK
  Accept-Ranges: resources

Notifications
=============

Another common requirement is where a client wants to be notified immediately
when some kind of event happens.

Ideally, such a notification would be implemented using a call-out from the
server to the client. However, there is no good portable standard to do this
over HTTP, and it also breaks with network address translation, and HTTP
proxies.  A second approach called busy-loop polling is horribly inefficient.

In my view, the best approach is what is is called "long polling". In long
polling, the client will retrieve a URL but the server will not generate a
response yet. The client will wait for a configurable amount of time, until it
will close the connection and reconnect. If the server becomes aware of an
event that requires notification of clients, it can provide that event
immediately to clients that are currently waiting.

Long polling should be disabled by default, and can be enabled by a client
using an Expect header. For example, a client could long poll for new
resources in a collection using a combination of long-polling and a
resource-based range query:

.. code-block:: none

  GET /api/collection
  Range: 100-
  Expect: nonempty-response

In this case, resource "100" would be the last resource that was read, and the
call is requesting the API to return at least one resource with an ID > 100.

Server implementers need to decide whether they want to implement long polling
using one thread per waiting client, or one thread that uses multiplexed IO to
wait for all clients. This is a trade-off to be made between ease of
implementation and scalability (that said, threads are pretty cheap on modern
operating systems).
