=====
Forms
=====

In this section, a solution is proposed to a usability issue that is present
in a lot of RESTful APIs today. This issue is that, without referring to
external documentation, an API user does not know what data to provide to
operations that take input. For example, the POST call is used to create a new
resource in a collection, and an API user can find out with the OPTIONS call
that a certain collection supports POST. However, he does not know what data
is required to create the new resource. So far, most RESTful APIs require the
user to refer to the API documentation to get that information. This violates
an important RESTful principle that APIs should be self-descriptive.

As far as I can see, this usability issue impacts two important use cases:

1. A person is browsing the API, trying to use and learn it as he goes. The
   person can either be using a web browser, or a command-line tool like
   "curl".
2. The development of a command-line or graphical interface to a RESTful API.
   Without a proper description of required input types, knowledge around
   these has to be encoded explicitly in the client. This has the disadvantage
   that new features are not automatically exposed, and that there's strict
   requirements around not making backwards incompatible changes.

When we look at the web, this issue doesn't exist. Millions of users interact
with all kinds of IT systems over the web at any given moment, and none of
them has to resort to API documentation to understand how to do that.
Interactions with IT systems on the web are all self-explanatory using
hypertext, and input is guided by forms.

In fact, forms are what I see as the solution here. But before I go into that,
let's see look into two classes of solutions that have been proposed to
address this issue, and why I think they actually do not solve it.  The first
is using a type definition language like XMLSchema to describe the input
types, the second is using some kind of service description language like
WADL.

Type Definition
===============

As eluded to in :doc:`resources`, some RESTful APIs use a type definition
language to provide information about how to construct request entities. In my
view this is a bad approach, and has the following issues:

1. It creates a tight coupling between servers and clients.
2. It still does not allow a plain web browser as an HTTP client.
3. It does not help in the automatic construction of CLIs.

Often, XMLSchema is used as the type definition language. This brings in a few
more issues:

1. XMLSchema is a horribly complicated specification on one side, but on the
   other side it can't properly solve the very common requirement of an
   unordered set of key:value pairs (Note that <xs:all> does not solve this.
   Its limitations make it almost useless.).
2. Type definitions in XMLSchema are context-free. This means that you
   cannot just define one type, you need to define one type for PUT, one for
   POST, etc. And since one XML element can only have one type, this means
   you're PUTing a different element that you'd POST or GET. This breaks that
   "homogeneous collection" principle of REST.

Service Definition
==================

`WADL <http://www.w3.org/Submission/wadl/>`_ is an approach to define a
service description language for RESTful APIs. It tries to solve the problem
by meticulously defining entry points and parameters to those. The problem I
have with WADL, is that it feels very non-RESTful. I can't seen a lot of
difference between a method on a WADL resource, and an RPC entry point.

It would be possible to construct a more RESTful service description language.
It would focus on mapping the RESTful concepts of resource, collection,
relationships, links. It would probably need a type definition language as
well to define the constraints on types for each method (again, PUT may have
different constraints than POST). There have been discussions in the RHEV-M
project on this.

In the end though, this approach also feels wrong. What I think is needed is
something that works like the web, where everything is self descriptive, and
guide only by the actual URL flow the user is going through, not by reference
to some external description.

Using Forms to Guide Input
==========================

The right solution to guide client input in my view is to use forms. Note that
I'm using the word "forms" here in a generic sense, as an entity that guides
user input, and not necessarily equal to an HTML form.

In my view, forms need to specify three pieces of information:

1. How to contact the target and format the inpu.
2. A list of all available input fields.
3. A list of constraints which which the input fields must comply.

HTML forms provide for #1 and #2, but not for #3 above. #3 is typically
achieved using client-side Javascript. Because Javascript is not normally
available to clients that use the API, we need to define another way to do
validation. The approach I propose here is to define a form definition
language that captures the information #1 through #3. A form expressed in this
language can be sent over to the client as-is, in case the client understands
the form language. It can also be translated into an HTML form with Javascript
in case the client is a web browser.

Form Definition Language
------------------------

Our forms are defined as a special RESTful resource with type "form". This
means they use the JSON data model, can be represented in JSON, YAML and XML
according to the rules in :doc:`resources`.

Forms have their own content type.  This is listed in the table below:

====  ============================
Type          Content-Type
====  ============================
Form  | application/x-form+json
      | application/x-form+yaml
      | application/x-form+xml
====  ============================

Rather than providing a formal definition, I will introduce the form
definition language by using an example. Below is an example of a form that
guides the the creation of a virtual machine. In YAML format::

  !form
  method: POST
  action: {url}
  type: vm
  fields:
  - name: name
    type: string
    regex: [a-zA-Z0-9]{5,32}
  - name: description
    type: string
    maxlen: 128
  - name: memory
    type: number
    min: 512
    max: 8192
  - name: restart
    type: boolean
  - name: priority
    type: number
    min: 0
    max: 100
  constraints:
  - sense: mandatory
    field: name
  - sense: optional
    field: description
  - sense: optional
    field: cpu.cores
  - sense: optional
    field: cpu.sockets
  - sense: optional
    exclusive: true
    constraints:
    - sense: mandatory
      field: highlyavailable
    - sense: optional
      field: priority

As can be seen the form consists of 3 parts: form metadata, field definitions,
and constraints.

Form Metadata
-------------

The form metadata is very simple. The following attributes are defined:

=========  ============================================================
Attribute                           Description
=========  ============================================================
method     The HTTP method to use. Can be GET, POST, PUT or DELETE.
url        The URL to submit this form to.
type       The type of the resource to submit.
=========  ============================================================

Fields
------

The list of available fields are specified using the "fields" attribute. This
should be a list of field definitions. Each field definition has the following
attributes:

=========  ===============================================================
Attribute                           Description
=========  ===============================================================
name       The field name. Should be in dotted-name notation.
type       One of "string", "number" or "boolean"
min        Field value must be greater than or equal to this (numbers)
max        Field value must be less than or equal to this (numbers)
minlen     Minimum field length (strings)
maxlen     Maximum field length (strings)
regex      Field value needs to match this regular expression (strings)
multiple   Boolean that indicates if multiple values are accepted (array).
=========  ===============================================================

Constraints
-----------

First we need to answer the question what kind constraints do we want to
express in our form definition language. I will start by mentioning that in my
view, it is impossible to express each and every constraint client side. Some
constraints for example require access to other data (e.g. when creating
relationships), are computationally intensive, or even unknown to the API
designer because they are undocumented for the application the API is written
for. So in my view we need to find a good subset that is useful, without
making it too complicated and without worrying about the fact that some
constraints possibly cannot not be expressed.

This leads me to define the following two kinds of constraints, which in my
view are both useful, as well as sufficient for our purposes:

1. Constraints on individual data values.
2. Presence constraints on fields, i.e. whether a field is allowed or not
   allowed, and if allowed, whether it is mandatory.

The constraints on data values are useful because CLIs and GUIs could use this
information to help a user input data. For example, depending on the type, a
GUI could render a certain field a checkbox, a text box, or a dropdown list.
For brevity, constraints on individual data values are defined as part of the
field definition, and were discussed in the previous section.

Presence constraints are also useful, as they allow an API user to generate a
concise synopsis on how to call a certain operation to be used e.g. in CLIs.
Presence constraints specify which (combination of) input fields can and
cannot exist. Each presence constraint has the following attributes:

===========  ============================================================
Attribute                           Description
===========  ============================================================
sense        One of "mandatory" or "optional"
field        This constraint refers to a field.
constraints  This constraint is a group with nested contrainst.
exclusive    This is an exclusive group (groups only).
===========  ============================================================

Either "field" or "constraints" has to be specified, but not both. A
constraint that has the field attribute set is called a simple constraint. A
constraint that has the constraints attribute set, is called a group.

Checking Constraints
--------------------

Value constraints should be checked first, and should be checked only on
non-null values.

After value constraints, the presence constraints should to be checked. This
is a bit more complicated because the constraints are not only used for making
sure that all mandatory fields exist, but also that no non-optional fields are
present. The following algorithm should be used:

1. Start with an empty list called "referenced" that will collect all
   referenced fields.
2. Walk over all constraints in order. If a constraint is a group,  you need
   to recurse into it, depth first, passing it the "referenced" list.
3. For every simple constraint that validates, add the field name to
   "referenced".
4. In exclusive groups, matching stops at the first matching sub-constraint,
   in which case the group matches. In non-exclusive groups, matching stops at
   the first non-matching sub-constraint, in which case the group does not
   match.
5. When matching a group, you need to backtrack to the previous value of
   "referenced" in case the group does not match.
6. A constraint only fails if it is mandatory and it is a top-level
   constraint. If a constraint fails, processing may stop.
7. When you've walked through all constraints, it is an error if there are
   fields that have a non-null value but are not in the referenced list.

Building the Request Entity
---------------------------

After all constraints have been satisfied, a client should build a request
entity that it will pass in the body of the POST, PUT or DELETE method.  In
case the form was requested in JSON, YAML or XML format, it is assumed that
the client is not a web browser, and the following applies:

First, a new resource is created of the type specified in the form metadata.
Dotted field names should be interpreted as follows. Each dot creates a new
object, and stores it under the name immediately left of the dot in its parent
object. This means that the parent must be an object as well, which means it
cannot correspond to a field definition with "multiple" set to "true" (which
would make it a list). This resource is then represented in a format that the
server supports, using the rules described in :doc:`resources`.

If a client requested a "text/html" representation of the form, it is assumed
that the client is a web browser, and we assume the form will be processed as
a regular HTML form. In this case, the server should have generate an HTML
form with the following properties:

* An HTML <form> should be generated, with an appropriate <input> element for
  each field.
* The HTML form's "method" attribute should be set to POST, unconditionally. In
  case the RESTful form's method is not POST, the server should include a
  hidden input element with the name "_method" to indicate to the server the
  original method. (HTML does not support PUT or DELETE in a form).
* The form's "enctype" should be set to "multipart/form-data" or
  "application/x-www-form-urlencoded", as appropriate for the input elements.
* A hidden field called "_type" is generated that contains the value of the
  "type" attribute in the form metadata.
* The server may generate Javascript and include that in the HTML to check the
  value and presence constraints.

Linking to Forms
================

Forms are associated with resources and collections by link objects. The name
of the link object defines the meaning of the form. The following standard
forms names are defined:

============  ==========  =============================
   Name         Scope             Description
============  ==========  =============================
form/search   collection  Form to search for resources
form/create   collection  Form to create a new resource
form/update   resource    Form to update a resource
form/delete   resource    Form to delete a resource
============  ==========  =============================
