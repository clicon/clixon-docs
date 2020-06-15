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
* `RFC 7895: YANG module library <http://www.rfc-editor.org/rfc/rfc7895.txt>`_

However, the following YANG syntax modules are not implemented (reference to RFC7950 in parenthesis):

* deviation (7.20.3)
* action (7.15)
* augment in a uses sub-clause (7.17) (module-level augment is implemented)
* require-instance
* instance-identifier type (9.13)
* status (7.21.2)
* YIN (13)
* Yang extended XPath functions: re-match(), deref)(), derived-from(), derived-from-or-self(), enum-value(), bit-is-set() (10.2-10.6)
* Default values on leaf-lists (7.7.2)
* Lists without keys (non-config lists may lack keys)

Regular expressions
^^^^^^^^^^^^^^^^^^^
Clixon supports two regular expressions engines:

`Posix`
   Posix is the default method, The regexps:s are translated to posix before matching with the standard Linux regex engine. This translation is not complete but can be considered "good-enough" for most yang use-cases. For reference, all standard `Yang models <https://github.com/YangModels/yang>`_ have been tested.
`Libxml2`
   Libxml2  uses the XSD regex engine. This is a complete XSD engine but you need to compile and link with libxml2 which may add overhead.

To use libxml2 in clixon you need enable libxml2 in both cligen and clixon:
::
   
  ./configure --with-libxml2 # both cligen and clixon

You then need to set the following configure option:
::

  <CLICON_YANG_REGEXP>libxml2</CLICON_YANG_REGEXP>


XML and XPath
-------------
Clixon has its own implementation of XML and XPath. See more in the detailed API reference.

The XML-related standards include:

* `XML 1.0 <https://www.w3.org/TR/2008/REC-xml-20081126>`_. (DOCTYPE/ DTD not supported)
* `Namespaces in XML 1.0 <https://www.w3.org/TR/2009/REC-xml-names-20091208>`_
* `XPath 1.0 <https://www.w3.org/TR/xpath-10>`_
       
The following XPath axes are supported:

* child,
* descendant,
* descendant_or_self,
* self
* parent

The following xpath axes are *not* supported: preceeding, preceeding_sibling, namespace, following_sibling, following, ancestor,ancestor_or_self, and attribute 


Unicode
-------
Unicode is not supported in YANG and XML

NETCONF
-------

Clixon implements the following NETCONF RFC:s:

* `RFC 6241: NETCONF Configuration Protocol <http://www.rfc-editor.org/rfc/rfc6241.txt>`_
* `RFC 6242: Using the NETCONF Configuration Protocol over Secure Shell (SSH) <http://www.rfc-editor.org/rfc/rfc6242.txt>`_
* `RFC 5277: NETCONF Event Notifications <http://www.rfc-editor.org/rfc/rfc5277.txt>`_
* `RFC 8341: Network Configuration Access Control Model <http://www.rfc-editor.org/rfc/rfc8341.txt>`_. Except notification.

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

Further, in `get-config` filter expressions, the RFC6241 XPath
Capability is preferred over default subtrees. This has two reasons:

1. XPath has better performance since the underlying system uses xpath, and subtree filtering is done after the complete tree is retreived.
2. Subtree filtering does not support namespaces yet.

Default values
^^^^^^^^^^^^^^

Clixon only stores explicit set default values in datastores, while unset values are populated in memory on retreival. This means that get-config will report all default values, not only those explicitly set. 

`RFC 6243: With-defaults Capability for NETCONF <http://www.rfc-editor.org/rfc/rfc6243.txt>`_ is not implemented. Among the modes descriibed in the RFC, Clixon implements "report-all" with-respect to GET and GET-CONFIG operations, but "explicit" with reespect to how configurations are saved in datastores.

   
RESTCONF
--------

Clixon Restconf is a daemon based on FastCGI C-API. Instructions are available to
run with NGINX.
The implementatation is based on `RFC 8040: RESTCONF Protocol <https://www.rfc-editor.org/rfc/rfc8040.txt>`_.

The following features of RFC8040 are supported:

* OPTIONS, HEAD, GET, POST, PUT, DELETE, PATCH
* stream notifications (Sec 6)
* query parameters: "insert", "point", "content", "depth", "start-time" and "stop-time".
* Monitoring (Sec 9)

The following features are not implemented:

* ETag/Last-Modified
* Query parameters: "fields", "filter", "with-defaults"

JSON
----

Clixon implements JSON according to  `ECMA JSON Data Interchange Syntax <http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf>`_ and  `RFC 7951 JSON Encoding of Data Modeled with YANG <https://www.rfc-editor.org/rfc/rfc8040.txt>`_.
