====================
Miscellaneous Topics
====================

Offline Help
============

A requirement that sometimes comes up is that to have an offline description
of the API that can be used to generate offline help for a command-line
interface (CLI) for that API. When connected to an API, all this information
is of course available because of the self-descriptive nature of a RESTful
API. But the request is to have that information offline as well, so that CLI
users can get help without have to connect first.

In my view this is a bad idea to offer this offline help. It introduces a
tight coupling between a client and a server, which will break the RESTful
model. A server cannot be independently upgraded anymore from a client, and
clients become tied to a specific server version. Also I doubt the usefulness
of this feature as having to connect first to get help does not seem like a
big issue to me.

That said, it is relatively straightforward to define a way in such an offline
description can be facilitated.

One change is required for this server-side. For each collection, the API
should implement a placeholder resource with a special ID (let's say "_", but
it doesn't matter as long as it cannot be a valid ID), and some fake data.
When a special HTTP Expect header is set, only this placeholder resource is
returned when querying a collection.

Having the placeholder resources in place for every collection, the following
procedure can be used client-side to retrieve all relevant metadata:

1. Retrieve the entry point of the API and store it in memory as a resource.
2. For every hyperlink in the entry point, fetch the target, and store it
   under the "target' property under the link object, recursively. When
   collections are retrieved, the special Expect header must be set.
3. For every hyperlink in the entry point, execute an OPTIONS call on the
   target, and store the resulting headers under the "options" property of the
   link object, recursively
4. Serialize the resulting object to a file.

The result of this is basically a recursive dump of the "GET"-able part of an
API with one placeholder resource in every collection. It will contain all
metadata including forms, options, available resource types and collections,
in a format that is very similar to the real on-line data. It can be used as
an off-line store to generate any help that you would be able to generate
online as well.

An API can store a version number on its entry point, so that clients can see
when there offline cached copy expired and can therefore generate a new one.
