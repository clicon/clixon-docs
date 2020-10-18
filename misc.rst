.. _clixon_misc:

****
Misc
****

These are sections that do not fit into the rest of the document.

CLI
===

Translators
-----------

CLIgen supports wrapper functions that can take the output of a
callback and transform it to something else.

The CLI can perform variable translation. This is useful if you want to
process the input, such as hashing, encrypting or in other way
translate the input.

Yang example::

  list translate{
     leaf value{
        type string;
     }
  }

CLI specification::

  translate value (<value:string translate:incstr()>),cli_set("/translate/value");

If you run this example using the `incstr()` function which increments the characters in the input, you get this result::

  cli> translate value HAL
  cli> show configuration
  translate {
      value IBM;
  }

You can perform translation on any type, not only strings.


Differences to CLIgen
---------------------

Clixon adds some features and structure to CLIgen which include:

- A plugin framework for both textual CLI specifications(.cli) and object files (.so)
- Object files contains compiled C functions referenced by callbacks in the CLI specification. For example, in the cli spec command: `a,fn()`, `fn` must exist in the object file as a C function.
- The CLIgen `treename` syntax does not work.
- A CLI specification file is enhanced with the CLIgen variables `CLICON_MODE`, `CLICON_PROMPT`, `CLICON_PLUGIN`.
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


Automatic upgrades
==================
There is an EXPERIMENTAL xml changelog feature based on
"draft-wang-netmod-module-revision-management-01" (Zitao Wang et al)
where changes to the Yang model are documented and loaded into
Clixon. The implementation is not complete.

When upgrading, the system parses the changelog and tries to upgrade
the datastore automatically. This feature is experimental and has
several limitations.

You enable the automatic upgrading by registering the changelog upgrade method in ``clixon_plugin_init()`` using wildcards::

   upgrade_callback_register(h, xml_changelog_upgrade, NULL, 0, 0, NULL);

The transformation is defined by a list of changelogs. Each changelog defined how a module (defined by a namespace) is transformed from an old revision to a nnew. Example from `auto upgrade test script <https://github.com/clicon/clixon/tree/master/test/test_upgrade_auto.sh>`_::  

  <changelogs xmlns="http://clicon.org/xml-changelog">
    <changelog>
      <namespace>urn:example:b</namespace>
      <revfrom>2017-12-01</revfrom>
      <revision>2017-12-20</revision>
      ...
    <changelog>
  </changelogs>

Each changelog consists of set of (orderered) steps::

    <step>
      <name>1</name>
      <op>insert</op>
      <where>/a:system</where>
      <new><y>created</y></new>
    </step>
    <step>
      <name>2</name>
      <op>delete</op>
      <where>/a:system/a:x</where>
    </step>

Each step has an (atomic) operation:

* rename - Rename an XML tag
* replace - Replace the content of an XML node
* insert - Insert a new XML node
* delete - Delete and existing node
* move - Move a node to a new place

A *step* has the following arguments:

* where - An XPath node-vector pointing at a set of target nodes. In most operations, the vector denotes the target node themselves, but for some operations (such as insert) the vector points to parent nodes.
* when - A boolean XPath determining if the step should be evaluated for that (target) node.

Extended arguments:

* tag - XPath string argument (rename)
* new - XML expression for a new or transformed node (replace, insert)
* dst - XPath node expression (move)

Step summary:

* rename(where:targets, when:bool, tag:string)
* replace(where:targets, when:bool, new:xml)
* insert(where:parents, when:bool, new:xml)
* delete(where:parents, when:bool)
* move(where:parents, when:bool, dst:node)

Internal NETCONF
================

Clixon uses NETCONF as the protocol between its clients
(cli/netconf/restconf) and the backend. This _internal_ Netconf
protocol have a couple of extensions to the Netconf protocol as follows:

* *content* - for ``get`` command with values "config", "nonconfig" or "all", to indicate which parts of state and config are requested. This option is taken from RESTCONF. Example::

    <rpc><get content="nonconfig"/></rpc>
    
* *depth* - for ``get`` and ``get-config`` how deep a tree is requested. ALso from RESTCONF. Example::

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
