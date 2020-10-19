.. _clixon_cli:

CLI
===

The CLI uses `<https://github.com/clicon/cligen>`_ is a central part of Clixon. CLIgen can stand on its own but is fully integrated into Clixon. This section describes the Clixon integration, for details on CLI syntax, etc, please consult the `tutorial <https://github.com/clicon/cligen/blob/master/cligen_tutorial.pdf>`_.

Once the backend is started, the easiest way to use Clixon is via
the CLI. Clixon comes with a generated CLI, the *auto-cli*, where all
configuration-related syntax is generated from YANG. You can also create a completely manually-made CLI.

Using the CLI
-------------

The following example shows an auto-cli session from the `main example <https://github.com/clicon/clixon/tree/master/example/main>`_ how to add an interface in candidate, validate and commit it to running, then look at it as xml and cli and finally delete it::
   
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
   cli> show configuration cli
   set interfaces interface eth9 
   set interfaces interface eth9 type ex:eth
   set interfaces interface eth9 enabled true
   cli> delete interfaces interface eth9

CLI specs and plugins
---------------------

When defining a CLI frontend, there are two kinds of CLI specification files:

* *Clispecs*: CLI specification files (clispecs) written in `CLIgen <https://github.com/clicon/cligen/blob/master/cligen_tutorial.pdf>`_ syntax.
* *Plugins*: Dynamic loadable plugin files loaded by `clixon_cli` at startup. Callbacks from clispec files are resolved and need to exist as symbols either in the CLixon libs or in the plugin file.

The following example shows examples of both files taken from the `main example <https://github.com/clicon/clixon/tree/master/example/main>`_. First, a clispec file containing two commands::

  show("Show state") config("Show configuration"), cli_show_config("candidate", "text", "/");
  example("Callback example") <var:int32>("any number"), mycallback("myarg");

In the CLI, these will generate CLI commands such as::

   show config
   example 23

The effect of typing the commands above will be of calling the callbacks: `cli_show_config` and `mycallback`. Both these functions must exist as C-functions. In fact, `cli_show_config` is a library function avaliable in the Clixon libs, while `mycallback` is defined in the main example CLI plugin.

In this way, a designer writes cli command specifications which
invokes C-callbacks. If there are no appropriate callbacks the
designer must write a new callback function.
   
Cli spec variables
^^^^^^^^^^^^^^^^^^
A CLI specification file typically starts with the following variables:

CLICON_MODE
  A colon-separated list of CLIgen `modes`. The CLI spec in the file are added to _all_ modes specified in the list. You can also use wildcards `*` and `?`.

CLICON_PROMPT
  A string describing the CLI prompt using a very simple format with: ``%H`` (host) , ``%U`` (user) , ``%T`` (tty),  ``%W`` (working path).

CLICON_PLUGIN
  The name of the object file containing callbacks in this file.

Clixon config options
^^^^^^^^^^^^^^^^^^^^^
The following config options are related to clispec and plugin files (clixon config options), ie they are set in the XML Clixon config file:

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
  Startup CLI mode. This should match a ``CLICON_MODE`` variable setting in one of the clispec files. Default is "base".

Inside the clispec files ``CLICON_MODE`` is used to specify to which modes the syntax in a specific file defines. For example, if you have major modes `configure` and `operation` you can have a file with commands for only that mode, or files with commands in both, (or in all).

First, lets add a single command in the configure mode::
   
  CLICON_MODE="configure";
  show configure;

Then add syntax to both modes::

  CLICON_MODE="operation:configure";
  show("Show") files("Show files");

Finally, add a command to all modes::

  CLICON_MODE="*";
  show("Show") all("Show all");
   
Note that CLI command trees are merged so that show commands in other files are shown together. Thus, for example, using the clispecs above the two modes will be three commands in total for the *configure* mode::

  > clixon_cli -m configure
  cli> show <TAB>
    all     routing      files

but only two commands  in the *operation* mode::

  > clixon_cli -m operation 
  cli> show <TAB>
    all      files

Terminal
--------
Clixon CLI have the following terminal related options:

CLICON_CLI_LINESCROLLING
  Set to 0 if you want CLI to wrap to next line.
  Set to 1 if you want CLI to scroll sideways when approaching right margin (default).

CLICON_CLI_LINES_DEFAULT
   Set to number of CLI terminal rows for pageing/scrolling. 0 means unlimited.  The number is set statically UNLESS:

   * there is no terminal, such as file input, in which case nr lines is 0
   * there is a terminal sufficiently powerful to read the number of lines from ioctl calls.

In other words, this setting is used ONLY on raw terminals such as serial consoles.

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
You use sub-trees using the tree operator `@`. Every mode gets assigned a tree which can be referenced as `@name`. This tree can be either on the top-level or as a sub-tree. For example, create a specific sub-tree that is used as sub-trees in other modes::
   
  CLICON_MODE="subtree";
  subcommand{
    a, a();
    b, b();
  }

then access that subtree from other modes::
   
  CLICON_MODE="configure";
  main @subtree;
  other @subtree,c();

The configure mode will now use the same subtree in two different commands. Additionally, in the `other` command, the callbacks will be overwritten by `c`. That is, if `other a`, or `other b` is called, callback function `c` will be invoked.

Help strings
------------
Help strings are specified using the following example syntax: ``("help string")``. help strings are shown at queries, eg "?"::

    cli> show <?>
       all       Show all
       routing   Show routing
       files     Show files

For long or multi-line help strings the following options exists:

CLICON_CLI_HELPSTRING_TRUNCATE
  Set to 0 to wrap long help strings to the next line. (default)
  Set to 1 to truncate long help strings at the right margin

CLICON_CLI_HELPSTRING_LINES
  Set to 0 to have no limit on the number of help string lines per command
  Set to <n> to limit the the number of help string lines

Long and multi-line help strings may especially be needed in the auto-cli, see `The Auto-CLI`_.

Running CLI scripts
-------------------

The CLI can run scripts using either the ``-1`` option for single commands::

  clixon_cli -1 show version
  4.8.0.PRE

Or using the ``-F <file>`` command-line option to redirect input from file

  clixon_cli -F file

Or using "shebang"::

  #!/usr/local/bin/clixon_cli -F
  show version
  quit

The Auto-CLI
------------

The auto-cli contains parts that are *generated* from a YANG specification.

YANG
^^^^
Consider a YANG specification, such as::

  container x{
    list y{
      key k;
      leaf k{
        type string;
      }
    }
  }

If the ``clixon_cli`` is started with ``-G -o CLICON_CLI_GENMODEL=1`` it prints the following cli-spec::
  
    x,overwrite_me("/example:x");{
      y (<k:string> |
         <k:string expand_dbvar("candidate","/example:x/y=%s/k")>),
	     overwrite_me("/example:x/y=%s/");{
      }
   }

This cli-spec forms the basis of the auto-cli and contains the following:
  - Keywords for the YANG symbol (eg ``x`` and ``y``).
  - Variable syntax for leafs (eg ``<k:string>``)
  - Edit autoamtic modes and prompt showing path
  - Completion callbacks for variables with existing datastore syntax (eg ``expand_dbvar()``). That is, existing datastore content will be shown as alternatives.
  - ``overwrite_me`` is a callback template which is overwritten by an actual callback in the clispec (eg ``cli_set()``)

The auto-cli syntax can be copied and loaded seperately (in another mode file), or much simpler, just use the ``@datamodel`` tree directly in the regular cli-spec::

  CLICON_PROMPT="%U@%H %W> ";
  edit @datamodel, cli_auto_edit("datamodel", "candidate");
  up, cli_auto_up("datamodel", "candidate");
  top, cli_auto_top("datamodel", "candidate");
  set @datamodel, cli_auto_set();
  merge @datamodel, cli_auto_merge();
  create @datamodel, cli_auto_create();
  delete("Delete a configuration item") @datamodel, cli_auto_del();
  delete("Delete a configuration item") all("Delete whole candidate configuration"), delete_all("candidate");
  show("Show a particular state of the system"){
    configuration("Show configuration"), cli_auto_show("datamodel", "candidate", "xml", false, false);
    state("Show configuration and state"), cli_auto_show("datamodel", "running", "xml", false, true);
  }

Example
^^^^^^^

An example run of the above example is::

  olof@alarik> clixon_cli -f /usr/local/etc/example.xml
  olof@alarik /> set x 
    <cr>
    y                    
  olof@alarik /> set x y 23
  olof@alarik /> show configuration 
  <x xmlns="urn:example:clixon"><y><k>23</k></y></x>
  olof@alarik /> edit x  
  olof@alarik /clixon-example:x> show configuration 
  <y><k>23</k></y>
  olof@alarik /clixon-example:x> up
  olof@alarik /> 
  
The example shows an automated cli generated by the YANG model, with
completion as well as config to cli syntax.

Options
^^^^^^^
The config options for generated config tree is:

CLICON_CLI_GENMODEL, a numeric that can have the following values:
  0. Do not generate CLISPEC syntax for the auto-cli.
  1. Generate a CLI specification for CLI completion of all loaded Yang modules. This CLI tree can be accessed in CLI-spec files using the tree reference syntax (eg ``@datamodel``).
  2. Same including state syntax in a tree called ``@datamodelstate``.

CLICON_CLI_MODEL_TREENAME
  A string treename. CLI specs can reference the model syntax using this reference. Example: set ``@mydatamodel``, cli_set(); Default is ``datamodel``. Note that there are two variants of this tree: ``datamodelshow`` and ``datamodelstate``.
  
CLICON_CLI_GENMODEL_COMPLETION
  Generate code for CLI completion of existing db symbols. If set to 0, the ``expand`` rules will not be generated.
  
CLICON_CLI_GENMODEL_TYPE, How to generate and show CLI syntax. 
  - ``NONE`` No keywords on non-keys: ``x y <k>`` (example has only a key so same as VARS)
  - ``VARS`` Keywords on non-key variables: ``x y <k>``
  - ``ALL``  Keywords on all variables: ``x y k <k>``
  - ``HIDE`` Keywords on non-key variables and hide container around lists: ``y <k>``

