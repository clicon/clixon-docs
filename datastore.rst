.. _clixon_datastore:
.. sectnum::
   :start: 8
   :depth: 3

*********
Datastore
*********


 .. image:: datastore1.jpg
   :width: 70%

Clixon configuration datastores follow the Netconf model (from `RFC 6241: NETCONF Configuration Protocol <http://rfc-editor.org/rfc/rfc6241.txt>`_):

`Candidate`
   A configuration datastore that can be manipulated without impacting the device's current configuration and that can be committed to the running configuration datastore.
`Running`
   A configuration datastore holding the complete configuration currently active on the device.
`Startup`
   The configuration datastore holding the configuration loaded by the device when it boots. Only present on devices that separate the startup configuration datastore from the running configuration datastore.

Note that there may appear other datastores, Clixon is not limited to the three datastores above. For example, a `tmp` datastore appears in several cases as an intermediate datastore.
	 
Datastore files
===============
The mandatory `CLICON_XMLDB_DIR` option determines where the
datastores are placed. Example:
::

   <CLICON_XMLDB_DIR>/usr/local/var/example</CLICON_XMLDB_DIR>

The permission of the datastores files is accessible to the user that
starts the backend only. Typically this is `root`, but if the backend is started as a non-privileged user, or if privileges are dropped (see :ref:`Backend section<clixon_backend>`) this may be another user, such as in the following example where `clicon` is used:
::

   sh> ls -l /usr/local/var/example
   -rwx------ 1 clicon clicon   0 sep 15 17:02 candidate_db
   -rwx------ 1 clicon clicon   0 sep 15 17:02 running_db
   -rwx------ 1 clicon clicon   0 sep 14 18:12 startup_db

Note that a user typically does not access the datastores directly, it is possible to read, but write operations should not be done, since the backend daemon may use a datastore cache, see `Datastore caching`_.

   
Datastore and file formats
==========================
By default, the datastore files use pretty-printed XML, with the top-symbol `config`. The following is an example of a valid datastore:
::

   <config>
     <hello xmlns="urn:example:hello">
       <world/>
     </hello>
   </config>

The format of the datastores can be changed using the following options:
   
`CLICON_XMLDB_FORMAT`
   Datastore format. `xml` is the primary alternative. `json` is also available, while `text` and `cli` are available as file formats but not specifically for the datastore.
`CLICON_XMLDB_PRETTY`
   XMLDB datastore pretty print. The default value is `true`, which inserts spaces and line-feeds making the XML/JSON human readable. If false, the XML/JSON is more compact.

Note that the format settings applies to all datastores.

Other formats
-------------
While only XML and JSON are currently supported as datastore formats, Clixon also supports `CLI` and `TEXT` formats for printing, and saving and loading files.

The main example contains example code showing how to load and save a config using other formats.

Example of showing a config as XML, JSON, TEXT and CLI::
  
   cli> show configuration xml 
   <table xmlns="urn:example:clixon">
      <parameter>
         <name>a</name>
         <value>17</value>
      </parameter>
      <parameter>
         <name>b</name>
         <value>99</value>
      </parameter>
   </table>
   cli> show configuration json
   {
     "clixon-example:table": {
       "parameter": [
         {
           "name": "a",
           "value": "17"
         },
         {
           "name": "b",
           "value": "99"
         }
       ]
     }
   }
   cli> show configuration text
   clixon-example:table {
       parameter a {
           value 17;
       }
       parameter b {
           value 99;
       }
   }
   cli> show configuration cli 
   set table parameter a 
   set table parameter a value 17
   set table parameter b 
   set table parameter b value 99

Save and load a file using TEXT::

   cli> save foo.txt text
   cli> load foo.txt replace text 

Internal C API
^^^^^^^^^^^^^^
CLI show and save commands uses an internal API for print, save and load of the formats. Such CLI functions include: `cli_show_config`, `cli_pagination`, `load_config_file`, `save_config_file`.

The following internal C API is available for output:

* XML: ``clixon_xml2file()`` and ``clixon_xml2cbuf()`` to file and memory respectively.
* JSON: ``clixon_json2file()`` and ``clixon_json2cbuf()``
* CLI: ``clixon_cli2file()``
* TEXT: ``clixon_txt2file()``

The arguments of these functions are similar with some local variance. For example::

   int
   clixon_xml2file(FILE             *f, 
                   cxobj            *xn, 
		   int               level, 
		   int               pretty,
		   clicon_output_cb *fn,
		   int               skiptop,
		   int               autocliext)

where:

* `f` is the output stream (such as `stdout`)
* `xn` is the top-level XML node
* `level` is indentation level to start with, normally `0`
* `pretty` makes the output indented and use newlines
* `fn` is the output function to use. `NULL` means `fprintf`, `cligen_output` is used for scrolling in CLI
* `skiproot` only prints the children by skipping the top-level XML node `xn`
* `autocliext` Set if you want to activate autocli extensions (eg `hide` extensions)


Module library support
======================
Clixon can store Yang module-state information according to `RFC 8525: YANG library <http://www.rfc-editor.org/rfc/rfc8525.txt>`_ in the
datastores. With module state, you know which Yang version the XML belongs to, which is useful when upgrading, see :ref:`upgrade <clixon_upgrade>`.


To enable yang module-state in the datastores add the following entry in the Clixon configuration:
::

   <CLICON_YANG_LIBRARY>true</CLICON_YANG_LIBRARY> # (default true)
   <CLICON_XMLDB_MODSTATE>true</CLICON_XMLDB_MODSTATE>

If the datastore does not contain module-state, general-purpose upgrade is the only upgrade mechanism available.

A backend with `CLICON_XMLDB_MODSTATE` disabled will silently ignore module state.

Example of a (simplified) datastore with Yang module-state:
::
   
   <config>
     <yang-library xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-library">
       <content-id>42</content-id>
       <module-set>
         <name>default</name>
         <module>
           <name>A</name>
           <revision>2019-01-01</revision>
           <namespace>urn:example:a</namespace>
         </module>
       </module-set>
     </yang-library>
     <a1 xmlns="urn:example:a">some text</a1>
   </config>

Note that the module-state is not available to the user, the backend
datastore handler strips the module-state info. It is only shown in
the datastore itself.

Datastore caching
=================
Clixon datastore cache behaviour is controlled by the `CLICON_DATASTORE_CACHE` and can have the following values:

`nocache`
   No cache, always read and write directly with datastore file. 
`cache`
   Use in-memory write-through cache. Make copies of the XML when accessing internally by callbacks and plugins. This is the default.
`cache-zerocopy`
   Use in-memory write-through cache and do not copy when doing callbacks.  This is the fastest but opens up for callbacks changing the cache. That is, plugin callbacks may not edit the XML in any way.

.. note::
        Netconf locks are not supported for nocache mode
