.. _clixon_cli:

CLI
===

The CLI uses `<https://github.com/olofhagsand/cligen>`_ is a central part of Clixon. CLIgen can stand on its own but is fully integrated into Clixon. This section describes the Clixon integration, for details on CLI syntax, etc, please consult the `tutorial <https://github.com/olofhagsand/cligen/blob/master/cligen_tutorial.pdf>`_.

Using the CLI
-------------

Once the backend is started, the easiest way to use Clixon is via the CLI. 

The following example shows how to add an interface in candidate, validate and commit it to running, then look at it (as xml) and finally delete it:
::
   
   clixon_cli -f /usr/local/etc/example.xml 
   cli> set interfaces interface eth9 ?
     description               enabled                   ipv4                     
     ipv6                      link-up-down-trap-enable  type                     
   cli> set interfaces interface eth9 type ex:eth
   cli> validate 
   cli> commit 
   cli> show configuration xml 
   <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
     <interface>
       <name>eth9</name>
       <type>ex:eth</type>
       <enabled>true</enabled>
     </interface>
   </interfaces>
 cli> delete interfaces interface eth9

CLI specs and plugins
---------------------

When defining a CLI frontend, there are two kinds of CLI specification files:

* *Clispecs*: CLI specification files (clispecs) written in `CLIgen <https://github.com/olofhagsand/cligen/blob/master/cligen_tutorial.pdf>`_ syntax.
* *Plugins*: Dynamic loadable plugin files loaded by `clixon_cli` at startup. Callbacks from clispec files are resolved and need to exist as symbols either in the CLixon libs or in the plugin file.

The following example shows examples of both files taken from the `main example <https://github.com/clicon/clixon/tree/master/example/main>`_. First, a clispec file containing two commands:
::

  show("Show state") config("Show configuration"), cli_show_config("candidate", "text", "/");
  example("Callback example") <var:int32>("any number"), mycallback("myarg");

In the CLI, these will generate CLI commands such as:
::

   show config
   example 23

The effect of typing the commands above will be of calling the callbacks: `cli_show_config` and `mycallback`. Both these functions must exist as C-functions. In fact, `cli_show_config` is a library function avaliable in the Clixon libs, while `mycallback` is defined in the main example CLI plugin.

In this way, a designer writes cli command specifications which
invokes C-callbacks. If there are no appropriate callbacks the
designer must write a new callback function.
   
The following config options are related to clispec and plugin files:

CLICON_CLI_DIR
  Directory containing frontend cli loadable plugins. Load all `.so` plugins in this directory as CLI object plugins.

CLICON_CLISPEC_DIR
  Directory containing frontend cligen spec files. Load all `.cli` files in this directory as CLI specification files.

CLICON_CLISPEC_FILE
  Specific frontend cligen spec file as alternative or complement to `CLICON_CLISPEC_DIR`. Also available as `-c` in clixon_cli.



Modes
-----
The CLI can have different *modes* which is controlled by a config option and some internal clispec variables. The config option is:

CLICON_CLI_MODE
  Startup CLI mode. This should match a CLICON_MODE variable setting in one of the clispec files. Default is "base".

Inside the clispec files `CLICON_MODE` is used to specify to which modes the syntax in a specific file defines. For example, if you have major modes `configure` and `operation` you can have a file with commands for only that mode, or files with commands in both, (or in all).

First, lets add a single command in the configure mode:
::
   
  CLICON_MODE="configure";
  show configure;

Then add syntax to both modes:
::

  CLICON_MODE="operation:configure";
  show("Show") files("files");

Finally, add a command to all modes:

::

  CLICON_MODE="*";
  show("Show") all("all");
   
Note that CLI command trees are merged so that show commands in other files are shown together. Thus, for example, using the clispecs above the two modes will be three commands in total for the *configure* mode:
::

  > clixon_cli -m configure
  cli> show <TAB>
    all     routing      files

but only two commands  in the *operation* mode:
::

  > clixon_cli -m operation 
  cli> show <TAB>
    all      files

    
History
-------
Clixon CLI supports persistent command history. There are two CLI history related configuration options:

CLICON_CLI_HIST_FILE
  The file containing the history, default value is: `~/.clixon_cli_history`

CLICON_CLI_HIST_SIZE
  Max number of history line, default value is 300.


The design is similar to bash history but is simpler in some respects:
   - The CLI loads/saves its complete history to a file on entry and exit, respectively
   - The size (number of lines) of the file is the same as the history in memory
   - Only the latest session dumping its history will survive (bash merges multiple session history).

Further, tilde-expansion is supported and if history files are not found or lack appropriate access will not cause an exit but will be logged at debug level

Sub-trees
^^^^^^^^^
You use sub-trees using the tree operator `@`. Every mode gets assigned a tree which can be referenced as `@name`. This tree can be either on the top-level or as a sub-tree. For example, create a specific sub-tree that is used as sub-trees in other modes:
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

Generated config syntax
^^^^^^^^^^^^^^^^^^^^^^^
A special kind of sub-tree is the system-generated syntax tree.
The config options for generated config tree is:

CLICON_CLI_GENMODEL
  If set, generate CLI specification for CLI completion of loaded Yang modules. This CLI tree can be accessed in CLI spec files using the tree reference syntax (eg @datamodel). Default 1.

CLICON_CLI_MODEL_TREENAME
  If set, CLI specs can reference the model syntax using this reference. Example: set @datamodel, cli_set();
  
CLICON_CLI_GENMODEL_COMPLETION
  Generate code for CLI completion of existing db symbols
  
CLICON_CLI_GENMODEL_TYPE
  How to generate and show CLI syntax: VARS|ALL

