.. _clixon_pagination:
.. sectnum::
   :start: 16
   :depth: 3

**********
Pagination
**********

.. This is a comment

Pagination and scrolling is the technique of showing large amount of data in smaller
chunks. Pagination was introduced in Clixon 5.3 and updated in 5.4. Expect further development and changes.

Overview
========
The pagination solution is based on the following drafts:

- `<https://datatracker.ietf.org/doc/html/draft-wwlh-netconf-list-pagination-00>`_
- `<https://datatracker.ietf.org/doc/html/draft-wwlh-netconf-list-pagination-nc-02>`_
- `<https://datatracker.ietf.org/doc/html/draft-wwlh-netconf-list-pagination-rc-02>`_
  
The pagination in Clixon is currently restricted to the `offset` and `limit` attributes. For example, the following requests a list of 120 items in chunks of 20::

   get offset=0 limit=20
   get offset=20 limit=20
   ...
   get offset=100 limit=20

This is referred to as `stateless` pagination, since the state may
change between "get" calls. For paging of a consistent snapshot view,
consider `locked` pagination.

Clixon pagination encompasses several aspects:

1. Locked pagination
2. NETCONF/RESTCONF protocol extensions
3. CLI scrolling
4. Plugin state API

   
Locked pagination
=================
Using NETCONF, one can lock the datastore during a session to ensure that the data
is unchanged, such as::

   lock-db
   get offset=0 limit=20
   get offset=20 limit=20
   ...
   get offset=100 limit=20
   unlock-db

The use of locks guarantees a consistent view (snapshot) of config
data, but risks undefinite blocking in a CLI pagination situation for example.

For state pagination data, database locks are not useful. Instead
clixon extends the database lock mechanism with a specific
`state-paginate-lock` lock that can be used by the state paginate callback
developer. It works the same way as a database lock, but there is no
config database associated with it, it is specially made for paginated
state data only.

   
Pagination protocol
===================
In a RESTCONF a pagination request looks as follows::
   
   GET /localhost/restconf/data/example-social:members/uint8-numbers?offset=20&limit=10 HTTP/1.1
   Host: example.com
   Accept: application/yang-collection+json

In NETCONF a similar request is::

   <rpc>
      <get>
         <filter type="xpath" select="/es:members/es:uint8-numbers" xmlns:es="http://example.com/ns/example-social"/>
         <list-pagination xmlns="urn:ietf:params:xml:ns:yang:ietf-list-pagination-nc">
   	    <offset>20</offset>
	    <limit>10</limit>
	 </list-pagination>
      </get>
   </rpc>

In return, Clixon returns a reply with the requested number of entries (in NETCONF)::

   <rpc-reply>
      <data>
         <members xmlns="http://example.com/ns/example-social">
	    <member>
	       <member-id>alice</member-id>
	       <privacy-settings>
	          <post-visibility>public</post-visibility>
	       </privacy-settings>
	       <favorites>
	          <uint8-numbers>20</uint8-numbers>
  	          <uint8-numbers>21</uint8-numbers>
                  ...
    	          <uint8-numbers>29</uint8-numbers>
	       </favorites>
	    </member>
	 </members>
      </data>
   </rpc-reply>

CLI scrolling
=============
CLIgen has a scrolling mechanism that can be integrated with pagination. For example, showing the list from the example above::

   clixon_cli
   cli> show state /es:members/es:member[es:member-id=\'bob\']/es:favorites/es:uint64-numbers cli
   uint64-numbers 0
   uint64-numbers 1
   ...
   uint64-numbers 9
   --More--

CLI callbacks
-------------
CLI scrolling is implemented by the `cligen_output` function similar
to `printf` in syntax. By using cligen_output for all output, CLIgen
ensures a scrolling mechanism.

Clixon includes an example CLI callback function that combines
the scrolling mechanism of the CLI with NETCONF pagination called
`cli_pagination` with the following arguments:

- `xpath` : XPath of a leaf-list or list
- `prefix` : Prefix used in XPath (only one can be specified)
- `namespace` : Namespace associated with prefix
- `format` : one of xml, text, json, or cli
- `limit` : Number of lines of the pagination window

In the main example, cli_pagination is called as follows::
  
   show state <xpath:string> cli, cli_pagination("", "es", "http://example.com/ns/example-social", "cli", "10");

An application can use the `cli_pagination` callback, or create a
tailor-made CLI callback based on the example callback.


Backend pagination API
======================
While pagination of config data is built-in, state data needs backend plugin
callbacks. There is a special state pagination callback API where a
callback is bound to an xpath, and is called when a pagination request is made on an xpath.

Such a callback is registered with an XPath and a callback as follows::

   clixon_pagination_cb_register(h, mycallback, "/myxpath", myarg);

where the callback has the following signature::

   int 
   mycallback(void            *h,
              char            *xpath,
	      pagination_data  pd,
	      void            *arg)

The ``pd`` parameter has the following accessor functions::

   uint32_t pagination_offset(pagination_data pd)
   uint32_t pagination_limit(pagination_data pd)
   int      pagination_locked(pagination_data pd)
   cxobj*   pagination_xstate(pagination_data pd)

Essentially, the state callback requests state data for list/leaf-list `xpath` in the interval `[offset...offset+limit]`.

If `locked` is true, the plugin can cache the state data, return
further requests from the same cache until the lock on the "runníng"
database is released, thus forming an (implicit) transaction.  For
this, the ca_lockdb callback can be used as an end to the transaction of ``state-paginate-lock``.
Note that there is not explicit "start transaction", the first locked
pagination request acts as one.

See a detailed example in the main example.
