.. _clixon_advanced:

********
Advanced
********

(Work in progress, these are sections that do not fit into the rest of the document)

CLI
===

Differences to CLIgen
---------------------

Clixon adds some features and structure to CLIgen which include:

- A plugin framework for both textual CLI specifications(.cli) and object files (.so)
- Object files contains compiled C functions referenced by callbacks in the CLI specification. For example, in the cli spec command: `a,fn()`, `fn` must exist in the object file as a C function.
- The CLIgen `treename` syntax does not work.
- A CLI specification file is enhanced with the following CLIgen variables:

  - `CLICON_MODE`: A colon-separated list of CLIgen `modes`. The CLI spec in the file are added to _all_ modes specified in the list. You can also use wildcards `*` and `?`.
  - `CLICON_PROMPT`: A string describing the CLI prompt using a very simple format with: `%H`, `%U` and `%T`.
  - `CLICON_PLUGIN`: the name of the object file containing callbacks in this file.

- Clixon generates a command syntax from the Yang specification that can be referenced as `@datamodel`. This is useful if you do not want to hand-craft CLI syntax for configuration syntax.

Example of `@datamodel` syntax:
::
   
  set    @datamodel, cli_set();
  merge  @datamodel, cli_merge();
  create @datamodel, cli_create();
  show   @datamodel, cli_show_auto("running", "xml");		   

The commands (eg `cli_set`) will be called with the first argument an api-path to the referenced object.


How to deal with large specs
----------------------------
CLIgen is designed to handle large specifications in runtime, but it may be
difficult to handle large specifications from a design perspective.

Here are some techniques and hints on how to reduce the complexity of large CLI specs:

Sub-modes
^^^^^^^^^
The `CLICON_MODE` is used to specify in which modes the syntax in a specific file should be added. For example, if you have major modes `configure` and `operation` you can have a file with commands for only that mode, or files with commands in both, (or in all).

First, lets add a basic set in each:
::
   
  CLICON_MODE="configure";
  show configure;

First, lets add a basic set in each:
::
   
  CLICON_MODE="configure";
  show configure;

Note that CLI command trees are *merged* so that show commands in other files are shown together. Thus, for example:
::

  CLICON_MODE="operation:files";
  show("Show") files("files");

will result in both commands in the operation mode:
::

  > clixon_cli -m operation 
  cli> show <TAB>
    routing      files

but 
::

  > clixon_cli -m operation 
  cli> show <TAB>
    routing      files
  
Sub-trees
^^^^^^^^^
You can also use sub-trees and the the tree operator `@`. Every mode gets assigned a tree which can be referenced as `@name`. This tree can be either on the top-level or as a sub-tree. For example, create a specific sub-tree that is used as sub-trees in other modes:
::
   
  CLICON_MODE="subtree";
  subcommand{
    a, a();
    b, b();
  }

then access that subtree from other modes:
::
   
  CLICON_MODE="configure";
  main @subtree;
  other @subtree,c();

The configure mode will now use the same subtree in two different commands. Additionally, in the `other` command, the callbacks will be overwritten by `c`. That is, if `other a`, or `other b` is called, callback function `c` will be invoked.
  
C-preprocessor
^^^^^^^^^^^^^^

You can also add the C preprocessor as a first step. You can then define macros, include files, etc. Here is an example of a Makefile using cpp:
::
   
   C_CPP    = clispec_example1.cpp clispec_example2.cpp
   C_CLI    = $(C_CPP:.cpp=.cli
   CLIS     = $(C_CLI)
   all:     $(CLIS)
   %.cli : %.cpp
        $(CPP) -P -x assembler-with-cpp $(INCLUDES) -o $@ $<


Path identifiers: Api-path vs XPath
===================================

NETCONF and RESTONF use different path identifiers, and Clixon uses both methods:

* *XML Path Language* defined in `XPath 1.0 <https://www.w3.org/TR/xpath-10>`_ as part of XML and used in NETCONF `RFC 6241: NETCONF Configuration Protocol <http://www.rfc-editor.org/rfc/rfc6241.txt>`_.
* *Api-path* defined and used in `RFC 8040: RESTCONF Protocol <https://www.rfc-editor.org/rfc/rfc8040.txt>`_

Both use a denotation to find a path in tree, `a/b`, but differ in several ways.

Usage
-----
Example of XPath in a NETCONF get-config RPC using the XPath capability 
::

   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <get-config>
         <source>
	    <candidate/>
	 </source>
	 <filter type="xpath" select="/interfaces/interface[name='eth0']/description" />
      </get-config>
   </rpc>

The XPath is "/interfaces/interface[name='eth0']/description".
   
Example of Api-path in a restconf GET request, doing the same as the NETCONF request abobe:
::

   curl -s -X GET http://localhost/restconf/data/ietf-interfaces:interfaces/interface=eth0/description

The Api-path in the example abve is: "ietf-interfaces:interfaces/interface=eth0/description".

Clixon uses Api-paths internally when accessing xml keys, but has functions to translate to XPaths.

Scope
-----

XPath is a powerful language for addressing parts of an XML document, where a sub-part is locating paths. For example, the following is a valid XPath:
::

   /assembly[name="robot_4"]//shape/name[containts(text(),'bolt')]/surface/roughness

Api-path is in comparison very limited to pure path expressions such
as:
::
   
   a/b=3,4/c

which corresponds to the XPath eg: `a[i=3][j=4]`.

Namespaces
----------

XPath uses `XML names <https://www.w3.org/TR/REC-xml-names/>`_, requiring an *XML namespace context* using the `xmlns` attribute to bind namespaces and prefixes.  An XML namespace
context can specify both:

  * A default namespace for unprefixed names (eg `/x/`), defined by for example: `xmlns="urn:example:default"`.
  * An explicit namespace for prefixed names prefix (eg: `/ex:x/`), defined by for example: `xmlns:ex="urn:example:example"`.

Further, XML prefixes are not *inherited*, each symbol must be prefixed with a prefix or default. That is, `/ex:x/y` is not the same as `/ex:x/ex:y`, unless `ex` is also default.

Example:; Assume an XML namespace context:
::
   
   <a xmlns="urn:example:default" xmlns:ex="urn:example:example">

with an associated XPath:
::

   /x/ex:y/ex:z[ex:i='w']`,

where the symbol x belongs to namespace: "urn:example:default" and symbols y, z and i belong to "urn:example:example".

In contrast, the namespaces of an Api-path is defined implitly by a
YANG context using *module-names* as prefixes.  The namespace is defined in the Yang module by the `namespace` keyword. Api-paths
must have a Yang definition wheras XPaths can be compeletely defined
in XML.

A module-name is also *inherited*, ie a child node inherits the prefix
of a parent, and there are no defaults. For example, `/moda:x/y` is the same as `/moda:a/moda:y`.

Finally, an Api-path uses a shorthand for defining list *indexes*. For example, `/modx:z=w` denotes the element in a list of `z`:s whose key is the value `w`. This assumes that `z` is a Yang list (or leaf-list) and the index value is known.

Example: Assume two YANG modules `moda` and `modx` with namespaces "urn:example:default" and "urn:example:example" respectively, with the following Api-path (equivalent to the XPath example above):
::

   /moda:x/modx:y/z=w

where, as above, x belong to "urn:example:default" and y, and z belong to "urn:example:example".



  
