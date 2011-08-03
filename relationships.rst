=============
Relationships
=============

As we have seen in :doc:`resources`, the resource is the fundamental unit in
RESTful API design. Resources model an object from the application data model.

Resources do not exist in isolation, but have relationships to other other
resources. Sometimes these relationships exist between the mapped objects in
the application data model as well, sometimes they are specific to the RESTful
resources.

One of the fundaments of the RESTful architecture style, is these
relationships are expressed by hyperlinks on the representation of a resource.

In our resource model, we interpret any object with an "href" attribute as a
hyperlink. The value of the href attribute contains an absolute URL that can
be retrieved with GET. Using GET on a such a URL get is guaranteed to be
side-effect free.

Two types of hyperlinks are normally used:

1. Using "link" objects. This is a special object of type "link" with
   "href" and a "rel" attributes. The "rel" attribute value identifies the
   semantics of the relationship. The link object is borrowed from
   `HTML <http://www.w3.org/TR/html4/struct/links.html>`_.
2. Using any object with a "href" attribute. The object type defines the
   semantics of the relationship. I'll call these "object links," not to be
   confused with the "link objects" above.

Below is an example of a virtual machine representation in YAML, that
illustrates the two different ways in which a relationship can be expressed::

  !vm
  id: 123
  href: /api/vms/123
  link:
    rel: collection/nics
    href: /api/vms/123/nics
  cluster:
    href: /api/clusters/456

The "link" object is used with rel="collection/nics" to refer to a
sub-collection to the VM holding the virtual network interfaces. The cluster
link instead points to the cluster containing this VM by means of a "cluster"
object.

Note that the VM itself has a href attribute as well. This is called a "self"
link, and it is a convention for a resource to provide such a link. That way,
the object can later be easily re-retrieved or updated without keeping
external state.

There are no absolute rules when to use link objects, and when not. That said,
we have been using the rules below in the rhevm-api project with good success.
They seem to be a good trade-off between compactness (favoring object links)
and the ability to understand links without understanding the specific
resource model (favoring link objects).

* Link objects are used to express structural relationships in the API. So for
  example, the top-level collections, singleton resources and sub-collections
  (including actions) are all referenced using link objects.
* Objects links are used to express semantical relationships from the
  application data model. In the example above, the vm to cluster link comes
  directly from the application data model and is therefore modeled as a link.

Note however that sub-collections can express both a semantical relationship,
as well as a structural relationship. See for example the "collection/nics"
sub-collection to a VM in the example above. In such a case, our convention
has been to use link objects.


Standard Structural Relationships
---------------------------------

It makes sense to define a few standard structural relationship names that can
be relevant to many different APIs. In this way, client authors know what to
expect and can write to some extent portable clients that work with multiple
APIs. The table below contains a few of these relationship names.

=================  =====================================
Relationship                     Semantics
=================  =====================================
collection/{name}  Link to a related collection {name}.
resource/{name}    Link to a related resource {name}.
form/{name}        Link to a related form {name}.
=================  =====================================

Here, {name} is replaced with a term that has meaning for the specific API.
For example, a collection of virtual machines could be linked from the entry
point of a virtualization API using rel="collection/vms".


Modeling Semantic Relationships
-------------------------------

Semantical relationships can be modeled either by an object link, or by a
sub-collection. I believe that choosing the right way to represent a
sub-collection is important to get a consistent API experience for the user.
I would advocate using the following rules:

1. In case of a 1:N relationship, where the target object is **existentially
   dependent** on the source object, I'd recommend to use a sub-collection.
   With existentially dependent I mean that a target object cannot exist
   without its source. In database parlance, this would be a FOREIGN KEY
   relationship with an ON DELETE CASCADE. An example of such a relationship
   are the NICs that are associated with a VM
2. In case of a 1:N relationship, where there is data that is associated with
   the link, I'd recommend to use a sub-collection. Note that we are talking
   about data here that is neither part of the source object, or the target
   object.  The resources in the sub-collection can hold the extra data. In
   this case, the data in the sub-resource would therefore be a merge of the
   data from the mapped object from the application data model, and link data.
3. In case of any other 1:N relationship, I'd recommend to use an object link.
4. In case of a N:M relationship, I'd recommend to use a sub-collection.
   The sub-collection can be defined initially to support the lookup direction
   that you most often need to follow. If you need to query both sides of the
   relationship often, then two sub-collections can be defined, one on each
   linked object.
