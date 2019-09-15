.. _clixon_standards:

Standards
=========

YANG
----

YANG and XML is the heart of Clixon.  Yang modules are used as a
specification for handling XML configuration data. The YANG spec is
used to generate an interactive CLI, netconf and restconf clients. It
also specifies the format of the XML datastore.

The YANG standards that Clixon follows include:

* `YANG 1.0 RFC 6020 <https://www.rfc-editor.org/rfc/rfc6020.txt>`_
* `YANG 1.1 RFC 7950 <https://www.rfc-editor.org/rfc/rfc7950.txt>`_
* `RFC 7895: YANG module library <http://www.rfc-base.org/txt/rfc-7895.txt>`_

However, the following YANG syntax modules are not implemented (reference to RFC7950 in parenthesis):

* deviation (7.20.3)
* action (7.15)
* augment in a uses sub-clause (7.17) (module-level augment is implemented)
* require-instance
* instance-identifier type (9.13)
* status (7.21.2)
* YIN (13)
* Yang extended Xpath functions: re-match(), deref)(), derived-from(), derived-from-or-self(), enum-value(), bit-is-set() (10.2-10.6)
* Default values on leaf-lists (7.7.2)
* Lists without keys (non-config lists may lack keys)

Regular expressions
^^^^^^^^^^^^^^^^^^^
Clixon supports two regular expressions engines:

`Posix`
   Posix is the default method, _translates_ XSD regexp:s to posix before matching with the standard Linux regex engine. This translation is not complete but can be considered "good-enough" for most yang use-cases. For reference, all standard `Yang models <https://github.com/YangModels/yang>`_ have been tested.
`Libxml2`
   Libxml2  uses the XSD regex engine. This is a complete XSD engine but you need to compile and link with libxml2 which may add overhead.

To use libxml2 in clixon you need enable libxml2 in both cligen and clixon:
::
   
  ./configure --with-libxml2 # both cligen and clixon

You then need to set the following configure option:
::

  <CLICON_YANG_REGEXP>libxml2</CLICON_YANG_REGEXP>


XML and XPATH
-------------
Clixon has its own implementation of XML and XPATH. See more in the detailed API reference.

The XML-related standards include:

* `XML 1.0 <https://www.w3.org/TR/2008/REC-xml-20081126>`_. (DOCTYPE/ DTD not supported)
* `Namespaces in XML 1.0 <https://www.w3.org/TR/2009/REC-xml-names-20091208>`_
* `XPATH 1.0 <https://www.w3.org/TR/xpath-10>`_
       
The following XPATH axes are supported:

* child,
* descendant,
* descendant_or_self,
* self
* parent

The following xpath axes are *not* supported: preceeding, preceeding_sibling, namespace, following_sibling, following, ancestor,ancestor_or_self, and attribute 


NETCONF
-------

Clixon implements the following NETCONF RFC:s:

* `RFC 6241: NETCONF Configuration Protocol <http://www.rfc-base.org/txt/rfc-6241.txt>`_
* `RFC 6242: Using the NETCONF Configuration Protocol over Secure Shell (SSH) <http://www.rfc-base.org/txt/rfc-6242.txt>`_
* `RFC 6243: With-defaults Capability for NETCONF <http://www.rfc-base.org/txt/rfc-6243.txt>`_. Clixon implements "explicit" default handling, but does not implement the RFC.
* `RFC 5277: NETCONF Event Notifications <http://www.rfc-base.org/txt/rfc-5277.txt>`_
* `RFC 8341: Network Configuration Access Control Model <http://www.rfc-base.org/txt/rfc-8341.txt>`_

The following RFC6241 capabilities/features are hardcoded in Clixon:

* :candidate (RFC6241 8.3)
* :validate (RFC6241 8.6)
* :xpath (RFC6241 8.9)
* :notification (RFC5277)

The following features are optional and can be enabled by setting CLICON_FEATURE:

* :startup (RFC6241 8.7)

Clixon does *not* support the following NETCONF features:

* :url capability
* copy-config source config
* edit-config testopts 
* edit-config erropts
* edit-config config-text
* edit-config operation

RESTCONF
--------

Clixon Restconf is a daemon based on FastCGI C-API. Instructions are available to
run with NGINX.
The implementatation is based on `RFC 8040: RESTCONF Protocol <https://tools.ietf.org/html/rfc8040>`_.

The following features of RFC8040 are supported:

* OPTIONS, HEAD, GET, POST, PUT, DELETE, PATCH
* stream notifications (Sec 6)
* query parameters: "insert", "point", "content", "depth", "start-time" and "stop-time".
* Monitoring (Sec 9)

The following features are not implemented:

* ETag/Last-Modified
* Query parameters: "fields", "filter", "with-defaults"

