.. _clixon_cli:

CLI
===

::

                   +---------------------------------------+
                   |  +------------+  IPC  +------------+  |
                   |  |    cli     | ----- |  backend   |  |
      User <-----> |  |------------|       |------------|  |
                   |  |cli plugins |       | be plugins |  |
                   |  +------------+       +------------+  |
                   +---------------------------------------+

		  
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

The effect of typing the commands above is calling the callbacks: `cli_show_config` and `mycallback`. Both these functions must exist as C-functions. In fact, `cli_show_config` is a library function available in the Clixon libs, while `mycallback` is defined in the main example CLI plugin.

In this way, a designer writes cli command specifications which
invokes C-callbacks. If there are no appropriate callbacks the
designer must write a new callback function.
   
Cli spec variables
^^^^^^^^^^^^^^^^^^
A CLI specification file typically starts with the following variables:

CLICON_MODE
  A colon-separated list of CLIgen `modes`. The CLI spec in the file are added to *all* modes specified in the list. You can also use wildcards `*` and `?`.

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

CLI callbacks
-------------
CLI callback functions are "library" functions that an application may call from a clispec. A
user is expected to create new application-specific callbacks.

As an example, consider the following clispec::

   example("Callback example") <var:int32>("any number"), mycallback("myarg");
 
containing a keyword (`example`) and a variable (`var`) and `mycallback` is a cli callback with argument: (`myarg`).

In C, the callback has the following signature::

  int mycallback(clicon_handle h, cvec *cvv, cvec *argv);

Suppose a user enters the following command in the CLI::

  cli> example 23

The callback is called with the following parameters::

  cvv: 
     0: example 23
     1: 23
  argv:
     0: "myarg"

which means that `cvv` contains dynamic values set by the user, and `argv` contains static values set by the clispec designer.
  
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
   
Note that CLI command trees are merged so that show commands in other files are shown together. Thus, for example, using the clispecs above the two modes are the three commands in total for the *configure* mode::

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
   Set to number of CLI terminal rows for pagination/scrolling. 0 means unlimited.  The number is set statically UNLESS:

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

Further, tilde-expansion is supported and if history files are not found or lack appropriate access will not cause an exit but are logged at debug level

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

The configure mode will now use the same subtree in two different commands. Additionally, in the `other` command, the callbacks are overwritten by `c`. That is, if `other a`, or `other b` is called, callback function `c` is invoked.

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
  - Non-terminal nodes can be entered as automatic modes with prompt showing the current path
  - Completion callbacks for variables with existing datastore syntax (eg ``expand_dbvar()``). That is, existing datastore content is shown as alternatives.
  - Output syntax as cli, xml, json, as netconf commands
  - ``overwrite_me`` is a callback template which is overwritten by an actual callback in the clispec (eg ``cli_set()``)


The auto-cli syntax can be copied and loaded separately (in another mode file), or much simpler, just use the ``@datamodel`` tree directly in the regular cli-spec::

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
    configuration("Show configuration"), cli_auto_show("datamodel", "candidate", "text", true, false);{
	    xml("Show configuration as XML"), cli_auto_show("datamodel", "candidate", "xml", true, false);
	    cli("Show configuration as CLI commands"), cli_auto_show("datamodel", "candidate", "cli", true, false, "set ");
    }
    state("Show configuration and state"), cli_auto_show("datamodel", "running", "text", true, true); {
    	    xml("Show configuration and state as XML"), cli_auto_show("datamodel", "running", "xml", true, true);
  }

Operations
^^^^^^^^^^
The operations used in the autocli are based on `RFC 6241: NETCONF Configuration Protocol <http://rfc-editor.org/rfc/rfc6241.html#section-7.2>`_:

  - ``merge``: merge with configuration at the corresponding level
  - ``set`` (actually ``replace``): replace existing configuration
  - ``create``: added to configuration
  - ``delete``: deleted from configuration

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

Hidden commands
^^^^^^^^^^^^^^^

You can implement the "hidden command" cligen feature by using the
"autocli-op" extension in `clixon-lib.yang`. This is done by
annotating a YANG specification with that extension. In the auto-cli,
that command will not active but not visible in the CLI.

Example YANG with hide extension of "val" leaf::
  
   import clixon-lib{
      prefix cl;
   }
   container x{
      list y{
         ...
         leaf val{
            type string;
            cl:autocli-op hide;
         }
      }
   }


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
  - ``OC_COMPRESS`` Compress OpenConfig paths, see https://github.com/openconfig/ygot/blob/master/docs/design.md for more information

CLICON_CLI_AUTOCLI_EXCLUDE,
  List of module names that are not generated autocli for.

Bits
----
The Yang bits built-in type as defined in RFC 7950 Sec 9.7 provides a set of bit names. In the CLI, the names should be given in a white-spaced delimited list, such as ``"fin syn rst"``.

The RFC defines a "canonical form"where the bits appear ordered by their position in YANG, but Clixon validation accepts them in any order.

Given them in XML and JSON follows thus, eg XML::

   <flags>fin rst syn</flags>

Clixon CLI does not treat individual bits as "first-level objects". Instead it only validates the whole string of bit names. Operations (add/remove) are made atomically on the whole string.

Translators
-----------
CLIgen supports wrapper functions that can take the output of a
callback and transform it to something else.

The CLI can perform variable translation. This is useful if you want to
process the input, such as hashing, encrypting or in other way
translate the input.

The following example is based on the main Clixon example and is included in the regression tests. In the following CLI specification, a "translate" command sets a modifed value to the "table/parameter=translate/value"::

  translate <value:string translate:cli_incstr()>, cli_set("/clixon-example:table/parameter=translate/value");

If you run this example using the `cli_incstr()` function which increments the characters in the input, you get this result::

  cli> translate HAL
  cli> show configuration
  table {
     parameter {
        name translate;
        value IBM;
     }
  }

The example is very simple and based on strings, but can be used also for other types and more advanced functions.
