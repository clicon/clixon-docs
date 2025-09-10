.. _clixon_standards:
.. sectnum::
   :start: 4
   :depth: 3

*********
Standards
*********

YANG
====
YANG and XML are central to Clixon.  Yang modules are used as a
specification for encoding XML or JSON configuration and state
data. The YANG spec is also used to generate an interactive CLI,
NETCONF and RESTCONF clients, as well as the format of the XML
datastore.

The YANG standards that Clixon follows include (see also `netconf`_):

* `YANG 1.0 RFC 6020 <https://www.rfc-editor.org/rfc/rfc6020.txt>`_
* `YANG 1.1 RFC 7950 <https://www.rfc-editor.org/rfc/rfc7950.txt>`_
* `YANG library RFC 8525 <http://www.rfc-editor.org/rfc/rfc8525.txt>`_ (partly)

Clixon deviates from the YANG standard as follows (reference to RFC7950 sections in parenthesis):

Not implemented:

* instance-identifier type (9.13)
* status (7.21.2)
* YIN (13)
* Default values on leaf-lists (7.7.2)
* error-message is not implemented as sub-statement of "range", "length" and "pattern"
* quoted string concatenation using + for other than the "string" non.-terminal (eg identifer-args)

Clixon supports the following extended XPath functions (10):

   - current()
   - re-match()
   - deref()
   - derived-from(),
   - derived-from-or-self()
   - bit-is-set()

The following extended XPath function is *not* supported (10):

   - enum-value()

See also support of standard XPath functions `XML and XPath`_

Regular expressions
-------------------
Clixon supports two regular expression engines:

`Posix`
   The default method, The regexps:s are translated to posix before matching with the standard Linux regex engine. This translation is not complete but can be considered "good-enough" for most yang use-cases. For reference, all standard `Yang models <https://github.com/YangModels/yang>`_ have been tested.
`Libxml2`
   Libxml2  uses the XSD regex engine. This is a complete XSD engine but you need to compile and link with libxml2 which may add overhead.

To use libxml2 in clixon you need enable libxml2 in both cligen and clixon::

  ./configure --with-libxml2 # both cligen and clixon

You then need to set the following configure option::

  <CLICON_YANG_REGEXP>libxml2</CLICON_YANG_REGEXP>

Metadata
--------
Clixon implements `Defining and Using Metadata with YANG RFC 7952 <http://www.rfc-editor.org/rfc/rfc7952.txt>`_ for XML and JSON.

This means that Yang-derived meta-data defined with::

    md:annotation <name>

is defined for attributes so that they can be mapped from XML to JSON, for example.

Assigned meta-data are hardcoded. The following attributes are defined:

* ietf-netconf-with-defaults:default from RFC 6243 / RFC 8040

Schema mount
------------
Yang schema mount is supported as defined in: `RFC 8528: YANG Schema Mount <http://www.rfc-editor.org/rfc/rfc8528.txt>`_ .

Enable by the `CLICON_YANG_SCHEMA_MOUNT` configuration option.

NETCONF
=======
Clixon implements the following NETCONF RFC:s:

* `RFC 5277: NETCONF Event Notifications <http://www.rfc-editor.org/rfc/rfc5277.txt>`_
* `RFC 6022: YANG Module for NETCONF Monitoring <http://www.rfc-editor.org/rfc/rfc6022.txt>`_.
* `RFC 6241: NETCONF Configuration Protocol <http://www.rfc-editor.org/rfc/rfc6241.txt>`_
* `RFC 6242: Using the NETCONF Configuration Protocol over Secure Shell (SSH) <http://www.rfc-editor.org/rfc/rfc6242.txt>`_
* `RFC 6243 With-defaults Capability for NETCONF <http:www.rfc-editor.org/rfc/rfc6243.txt>`_
* `RFC 8071: NETCONF Call Home and RESTCONF Call Home <http://www.rfc-editor.org/rfc/rfc8071.txt>`_. NETCONF over SSH (external) and RESTCONF call home (internal) over TLS are implemented.
* `RFC 8341: Network Configuration Access Control Model <http://www.rfc-editor.org/rfc/rfc8341.txt>`_ (NACM). Notification not implemented.
* `RFC 9144: Comparison of Network Management Datastore Architecture (NMDA) Datastores <http://www.rfc-editor.org/rfc/rfc9144.txt>`_. all, report-origin, subtree-filter not implemented

The following RFC6241 capabilities/features are hardcoded in Clixon:

* :candidate (RFC6241 8.3)
* :validate (RFC6241 8.6)
* :xpath (RFC6241 8.9)
* :notification (RFC5277)
* :with-defaults (RFC6243)

The following features are optional and can be enabled by setting CLICON_FEATURE:

* :confirmed-commit:1.1 (RFC6241 8.4)
* :startup (RFC6241 8.7)
* :writable-running (RFC6241 8.2) - just write to running, no commit semantics

Clixon does *not* support the following NETCONF features:

* :url capability
* copy-config source config
* edit-config testopts
* edit-config erropts
* edit-config config-text
* edit-config operation

Further, in `get-config` filter expressions, the RFC6241 XPath
Capability is preferred over default subtrees. This has two reasons:

1. XPath has better performance since the underlying system uses xpath, and subtree filtering is done after the complete tree is retrieved.
2. Subtree filtering does not support namespaces.

Clixon supports netconf locks in default settings.

RFC 6022 YANG Module for NETCONF Monitoring
-------------------------------------------
Clixon extends the RFC 6022 session parameter ``transport`` with "cli", "restconf", "netconf" and "snmp".  In particular, the ``clixon_netconf`` application uses stdio to get input and print output and is used in a "piping" fashion, for example directly in a terminal shell or as a part of a SSH sub-system, and therefore has no direct knowledge of whether the NETCONF transport is over SSH or not.

The ``source-host`` parameter is set only in certain
circumstances when the source host is in fact known. This includes native RESTCONF for example.

Further, ``hello`` counters are backend based, ie the internal
protocol, which means hellos from RESTCONF, SNMP and CLI clients are
included and that eventual dropped hello messages from external NETCONF sessions are not.

Default handling
----------------
Clixon treats default data according to what is defined as `explicit basic mode` in `RFC 6243: With-defaults Capability for NETCONF <http://www.rfc-editor.org/rfc/rfc6243.txt>`_, i.e. the server considers any data node that is not explicitly set data to be default data.

One effect is that if you view the contents of datastores (or import/export them), they should be in `explicit basic mode`.

The `:with-defaults` capability indicates that clixon default behaviour is explicit and also indicates that additional retrieval modes supported by the server are:.

* explicit
* trim
* report-all
* report-all-tagged

Internally in memory, however, `report-all` is used.

Private candidate
-----------------
Clixon implements private candidate as defined in `NETCONF and RESTCONF Private Candidate Datastores <https://datatracker.ietf.org/doc/html/draft-ietf-netconf-privcand-07>`_ with the following restrictions:

* ``revert-on-conflict`` is the only resolution mode supported
* No augments on `compare` operation
* Leaf-list conflict resolution fine-grained as opposed to draft
* Delete candidate on commit (draft is unclear)
* ``<delete-config>`` is not possible on candidate configuration (draft vs RFC6241)

RESTCONF
========
Clixon supports the two RESTCONF compile-time variants: *FCGI* and *Native*. Both implements `RFC 8040: RESTCONF Protocol <https://www.rfc-editor.org/rfc/rfc8040.txt>`_.

The following features of RFC8040 are supported:

* OPTIONS, HEAD, GET, POST, PUT, DELETE, PATCH
* Stream notifications (Sec 6)
* Query parameters: `insert`, `point`, `content`, `depth`, `start-time`, `stop-time` and `with-defaults`.
* Monitoring (Sec 9)

The following features are *not* implemented:

* ETag/Last-Modified
* Query parameters: `fields` and `filter`

RESTCONF event notification as described in RFC7950 section 6 is supported as follows:

* Limited to regular subscription, start-time and stop-time

`NMDA` is partly supported according to `RFC 8324 <https://tools.ietf.org/html/rfc8342>`_ and `RFC 8527 <https://tools.ietf.org/html/rfc8527>`_. With-defaults and with-origin are not implemented.

`RFC 8072: YANG Patch Media Type <https://www.rfc-editor.org/rfc/rfc8072.txt>`_ is not implemented.

In the native mode, Clixon also supports:

* HTTP/1.1 as transport using a native implementation (RFC 7230),
* HTTP/2 as transport implemented by libnghttp2 (RFC7540),
* Transport Layer Security (TLS) implemented by libopenssl, versions 1.1.1 and 3.0
* ALPN as defined in RFC 7301 for http/1, http/2 protocol selection by libopenssl

SNMP
====
The Clixon-SNMP frontend implements the MIB-YANG mapping as defined in RFC 6643.

XML and XPath
=============
Clixon has its own implementation of XML and XPath. See more in the detailed API reference.

The XML-related standards include:

* `XML 1.0 <https://www.w3.org/TR/2008/REC-xml-20081126>`_. (DOCTYPE/ DTD not supported)
* `Namespaces in XML 1.0 <https://www.w3.org/TR/2009/REC-xml-names-20091208>`_
* `XPath 1.0 <https://www.w3.org/TR/xpath-10>`_

Clixon XML supports version and UTF-8 only.

The following XPath axes are supported:

* child,
* descendant,
* descendant-or-self,
* self
* parent

The following xpath axes are *not* supported:

* preceding
* preceding_sibling
* namespace
* following_sibling
* following
* ancestor
* ancestor_or_self
* attribute

The following XPath functions as defined in Section 2.3 / 4 of the XPath 1.0 standard are supported:

* position
* count
* local-name
* name
* string
* starts-with
* contains
* substring-before
* substring-after
* substring
* string-length
* translate
* boolean
* not
* true
* false
* text
* node

The following standard XPath functions are *not* supported:

* ceiling
* comment
* concat
* floor
* id
* lang
* last
* namespace-uri
* normalize-space
* number
* processing-instructions
* round
* sum

Pagination
==========
The pagination solution is based on the following drafts:

- `<https://www.ietf.org/archive/id/draft-ietf-netconf-list-pagination-07.html>`_
- `<https://www.ietf.org/archive/id/draft-ietf-netconf-list-pagination-nc-07.html>`_
- `<https://www.ietf.org/archive/id/draft-ietf-netconf-list-pagination-rc-07.html>`_

Clixon implements all attributes except `cursor`, `locale`, `sublist-limit` and `remaining`.

See :ref:`Pagination section <clixon_pagination>` for more info.

Unicode
=======
Unicode is not supported in YANG and XML.

JSON
====
Clixon implements JSON according to:

- `ECMA JSON Data Interchange Syntax <http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf>`_
- `RFC 7951 JSON Encoding of Data Modeled with YANG <https://www.rfc-editor.org/rfc/rfc7951.txt>`_.
- `RFC 8259 The JavaScript Object Notation (JSON) Data Interchange Format <https://www.rfc-editor.org/rfc/rfc8259.txt>`_
