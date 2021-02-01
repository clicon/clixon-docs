.. _clixon_misc:

****
Misc
****

These are sections that do not fit into the rest of the document.

CLI
===


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

The transformation is defined by a list of changelogs. Each changelog defined how a module (defined by a namespace) is transformed from an old revision to a new. Example from `auto upgrade test script <https://github.com/clicon/clixon/tree/master/test/test_upgrade_auto.sh>`_::  

  <changelogs xmlns="http://clicon.org/xml-changelog">
    <changelog>
      <namespace>urn:example:b</namespace>
      <revfrom>2017-12-01</revfrom>
      <revision>2017-12-20</revision>
      ...
    <changelog>
  </changelogs>

Each changelog consists of set of (ordered) steps::

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

Extensions
==========

Clixon implements YANG extensions.  There are several uses, but one is
to "annotate" a YANG specification with application-specific data that can be used
in plugin code for some reason.

An extension with an argument is introduced in YANG as follows::

   module example-lib {
      namespace "urn:example:lib";
      extension mymode {
         argument annotation;
      }

Such an extension can then be used in YANG declarations in two ways, either
*inline* or *augmented*.

An inlined extension is useful in a YANG module that the designer has
control over and can add extension reference directly in the YANG
specification.

Assume for example that an interface declaration is extended with the extension declared above, as follows::

   module my-interface {
     import example-lib{
       prefix exl;
     }
     container "interfaces" {
       list "interface" {
         exl:mymode "my-interface";
         ...

If you instead use an external YANG, where you cannot edit the YANG
itself, you can use augmentation instead, as follows::

  module my-augments {
   import example-lib{
      prefix exl;
   }
   import ietf-interfaces{
      prefix if;
   }
   augment "/if:interfaces/if:interface"{
      exl:mymode "my-interface";
   }
   ...

When this is done, it is possible to access the extension value in
plugin code and use that value to perform application-specific
actions. For example, assume an XML interface object ``x`` retrieve
the annotation argument::

     char      *value = NULL;
     yang_stmt *y = xml_spec(x);

     if (yang_extension_value(y, "mymode", "urn:example:lib", &value) < 0)
        err;
     if (value != NULL){
        // use extension value
        if (strcmp(value, "my-interface") == 0)
	   ...
	 
A more advanced usage is possible via an extension callback
(``ca_callback``) which is defined for backend, cli, netconf and
restconf plugins. This allows for advanced YANG transformations. Please
consult the main example to see how this could be done.

High availability
=================

This is a brief note on a potential future feature.

Clixon is mainly a stand-alone app tightly coupled to the application/device with "shared fate", that is, if clixon fails, so does the application.

That said, the primary state is the *backend* holding the *configuration database* that can be shared in several ways. This is not implemented in Clixon, but potential implementation strategies include:
  * *Active/standby*: With a standard failure/liveness detection of a master backend, a standby could be started when the master fails using "-s running" (just picking up the state from the failed master). The default cache write-through can be used (``CLICON_DATASTORE_CACHE = cache``). Would suffer from outage during standby boot.
  * *Active/active*: The config-db cache is turned off (``CLICON_DATASTORE_CACHE = nocache``) and two backend process started with a load-balancing in front. Turning the cache off would suffer from performance degradation (and its not currently tested in regression tests). Would also need a failure/liveness detection.

In both cases the *config-db* would be a single-point-of-failure but could be mitigated by a replicated file system, for example.

Regarding clients:
  * the *CLI* and *NETCONF* clients are stateless and spun up on demand.
  * the *RESTCONF* daemon is stateless and can run as multiple instances (with a load-balancer)
