.. _clixon_netconf:

NETCONF
=======

Overview
--------

Netconf is an external client interface (cli and restconf are
others). Netconf is also used in the internal IPC.

::

                   +---------------------------------------+
                   |  +------------+  IPC  +------------+  |
                   |  |  netconf   | ----- |  backend   |  |
      User <-----> |  |------------|       |------------|  |
                   |  | nc plugins |       | be plugins |  |
                   |  +------------+       +------------+  |
                   +---------------------------------------+

External Netconf
----------------

Clixon implements the following NETCONF RFC:s:
Clixon implements the following NETCONF RFC:s:

* `RFC 6241: NETCONF Configuration Protocol <http://www.rfc-editor.org/rfc/rfc6241.txt>`_
* `RFC 6242: Using the NETCONF Configuration Protocol over Secure Shell (SSH) <http://www.rfc-editor.org/rfc/rfc6242.txt>`_
* `RFC 5277: NETCONF Event Notifications <http://www.rfc-editor.org/rfc/rfc5277.txt>`_
* `RFC 8341: Network Configuration Access Control Model <http://www.rfc-editor.org/rfc/rfc8341.txt>`_ (NACM). Notification not implemented.

The following RFC6241 capabilities/features are hardcoded in Clixon:

* :candidate (RFC6241 8.3)
* :validate (RFC6241 8.6)
* :xpath (RFC6241 8.9)
* :notification (RFC5277)

The following features are optional and can be enabled by setting CLICON_FEATURE:

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
2. Subtree filtering does not support namespaces yet.

Further, the capability negotiation (hello protocol) as defined in RFC6241 Sec 8.1 and RFC7950 Sec 5.6.4 is only partly implemented.

IPC
---

Clixon uses NETCONF in the IPC protocol between its clients
(cli/netconf/restconf) and the backend. This *internal* Netconf (IPC)
is slightly different from regular Netconf:

- A different framing
- Netconf extentions

.. note::
        The IPC is an internal interface, do not use externally
  
Framing
^^^^^^^
A fixed header using session id and message length before the netconf message::

  struct clicon_msg {
     uint32_t    op_len;     /* length of message. network byte order. */
     uint32_t    op_id;      /* session-id. network byte order. */
     char        op_body[0]; /* rest of message, actual data */
  };


Extensions
^^^^^^^^^^

The protocol have a couple of extensions to the Netconf protocol as follows:

* *content* - for ``get`` command with values "config", "nonconfig" or "all", to indicate which parts of state and config are requested. This option is taken from RESTCONF. Example::

    <rpc><get content="nonconfig"/></rpc>
    
* *depth* - for ``get`` and ``get-config`` how deep a tree is requested. Also from RESTCONF. Example::

    <rpc><get depth="2"/></rpc>
    
* *username* - for top-level ``rpc`` command. Indicates which user the client represents ("pseudo-user"). This is either the actual user logged in as the client (eg "peer-user") or can represent another user. The credentials mode determines the trust-level of the pseudo-username. Example::

    <rpc username="root"><close-session/></rpc>
    
* *autocommit* - for ``edit-config``. If true, perform a ``commit`` operation immediately after an edit. If this fails, make a ``discard`` operation. Example::

    <rpc><edit-config autocommit="true"><target><candidate/></target><config>...</config></edit-config></rpc>
    
* *copystartup* - for ``edit-config`` combined with autocommit. If true, copy the running db to the startup db after a commit. The combination with autocommit is the default for RESTCONF operations. Example::

     <rpc><edit-config autocommit="true" copystartup="true"><target><candidate/></target><config>...</config></edit-config></rpc>

* *objectcreate* and *objectexisted* - in the data field of ``edit-config`` XML data tree. In the request set objectcreate to false/true whether an object should be created if it does not exist or not. If such a request exists, then the ok reply should contain "objectexists" to indicate whether the object existed or not (eg prior to the operation). The reason for this protocol is to implement some RESTCONF PATCH and PUT functionalities. Example::

      <rpc><edit-config objectcreate="false"><target><candidate/></target>
         <config>
            <protocol objectcreate="true">tcp</protocol>
         </config>
      </edit-config></rpc>]]>]]>
      <rpc-reply><ok objectexisted="true"/></rpc-reply>]]>]]>

The reason for introducing the objectcreate/objectexisted attributes are as follows:
      * RFC 8040 4.5 PUT: if the PUT request creates a new resource, a "201 Created" status-line is returned.  If an existing resource is modified, a "204 No Content" status-line is returned.
      * RFC 8040 4.6 PATCH: If the target resource instance does not exist, the server MUST NOT create it.


