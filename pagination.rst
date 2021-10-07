.. _clixon_pagination:

Pagination
==========

.. This is a comment

Pagination and scrolling is the technique of showing large amount of data in smaller
chunks. pagination was introduced in Clixon 5.3.

Overview
--------

The pagination protocol is loosely based on `<https://tools.ietf.org/id/draft-wwlh-netconf-list-pagination-nc-01.html>`_ and others for lists and leaf-lists.

The pagination in Clixon is currently restricted to the `offset` and `limit` attributes. For example, the following requests a list of 120 items in chunks of 20::

   get offset=0 limit=20
   get offset=20 limit=20
   ...
   get offset=100 limit=20

Using NETCONF, one can lock the datastore during a session to ensure that the data
is unchanged, such as::

   lock-db
   get offset=0 limit=20
   get offset=20 limit=20
   ...
   get offset=100 limit=20
   unlock-db

Clixon pagination encompasses several aspects:

1. NETCONF/RESTCONF protocol extensions
2. CLI scrolling
3. Plugin state API
   
Pagination protocol
-------------------

In a RESTCONF a pagination request looks as follows::
   
   GET /localhost/restconf/data/example-social:members/uint8-numbers?offset=20&limit=10 HTTP/1.1
   Host: example.com
   Accept: application/yang-collection+json

In NETCONF a similar request is::

   <rpc>
      <get>
         <filter type="xpath" select="/es:members/es:uint8-numbers" xmlns:es="http://example.com/ns/example-social"/>
         <list-pagination xmlns="http://clicon.org/clixon-netconf-list-pagination">true</list-pagination>
   	 <offset xmlns="http://clicon.org/clixon-netconf-list-pagination">20</offset>
	 <limit xmlns="http://clicon.org/clixon-netconf-list-pagination">10</limit>
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
-------------

CLIgen has a scrolling mechanism that can be integrated with pagination. For example, showing the list from the example above::

   clixon_cli
   cli> show state /es:members/es:member[es:member-id=\'bob\']/es:favorites/es:uint64-numbers cli
   uint64-numbers 0
   uint64-numbers 1
   ...
   uint64-numbers 9
   --More--

CLI callbacks
^^^^^^^^^^^^^

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


Plugin state API
----------------

While pagination of config data is built-in, state data needs plugin callbacks. A new state callback API has been made which extends the previous state callback as follows::

  char             *xpath,
  pagination_mode_t pagmode,
  uint32_t          offset, 
  uint32_t          limit
  uint32_t         *remaining

Essentially, the state callback requests state data forlist/leaf-list `xpath` in the interval `[offset...offset+limit]` and returns `remaining`.

The pagination mode is either:

  - `PAGINATION_NONE`      No list pagination: limit/offset are no-ops 
  - `PAGINATION_STATELESS` Stateless list pagination, do not expect more pagination calls
  - `PAGINATION_LOCK`      Transactional list pagination, can expect more pagination until lock release

In the `PAGINATION_LOCK` case, the plugin can cache the state data, return
further requests from the same cache until the lock on the "runníng"
database is released, thus forming an (implicit) transaction.  For
this, the ca_lockdb callback can be used as an end to the transaction.
Note that there is not explicit "start transaction", the first locked
pagination request acts as one.

