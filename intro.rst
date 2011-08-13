============
Introduction
============

This essay is an attempt to put down my thoughts on how to design a
real-world yet beautiful RESTful API. It draws from the experience I have
gained being involved in the design of the `RESTful API
<http://bitbucket.org/geertj/rhevm-api/wiki/Home>`_ for Red Hat's Enterprise
Virtualization product, `twice <http://fedorahosted.org/rhevm-api/>`_.  During
the design phase of the API we had to solve many of the real-world problems
described above, but we weren't willing add non-RESTful or "RPC-like"
interfaces to our API too easily.

In my definition, a real-world RESTful API is an API that provides answers to
questions that you won't find in introductory texts, but that inevitably
surface in the real world, such as whether or not resources should be
described formally, how to create useful and automatic command-line
interfaces, how to do polling, asynchronous and other non-standard types of
requests, and how to deal with operations that have no good RESTful mapping.

A beautiful RESTful API on the other hand is one that does not deviate from
the principles of RESTful architecture style too easily. One important design
element for example that is not always addressed is the possibility for
complete auto-discovery by the API user, so that the API can be used by a
human with a web browser, without any reference to external documentation. I
will discuss this issue in detail in :doc:`forms`.

This essay does not attempt to explain the basics about REST, as there are
many other texts about that. For a good description of REST, I would recommend
reading at least `the Atom Publishing Protocol
<http://tools.ietf.org/html/rfc5023>`_ and `this blog post
<http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven>`_ by
Roy Fielding.

While the views exposed in this essay are my own, they are heavily based on
the public discussion on the `rhevm-api mailing list
<https://fedorahosted.org/mailman/listinfo/rhevm-api>`_.  My thanks goes to
all people who have contributed and are still contributing to it, including
Mark McLoughlin, Michael Pasternak, Ori Liel, Sander Hoentjen, Ewoud Kohl van
Wijngaarden, Tomas V.V. Cox and everybody else I forgot.
