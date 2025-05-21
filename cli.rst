.. _clixon_cli:
.. sectnum::
   :start: 9
   :depth: 3

***
CLI
***

.. image:: cli1.jpg
   :width: 80%

Overview
========

The Clixon CLI provides an interactive command-line interface
to a user. Each usage instantiates a new process which communicates
via NETCONF with the backend daemon over an IPC socket.

The Clixon CLI uses `CLIgen <https://www.cligen.se>`__, an interactive
interpreter of commands. Syntax is given as *cli-specifications* which
specify callbacks defined in plugins.

For details on CLIgen syntax and behavior, please consult the `CLIgen tutorial <https://github.com/clicon/cligen/blob/master/cligen_tutorial.pdf>`_.

Clixon comes with a generated CLI, the `autocli`_, where all
configuration-related syntax is generated from YANG.

You can also create a completely manually-made CLI.

The CLI depends on the following:

* *Clixon-config*: The Clixon config-file contains initial CLI configuration, such as where to find cli-specs, plugins and autocli configuration.
* *Cli-specs*: CLI specification files written in `CLIgen <https://github.com/clicon/cligen/blob/master/cligen_tutorial.pdf>`_ syntax.
* *Plugins*: Dynamic loadable plugin files loaded at startup. Callbacks from cli-spec files are resolved and need to exist as symbols either in the Clixon libs or in the plugin file.

The following example from the `main example <https://github.com/clicon/clixon/tree/master/example/main>`_. First, a cli-spec file containing two commands::

  set("Set configuration symbol") @datamodel, cli_auto_set();
  show("Show a particular state of the system") configuration("Show configuration"), cli_show_config("candidate", "default", "/");
  example("Callback example") <var:int32>("any number"), mycallback("myarg");

In the CLI, these generate CLI commands such as::

   set interfaces interface eth9
   show config
   example 23

The effect of typing the commands above is calling callbacks, either
library functions in Clixon libs(``cli_show_config()``), or
application-defined in a plugin(``mycallback()``)

In this way, a designer writes cli command specifications which
invokes C-callbacks. If there are no appropriate callbacks the
designer must write a new callback function.

Example usage
-------------
The following example shows an auto-cli session from the `main example <https://github.com/clicon/clixon/tree/master/example/main>`_ how to add an interface in candidate, validate and commit it to running, then look at it as xml and cli and finally delete it::

   clixon_cli -f /usr/local/etc/clixon/example.xml
   user@host> set interfaces interface eth9 ?
     description               enabled                   ipv4
     ipv6                      link-up-down-trap-enable  type
   user@host> set interfaces interface eth9 type ex:eth
   user@host> validate
   user@host> commit
   user@host> show configuration xml
   <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
     <interface>
       <name>eth9</name>
       <type>ex:eth</type>
       <enabled>true</enabled>
     </interface>
   </interfaces>
   user@host> show configuration cli
   set interfaces interface eth9
   set interfaces interface eth9 type ex:eth
   set interfaces interface eth9 enabled true
   user@host> delete interfaces interface eth9

Command-line options
--------------------
The `clixon_cli` client has the following command-line options:
  -h              Help
  -V              Show version and exit
  -D <level>      Debug level
  -f <file>       Clixon config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut, (n)one or (f)ile. Stderr is default.
  -C <format>     Dump configuration options on stdout after loading. Format is one of xml|json|text|cli|default
  -F <file>       Read commands from file (default stdin)
  -1              Run once, do not enter interactive mode
  -a <family>     Internal IPC backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal IPC socket domain path or IP addr (see -a)
  -d <dir>        Specify cli plugin directory
  -m <mode>       Specify plugin syntax mode
  -q              Quiet mode, do not print greetings or prompt, terminate on ctrl-C
  -p <dir>        Add Yang directory path (see CLICON_YANG_DIR)
  -G              Print auo-cli CLI syntax generated from YANG
  -L              Debug print dynamic CLI syntax including completions and expansions
  -y <file>       Load yang spec file (override yang main modul)e
  -c <file>       Specify cli spec file
  -U <user>       Over-ride unix user with a pseudo user for NACM.
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

Inline CLI commands
^^^^^^^^^^^^^^^^^^^
CLI commands can be given directly after the options. These are executed directly::

  clixon_cli -f example.xml show config
  clixon_cli -f example.xml set table parameter b \; show config

Multiple commands are separated with `\;`.
One can also add extra application-dependent plugin options after `--` which can be read with `clicon_argv_get()`::

    clixon_cli -f example.xml show config -- -x extra-option

Configure options
=================
The following config options are related to clispec and plugin files (clixon config options), ie they are set in the XML Clixon config file:

CLICON_CLI_DIR
  Directory containing frontend cli loadable plugins. Load all `.so` plugins in this directory as CLI object plugins.

CLICON_CLISPEC_DIR
  Directory containing frontend cligen spec files. Load all `.cli` files in this directory as CLI specification files.

CLICON_CLI_PIPE_DIR
  Directory where CLI pipe scripts or binaries are searched for in cli callback

CLICON_CLISPEC_FILE
  Specific frontend cligen spec file as alternative or complement to `CLICON_CLISPEC_DIR`. Also available as `-c` in clixon_cli.

CLICON_CLI_OUTPUT_FORMAT
  Default CLI output format.

Terminal I/O
------------
Clixon CLI have the following configuration options related to terminal I/O:

CLICON_CLI_LINESCROLLING
  Set to `0` if you want CLI to wrap to next line.
  Set to `1` if you want CLI to scroll sideways when approaching right margin (default).

CLICON_CLI_LINES_DEFAULT
Set to number of CLI terminal rows for pagination/scrolling. `0` means unlimited.  The number is set statically UNLESS:

   * there is no terminal, such as file input, in which case nr lines is `0`
   * there is a terminal sufficiently powerful to read the number of lines from ioctl calls.

In other words, this setting is used ONLY on raw terminals such as serial consoles.

CLICON_CLI_TAB_MODE
   Set CLI tab mode. See detailed info in YANG source

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

CLI command log
^^^^^^^^^^^^^^^
The history API also allows for adding CLI command logging. Register the callback in the plugin init, and then call logging (or debug) for each CLI command. Example::

   static int
   cli_history_cb(cligen_handle ch,
                  char         *cmd,
                  void         *arg)
   {
      int           retval = -1;

      return clixon_log(arg, LOG_INFO, "command: %s", cmd);
   }

   clixon_plugin_api *
   clixon_plugin_init(clixon_handle h)
   {
      ...
      cligen_hist_fn_set(cli_cligen(h), cli_history_cb, h);

Help strings
------------
Help strings are specified using the following example syntax: ``("help string")``. help strings are shown at queries, eg "?"::

    user@host> show <?>
       all       Show all
       routing   Show routing
       files     Show files

For long or multi-line help strings the following configure options exists:

CLICON_CLI_HELPSTRING_TRUNCATE
  Set to 0 to wrap long help strings to the next line. (default)
  Set to 1 to truncate long help strings at the right margin

CLICON_CLI_HELPSTRING_LINES
  Set to 0 to have no limit on the number of help string lines per command
  Set to <n> to limit the the number of help string lines

Long and multi-line help strings may especially be needed in the auto-cli, see `autocli`_.

Modes
-----
The CLI can have different *modes* which is controlled by a config option and some internal clispec variables. The config options are:

CLICON_CLI_MODE
  Startup CLI mode. This should match a ``CLICON_MODE`` variable setting in one of the clispec files. Default is "base".
CLICON_CLI_VARONLY
  Do not include keys in cvec in cli vars callbacks

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
  user@host> show <TAB>
    all     routing      files

but only two commands  in the *operation* mode::

  > clixon_cli -m operation
  user@host> show <TAB>
    all      files

Cli-spec variables
------------------
A CLI specification file (note not clixon config file) typically starts with the following variables:

CLICON_MODE
  A colon-separated list of CLIgen `modes`. The CLI spec in the file are added to *all* modes specified in the list. You can also use wildcards ``*`` and '`?``.

CLICON_PROMPT
  A string describing the CLI prompt using a very simple format with: ``%H`` (host) , ``%U`` (user) , ``%T`` (tty),  ``%W`` (last element of working path), ``%w`` (full working path).

CLICON_PLUGIN
  The name of the object file containing callbacks in this file.

CLICON_PIPETREE
  Name of a pipe output tree as described in

CLI callbacks
=============
CLI callback functions are "library" functions that an application may call from a clispec. A
user is expected to create new application-specific callbacks.

There are two major types of CLI callbacks:

1. Command callbacks, to implement the semantics of a CLI command
2. Expand callbacks, for providing a set of values to a variable

Command callbacks
-----------------
A CLI command callback is invoked as a result of a command and implements its semantics,
and is  given as the last statement in a command.

Clixon provides several command callbacks, but a developer typically
adds new commands applicable for the application.

Consider the following clispec example of the command callback ``mycallback()``::

   example("Callback example") <var:int32>("any number"), mycallback("myarg");

The example contains a keyword (`example`) and a variable (`var`), while `mycallback` is the CLI callback with argument: (`myarg`).

In C, the callback has the following signature::

  int mycallback(clixon_handle h, cvec *cvv, cvec *argv);

Suppose a user enters the following command in the CLI::

  user@host> example 23

The callback is called with the following parameters::

  cvv:
     0: example 23
     1: 23
  argv:
     0: "myarg"

which means that `cvv` contains dynamic values set by the user, and `argv` contains static values set by the clispec designer.

Expand callbacks
----------------
Some variables may need a dynamic set of values to choose from, for example, currently existing interface names.

An expand callback is always given `inside` a variable declaration.

Clixon currently provides the following set of expand callbacks:

- ``expand_dbvar()``: finding datastore symbols
- ``expand_dir()``: for files in a directory
- ``expand_yang_list()``: finding YANG symbols

Just like command callbacks, a developer can add new expand functions.

An example of finding datastore symbols in the `candidate` datastore::

   table parameter <var:int32 expand_dbvar("candidate", "/clixon-example:table/parameter=%s/value")>

where the second argument is an `api-path` (see Section `Api-path-fmt`_) defining a datastore object.

Example, assume there are two table parameters in the candidate datastore::

  cli> table parameter <?>
       a     Table paramter
       b     Table parameter

Expanding of leafrefs
^^^^^^^^^^^^^^^^^^^^^
Expansion of leafrefs is by default the referred node, but can be changed by the the `leafref-no-refer` label.
Typically, "add" operations follow the reference to the referred nodes while "delete" operations do not.

With an example YANG::

   list parameter{
      key name;
      leaf name{
         description "referred node";
         type string;
      }
   }
   leaf-list leafref{
      description "referring node";
      type leafref{
         path "../parameter/name";
   }

Example clispec::

   set leafref <var:int32 expand_dbvar(...) ...;
   delete leafref <var:int32 expand_dbvar(...), leafref-no-refer> ...;

Similarly, using tree references::

  add @tree, cli_add();
  delete @tree, @add:leafref-no-refer, cli_delete();

Show commands
=============
Clixon includes show commands for showing datastore and state content. An application may use these functions as basis for more specialized show functions. Some show functions are:

- ``cli_show_config()`` - Multi-purpose show function for manual CLI show commands
- ``cli_show_auto()`` - Used in conjunction with the autocli with expansion trees
- ``cli_show_auto_mode()`` - Used in conjunction with the autocli with edit-modes
- ``cli_pagination()`` - Show paginated data of a large list

The CLI show functions are utility functions in the sense that they are not part of the core functionality and a user or product may want to specialize them.

.. note::
        CLI library functions are subject to change in new releases

cli_show_config
---------------
The ``cli_show_config`` is a basic function to display datastore and state data. A typical use in a cli spec is as follows::

    show("Show configuration"), cli_show_config("candidate", "text");

Using this command in the CLI could provicde the following output::

    cli> show
    <table xmlns="urn:example:clixon">
       <parameter>
          <name>a</name>
          <value>x</value>
       </parameter>
    </table>

The callback has the following parameters, only the first is mandatory:

 * `dbname` : Name of datastore to show, such as "running", "candidate" or "startup"
 * `format` : Show format, one of `text`, `xml`, `json`, `cli`, `netconf`, or `default` (see :ref:`datastore formats <clixon_datastore>`)
 * `xpath`  : Static xpath (only present in this API function)
 * `namespace` : Default namespace for xpath (only present in this API function)
 * `pretty` : If `true`, make output pretty-printed
 * `state`  : If `true`, include non-config data in output
 * `default` : Optional default retrieval mode: one of `report-all`, `trim`, `explicit`, `report-all-tagged`. See also extended values below
 * `prepend` : Optional prefix to prepend before cli syntax output, only valid for CLI format.
 * `fromroot` : If `false` show from xpath node, if `true` show from root

cli_show_auto
-------------
The ``cli_show_auto()`` callback is used together with the autocli to show sub-parts of a configured tree using expansion. A typical definition is as follows::

      show("Show expand") @datamodelshow, cli_show_auto("candidate", "xml");

That is, it must always be used together with a tree-reference as described in Section `autocli`_.

An example CLI usage is::

    cli> show table parameter a
    <parameter>
       <name>a</name>
       <value>x</value>
    </parameter>

The arguments are similar to `cli_show_config` with the difference that the `xpath` is implicitly defined by the current position of the tree reference::

 * `dbname` : Name of datastore to show, such as "running", "candidate" or "startup"
 * `format` : Show format, one of `text`, `xml`, `json`, `cli`, `netconf`, or `default` (see :ref:`datastore formats <clixon_datastore>`)
 * `pretty` : If `true`, make output pretty-printed
 * `state`  : If `true`, include non-config data in output
 * `default` : Optional default retrieval mode: one of `report-all`, `trim`, `explicit`, `report-all-tagged`. See also extended values below
 * `prepend`  : Opional prefix to print before cli syntax output, only valid for CLI format.

cli_show_auto_mode
------------------
The ``cli_show_auto_mode()`` callback also used together with the autocli but instead of expansion uses the edit-modes (see Section `edit modes`_).
A typical definition is::

      show, cli_show_auto_mode("candidate");

An example usage using edit-modes is::

      cli> edit table
      cli> show
      <parameter>
        <name>a</name>
        <value>x</value>
      </parameter>

Same parameters as ``cli_show_auto``

cli_start_program
------------------
You need to add a callback and its name in the `.cli` file.

Here is an example of how you can do it::

    run_program_err("Run program"), cli_start_program();
    run_program_python3("Run program"), cli_start_program("python3");
    run_program_python3_source_arg("Run program"), cli_start_program("python3", "/tmp/test.py");
    run_program_python3_source_arg_vector("Run program") <source:rest>("Path program"), cli_start_program("python3");
    run_program_python3_source_arg_vector_err("Run program") <source:rest>("Path program"), cli_start_program("python3", "/tmp/test2.py");
    run_program_bash("Run program"), cli_start_program("bash");

What the above-described callbacks do:

Starts a new process and the specified program in it. The function creates a new child process and starts the specified program in it.
Before starting the program in the child process, environment variables are set from the cvv.
The function checks the parameters passed to it. Situations where at least one function argument is absent
or when two arguments are present both in the function and in the cvv vector are considered invalid.


``run_program_python3_source_arg_vector_err`` will fail because you can either enter interactively or write in a call::

    roma@f-15 /> run_program_python3_source_arg_vector_err /tmp/test.py
    Aug 14 08:52:47.375126: cli_start_program: 830: Plugins: A lot of arguments: Invalid argument
    CLI command error

``run_program_err`` will fail because there are no parameters::

    user@pc_name /> run_program_err
    Aug 14 08:37:41.386765: cli_start_program: 826: Plugins: Can not found argument in a function: Invalid argument
    CLI command error

``run_program_bash`` executes `bash`::

    user@pc_name /> run_program_bash
    user_pc@pc_name:~#

``run_program_python3`` runs `python3`::

    user@pc_name /> run_program_python3
    Python 3.11.2 (main, May  2 2024, 11:59:08) [GCC 12.2.0] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>>

``run_program_python3_source_arg`` runs `/tmp/test.py` using `python3`.

`/tmp/test.py` file ::

    #!/usr/bin/python3
    print("Hello world! :)")

Run: ::

    user@pc_name /> run_program_python3_source_arg
    Hello world! :)
    user@pc_name />

``run_program_python3_source_arg_vector`` entails interactive input of the filename ::

    user@pc_name run_program_python3_source_arg_vector /tmp/test.py
    Hello world! :)
    user@pc_name />

Common show parameters
----------------------

with-default parameter
^^^^^^^^^^^^^^^^^^^^^^
All show commands have an optional `with-default` retrieval mode: one of `report-all`, `trim`, `explicit`, `report-all-tagged`. There are also extra propriatary modes of `default` serving as examples:

 * `NULL`, default with-default value, usually `report-all`
 * `report-all-tagged-default`, which gets the config as `report-all-tagged` but strips the tags/attributes (same as `report-all`).
 * `report-all-tagged-strip`, which also gets the config as `report-all-tagged` but strips the nodes associated with the default tags (same as `trim`).

pretty parameter
^^^^^^^^^^^^^^^^
All show commands have a pretty-print parameter. If `true` the putput is pretty-printed.
Indentation level is controlled by the ``PRETTYPRINT_INDENT`` compile-time option

Output pipes
============
Output pipes resemble UNIX shell pipes and are useful to filter or modify CLI output. Example::

  cli> print all | grep parameter
    <parameter>5</parameter>
    <parameter>x</parameter>
  cli> show config | showas json
    {
      "table": {
        "parameter": [
            ...
  cli>

Output pipe functions are declared using a special variant of a CLI tree
with a name starting with a `vertical bar`. Example::

  CLICON_MODE="|mypipe";
  \| {
     grep <arg:rest>, pipe_grep_fn("-e", "arg");
     showas json, pipe_json_fn();
  }

where ``pipe_grep_fn`` and ``pipe_json_fn`` are special callbacks that use stdio to modify output.

Such a pipe tree can be referenced with either an explicit reference, or an implicit rule.

Only a single level of pipes is possibly in this release. For example, ``a|b|c`` is `not` possible.

.. note::
        Only one level of pipes is supported

Explicit reference
------------------
An explicit reference is for single commands. For example, adding a pipe to the print commands::

   print, print_cb("all");{
      @|mypipe, print_cb("all");
      all @|mypipe, print_cb("all");
      detail;
   }

where a pipe tree is added as a tree reference, appending pipe functions to the regular ``print_cb`` callback.
Note that the following commands are possible in this example::

   print
   print | count
   print all | count
   print detail

Implicit rule
-------------
An implicit rule adds pipes to `all` commands in a cli mode. An example of an implicit rule is as follows::

   CLICON_PIPETREE="|mypipe";
   print, print_cb("all");{
      all, print_cb("all");
      detail, print_cb("detail);
   }

where the pipe tree is added implicitly to all commands in that file, and possibly on other files with the same mode.

Pipe trees also work for sub-trees, ie a subtree referenced by the top-level tree may also use output pipes.

Combinations
------------
It is possible to combine an implicit (default) rule with an explict rule as follows::

   CLICON_MODE="|commonpipe";
   print, print_cb("all");{
      @|mypipe, print_cb("all");
      all @|mypipe, print_cb("all");
      detail;
   }

In this example, `print` and `print all` use the `¡mypipe` menu, while `print detail` uses the `|common` menu

Inheriting
----------
Sub-trees inherit pipe commands from the top-level according to the following rules:
  1. Top-level implicit rules are inherited to all sub-trees, unless
  2. Explicit rules are present at the tree-reference
  3. No pipe commands are allowed in a pipe-command (only single level allowed)

Rules 1 and 2 are illustrated as follows::

   CLICON_MODE="|commonpipe";
   aaa {
      @datamodel, cli_show();
      @|mypipe, cli_show();
   }
   bbb {
      @datamodel, cli_show();
   }

Pipe commands in the `datamodel` tree are `|mypipe` if preceeded by `aaa`, but `|commonpipe` if preceeded by `bbb`

Generic pipe functions
----------------------

You can write own scripts and use them as output pipe functions by placing them in `CLICON_CLI_PIPE_DIR`.

Assume for example that ``CLICON_CLI_PIPE_DIR=/usr/local/lib/controller/pipes``, and that a script `first.sh` is placed in that dir, with the following content::

  #!/usr/bin/env bash
  head -1

Then add a pipe command  as follows::

   generic("Generic callbacks") <callback:string expand_dir("/usr/local/lib/controller/pipes", "\.sh$")>("Callback"), pipe_generic("callback");

Then, start the CLI::

  cli> show config | generic ?
  first.sh
  cli> show config | generic first.sh
  <config>
  cli>

CLI aliases
===========
Clixon provides a simple API for creating CLI aliases.

Three clispec utility functions exists that create and show aliases as follows:

1. `cli_alias_add` : Add a constant alias command by specifying name and command
2. `cli_aliasref_add` : Add an alias command by using completion
3. `cli_alias_show` : Show aliases

An example clispec using these three functions::

   alias("Define alias function") <name:string>("Name of alias") <command:rest>("Alias commands"), cli_alias_add("name", "command");
   aliasref("Define alias function using completion") <name:string>("Name of alias") @example, cli_aliasref_add("name");
   show("Show a particular state of the system") alias, cli_alias_show();

Example of setting a CLI alias, invoking it, and showing its definition::

  cli> alias cmd show config
  cli> cmd
    <table>
       <parameter/>
    </table>
  cli> show alias
    cmd: show config
  cli>

Restrictions
------------
The alias API is limited and an application developer may need to modify or extend these functions when integrating aliases into a system. These cases include:

1. Saving aliases persistently, eg an src file.
2. Adding help-texts
3. Mode/tree support

Autocli
=======
The Clixon CLI contains parts that are *generated* from a YANG
specification. This *autocli* is generated from YANG into CLI specifications,
parsed and merged into the top-level Clixon CLI.

The autocli is configured using three basic mechanisms:

1. `Config file`_ : Modify behavior of the generated tree
2. `Tree expansion`_: How the generated cli is merged into the overall CLI
3. `YANG Extensions`_: Modify CLI behavior via YANG

Each mechanism is described in sub-sections below, but first an overview of autocli usage.

Overview
--------
Consider a (simplified) YANG specification, such as::

  module example {
    container table {
      list parameter{
        key name;
        leaf name{
          type string;
        }
      }
    }
  }

An example of a generated syntax is as follows (again simplified)::

   table; {
      parameter <name:string>;
   }

The auto-cli syntax is loaded using a `sub-tree operator`_ such as ``@datamodel`` into the Clixon CLI as follows::

  CLICON_PROMPT="%U@%H %W> ";
  set @datamodel, cli_auto_set();
  merge @datamodel, cli_auto_merge();
  delete @datamodel, @add:leafref-no-refer, cli_auto_del();
  show config, cli_auto_show("datamodel", "candidate", "text", true, false);{
     @datamodel, cli_show_auto("candidate", "text");
  }

For example, the `set` part is expanded using the CLIgen tree-operator to something like::

  set table, cli_auto_set(); {
        parameter <name:string>, cli_auto_set();
  }

An example run of the above example is as follows::

  > clixon_cli
  user@host /> set table ?
    <cr>
    parameter
  user@host /> set table parameter 23
  user@host /> show config
  table {
     parameter {
        name 23;
     }
  }
  user@host />

where the generated autocli extends the Clixon CLI with YANG-derived configuration statements.

NETCONF operations
------------------
The autocli `set/merge/delete` commands are modelled after NETCONF operations as defined in the `NETCONF RFC <https://datatracker.ietf.org/doc/html/rfc6241#section-7.2>`_, with the following (shortened) definition, and mapping to the autocli operations above):

* `merge`: (Autocli `merge`) The configuration data is merged with the configuration in the configuration datastore
* `replace`: (Autocli `set`) The configuration data replaces any related configuration in the configuration datastore.
* `create`: The configuration data is added to the configuration if and only if the configuration data does not already exist in the configuration datastore.
* `delete`: The configuration data is deleted from the configuration if and only if the configuration data currently exists in the configuration datastore.
* `remove`: (Autocli `delete`) The configuration data is deleted from the configuration if the configuration data currently exists in the configuration datastore.  If the configuration data does not exist, the "remove" operation is silently ignored

In particular, the autocli `set` operation may cause some
confusion. For terminals, i.e., CLI commands derived from YANG leaf or
leaf-list, the behavior of "replace/set" and "merge" are
identical. However for non-terminals (i.e., CLI commands derived from
YANG container or list) "replace/set" and "merge" differ: "set"
replaces the existing configuration. "Merge" merges the existing
configuration.

Example, where x is (derived from) a container and y is (derived from) a leaf::

  set x y 22
  set x y 24      # Replace y: y changes value to 24
  set x           # Replace x: y is removed
  merge x y 26
  merge x y 28    # Merge y: y changes value to 28
  merge x         # Merge x: y still has value 28

Therefore, most users may want to use `merge` as default autocli operation, instead of `set`.

.. note::
        Use autocli `merge` as default operation

Config file
-----------
The clixon config file has a ``<autocli>`` sub-clause for global
autocli configurations.  A typical CLI configuration
with default autocli settings is as follows::

  <clixon-config xmlns="http://clicon.org/config">
    <CLICON_CONFIGFILE>/usr/local/etc/clixon/example.xml</CLICON_CONFIGFILE>
    ...
    <autocli>
      <module-default>true</module-default>
      <list-keyword-default>kw-nokey</list-keyword-default>
      <treeref-state-default>false</treeref-state-default>
      <edit-mode-default>list container</edit-mode-default>
      <completion-default>true</completion-default>
    </autocli>
  </clixon-config>

The autocli configuration consists of a set of default *options*, followed by a set of *rules*. For more info see the ``clixon-autocli.yang`` specification.

Options
^^^^^^^
The following options set default values to the auto-cli, some of these may be further refined by successive rules.

`module-default`
   How to generate the autocli from modules:

   - If `true`, all modules with a top-level datanode are generated, ie they get a top-level entry in the ``@basemodel`` tree. You can explicitly disable modules. This is default
   - If `false`, you need to explicitly enable modules for autocli generation  using `module enable rules`_.

`list-keyword-default`
   How to generate the autocli from YANG lists.
   There are several variants defined. To understand the different variants, consider a simple YANG LIST defintion as follows::

      list a {
         key x;
	 leaf x;
	 leaf y;
      }

   The different variants with the resulting autocli are as follows:

   - `kw-none` : No extra keywords, only variables: ``a <x> <y>``
   - `kw-nokey` : Keywords on non-key variables: ``a <x> y <y>``. This is default.
   - `kw-all` : Keywords on all variables: ``a x <x> y <y>``

`treeref-state-default`
   If generate autocli from YANG *state* data. The motivation for this option is that many specs have very large state parts. In particular, some openconfig YANG specifications have  ca 10 times larger state than config parts.

   - If `true`, generate CLI from YANG state/non-config statements, not only from config data.
   - If `false` do not generate autocli commands from YANG state data. This is default.

`edit-mode-default`
   Open automatic edit-modes for some YANG keywords and do not allow others.
   A CLI edit mode opens a carriage-return option and changes the context to be
   in that local context.
   For example::

      user@host> interfaces interface e0<cr>
      eth0>

   Default is to generate edit-modes for all YANG containers and lists. For more info see `edit modes`_

`completion-default`
   Generate code for CLI completion of existing db symbols.
   That is, check existing configure database for completion options.
   This is normally always enabled.

`grouping-treeref`
   Controls the behaviour when generating CLISPEC of YANG `uses` statements into the
   corresponding `grouping` definition. If `true`, use indirect tree reference ``@treeref``
   to reference the grouping definition. This may reduces memory footprint of the CLI.

`clispec-cache`
   Autocli cache mode for saving generated autocli clispecs between runs.
   Set to `readwrite` to get a dynamic cache behavior.

`clispec-cache-dir`
   Directory for generated clispecs. Directory is created if it does not exist.
   The cli client must have read/write access to this directory.
   The directory is expanded, ie can be relative and include tilde.

Rules
^^^^^
To complement options, a set of rules to further define the autocli can be defined.
Common rule fields are:

`name`
   Arbitrary name assigned for the rule, must be unique.

`operation`
   Rule operation, There are currently two operations defined: `module enable` and command `compress`.

`module-name`
   Name of the module associated with this rule.
   Wildchars '*' and '?' can be used (glob pattern).
   Revision and yang suffix are omitted.
   Example: ``openconfig-*``

Module enable rules
^^^^^^^^^^^^^^^^^^^
Module enable rules are used in combination with
``module-default=false`` to enable CLI generation for a limited set of
YANG modules.

For example, assume you want to enable modules `example1`, `example2` and no others::

   <autocli>
      <module-default>false</module-default>
      <rule>
         <name>include example</name>
         <operation>enable</operation>
         <module-name>example*</module-name>
     </rule>
   </autocli>

If the option ``module-default`` is ``true``, module enable rules have no effect since all modules are already enabled.

You can also disable all modules by default, and enable them individually, like the following example shows::

   <autocli>
      <module-default>true</module-default>
      <rule>
         <name>exclude example 2</name>
         <operation>disable</operation>
         <module-name>example2</module-name>
     </rule>
   </autocli>

Likewise, ``disable`` rules have no effect if ``module-default`` is ``false``.

Compress rules
^^^^^^^^^^^^^^
Compress rules are used to skip CLI commands, making the complete command name shorter.

For example, assume YANG definition::

   container interfaces {
      list interface {
         ...
      }
   }

Instead of typing ``interfaces interface e0`` you would want to type only ``interface e0``.
The following rule matches all YANG containers with lists as its only child, and removes the keyword ``interfaces``::

   <rule>
      <name>compress</name>
      <operation>compress</operation>
      <yang-keyword>container</yang-keyword>
      <yang-keyword-child>list</yang-keyword-child>
   </rule>

Note that this matches the openconfig compress rule: `The surrounding 'container' entities are removed from 'list' nodes <https://github.com/openconfig/ygot/blob/master/docs/design.md#openconfig-path-compression>`_

A second openconfig compress rule is `The 'config' and 'state' containers are "compressed" out of the schema. <https://github.com/openconfig/ygot/blob/master/docs/design.md#openconfig-path-compression>`_ as examplified here (for 'config' only)::

   <rule>
      <name>openconfig compress</name>
      <operation>compress</operation>
      <yang-keyword>container</yang-keyword>
      <schema-nodeid>config</schema-nodeid>
      <module-name>openconfig*</module-name>
   </rule>

Specific fields for compress are:

`yang-keyword`
   If present identifes a YANG keyword which the rule applies to.
   Example: ``container``

`schema-nodeid`
   A single <id> identifying a YANG schema-node identifier as defined in RFC 7950 Sec 6.5.
   Example: ``config``

`yang-keyword-child`
   The YANG statement has a single child, and the yang type of the child is the value of this option.
   Example: : ``container``

`extension`
   The extension is set either in the node itself, or in the module the node belongs to.
   Extension prefix must be set.
   Example: ``oc-ext:openconfig-version``

Tree expansion
--------------
In the example above, the tree-reference ``@datamodel`` is used to
merge the YANG-generated cli-spec into the overall cli-spec. There are
several variants of how the generated tree is expanded with slight differences in which
symbols are shown, how completion works, etc.

They are all derivates of the basic ``@basemodel`` tree.  The following tree variants are
defined:

* ``@basemodel`` - The most basic tree including everything
* ``@datamodel`` - The most common tree for configuration with state
* ``@datamodelshow`` - A tree made for showing configuration syntax
* ``@datamodelmode`` - A tree for editing modes
* ``@datamodelstate`` - A tree for showing state as well as configuration

Note to use ``@datamodelstate`` config option ``treeref-state-default`` must be set.

YANG Extensions
---------------
A third method to define the autocli is using :ref:`YANG extensions<clixon_yang>`, where a YANG specification is annotated with extension.

Clixon provides a dedicated YANG extension for the autocli for this purpose: ``clixon-lib:autocli``.

The following example shows the main example usage of the "hide" extension of the "hidden" leaf::

   import clixon-autocli{
      prefix autocli;
   }
   container table{
      list parameter{
         ...
         leaf hidden{
            type string;
            autocli:hide;
         }
      }
   }

The CLI ``hidden`` command is not shown but the command still exists::

  cli /> set table parameter a ?
  value
  <cr>
  cli /> set table parameter a hidden 99
  cli /> show configuration
  table {
    parameter {
        name a;
        hidden 99;
    }
  }

The following autocli extensions are defined:

``hide``
   Do not show the command in eg auto-completion. This was primarily intended for operational commands such as ``start shell`` but is this context used for hiding commands generated from the associated YANG node.
``hide-show``
   Do not show the config in show configuration commands. However, retreiving a config via NETCONF or examining the datastore directly shows the hidden configure commands.
``skip``
   Skip the command altogether.
``strict-expand``
   Only show exactly the expanded options of a variable. It shuld not be possible to add a *new* value that is not in the expanded list.o
``alias``
   Replace the command with another value, only implemented for YANG leaves.

Edit modes
----------
The autocli supports *automatic edit modes* where by entering a ``<cr>``, you enter an edit mode. An edit mode is created for every YANG container or list.

For example, the example YANG previously given and the following cli-spec::

   edit @datamodelmode, cli_auto_edit("basemodel");
   up, cli_auto_up("basemodel");
   top, cli_auto_top("basemodel");
   set @datamodel, cli_auto_set();

Then an example session for illustration is as follows, where first a small config is created, then a list instance mode is entered(``parameter a``), a value changed, and a container mode (``table``)::

  user@host /> set table parameter a value 42
  user@host /> set table parameter b value 77
  user@host /> edit table parameter a
  user@host parameter=a/>
  user@host parameter=a/> show configuration
    name a;
    value 42;
  user@host parameter=a/> set value 99
  user@host parameter=a/> up
  user@host table> show configuration
  parameter {
      name a;
      value 99;
  }
  parameter {
      name b;
      value 77;
  }
  user@host table> top
  user@host />

Advanced
========
This section describes some advanced options in the Clixon CLI not described elsewhere.

Backend socket
--------------
By default, the CLI uses a UNIX socket as an IPC to communicate with
the backend.  It is possible to use an IP socket but with a restricted functionality, see :ref:`backend section<clixon_backend>`.

Start session
^^^^^^^^^^^^^
The session creation is "lazy" in the sense that a NETCONF session is
only established when needed. After the session has been
established, it is maintained (cached) by the CLI client to keep track
of candidate edits and locks, as described in 7.5 of `RFC 6241 <https://www.rfc-editor.org/rfc/rfc6241.html>`_.

If there is no backend running at the time of session establishment, a warning is printed::

  cli /> show config
  Mar 18 11:53:43: clicon_rpc_connect_unix: 541: Protocol error: /usr/local/var/example/example.sock: config daemon not running?: No such file or directory
  Protocol error: /usr/local/var/example/example.sock: config daemon not running?: No such file or directory
  cli />

If at a later time, the backend is started, the session is established normally

Close session
^^^^^^^^^^^^^
After a session is established and the *backend* exits, crashes or restarts, any
state associated with the session will be lost, including:

* explicit locks
* edits in candidate-db

If the backend exits during an existing session, it will close with the same error message as above::

  cli /> show config
  Mar 18 11:53:43: clicon_rpc_connect_unix: 541: Protocol error: /usr/local/var/example/example.sock: config daemon not running?: No such file or directory
  Protocol error: /usr/local/var/example/example.sock: config daemon not running?: No such file or directory
  cli />

If the backend restarts, a new session is created with a warning::

  cli /> show configuration
  Mar 18 11:57:55: The backend was probably restarted and the client has reconnected to the backend. Any locks or candidate edits are lost.
  cli />

Alternative
^^^^^^^^^^^
It is possible to change the default behavior by undefining the compile-option: `#undef PROTO_RESTART_RECONNECT`. If so, the CLI is exited when the existing session is closed in anyway::

  cli /> show configuration
  Mar 18 12:02:57: clicon_rpc_msg: 210: Protocol error: Unexpected close of CLICON_SOCK. Clixon backend daemon may have crashed.: Cannot send after transport endpoint shutdown
  Protocol error: Unexpected close of CLICON_SOCK. Clixon backend daemon may have crashed.: Cannot send after transport endpoint shutdown
  bash#

Sub-tree operator
-----------------
Sub-trees are defined using the tree operator `@`. Every mode gets assigned a tree which can be referenced as `@name`. This tree can be either on the top-level or as a sub-tree. For example, create a specific sub-tree that is used as sub-trees in other modes::

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

  user@host> translate HAL
  user@host> show configuration
  table {
     parameter {
        name translate;
        value IBM;
     }
  }

The example is very simple and based on strings, but can be used also for other types and more advanced functions.

Autocli tree labels
-------------------
The autocli trees described in `tree expansion`_ are implemented using filtering of CLIgen
labels. While ``@basemodel`` includes all labels, the other trees have
removed some labels.

For most uses, the pre-defined trees above are enough, using explicit label filtering is more powerful.

The currently defined labels are:

* ``act-list``      : Terminal entries of YANG LIST nodes.
* ``act-container`` : Terminal entries of YANG CONTAINER nodes.
* ``ac-leaf``       : Leaf/leaf-list nodes
* ``act-prekey``    : Terminal entries of LIST leaf keys, except the last keys in multi-key cases.
* ``act-lastkey``   : Terminal entries of LIST leaf keys, except the last keys in multi-key cases.
* ``act-leafconst`` : Terminal entries of non-empty non-key YANG LEAF/LEAF_LISTs command nodes.
* ``act-leafvar``   : Terminal entries of non-key YANG LEAF/LEAF_LISTs variable nodes.
* ``ac-state``      : Nodes which have YANG ``config false`` as child
* ``ac-config``     : Nodes nodes which do not have any state nodes as siblings

Labels with prefix ``act_`` are *terminal* labels in the sense that they mark a terminal command, ie the node itself; while labels with ``ac_`` represent the non-terminal, ie the whole sub-tree.

As an example, the ``@datamodel`` tree is ``basemodel`` with labels removed as follows::

   @basemodel, @remove:act-prekey, @remove:act-list, @remove:act-leaf, @remove:ac-state;

which is an alternative way of specifying the datamodel tree.

Extensions to CLIgen
--------------------
Clixon adds some features and structure to CLIgen which include:

- A plugin framework for both textual CLI specifications(.cli) and object files (.so)
- Object files contains compiled C functions referenced by callbacks in the CLI specification. For example, in the cli spec command: `a,fn()`, `fn` must exist in the object file as a C function.
- The CLIgen `treename` syntax does not work.
- A CLI specification file is enhanced with the CLIgen variables `CLICON_MODE`, `CLICON_PROMPT`, `CLICON_PLUGIN` and `CLICON_PIPETREE`.
- Clixon generates a command syntax from the Yang specification that can be referenced as `@datamodel`. This is useful if you do not want to hand-craft CLI syntax for configuration syntax.

Example of `@datamodel` syntax:
::

  set    @datamodel, cli_set();
  merge  @datamodel, cli_merge();
  create @datamodel, cli_create();
  show   @datamodel, cli_show_auto("running", "xml");

The commands (eg `cli_set`) will be called with the first argument an api-path to the referenced object.

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

Two caveats regarding "shebang":
  1. The clixon config file is `/usr/local/etc/clixon.xml`
  2. The mode is `CLICON_CLI_MODE`

You may mod this by using soft links or creating a new executable to use use in the "shebang" with other default values.

How to deal with large specs
----------------------------
CLIgen is designed to handle large specifications in runtime, but it may be
difficult to handle large specifications from a design perspective.

Here are some techniques and hints on how to reduce the complexity of large CLI specs:

Sub-modes
^^^^^^^^^
The `CLICON_MODE` is used to specify in which modes the syntax in a specific file should be added. For example, if you have major modes `configure` and `operation` you can have a file with commands for only that mode, or files with commands in both, (or in all).

First, lets add a basic set in each::

  CLICON_MODE="configure";
  show configure;

and::

  CLICON_MODE="operation";
  show configure;

Note that CLI command trees are *merged* so that show commands in other files are shown together. Thus, for example:
::

  CLICON_MODE="operation:files";
  show("Show") files("files");

will result in both commands in the operation mode:
::

  > clixon_cli -m operation
  user@host> show <TAB>
    configure      files

but::

  > clixon_cli -m configure
  user@host> show <TAB>
    configure

Sub-trees
^^^^^^^^^
You can also use sub-trees and the the tree operator `@`. Every mode gets assigned a tree which can be referenced as `@name`. This tree can be either on the top-level or as a sub-tree. For example, create a specific sub-tree that is used as sub-trees in other modes::

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

C-preprocessor
^^^^^^^^^^^^^^
You can also add the C preprocessor as a first step. You can then define macros, include files, etc. Here is an example of a Makefile using cpp::

   C_CPP    = clispec_example1.cpp clispec_example2.cpp
   C_CLI    = $(C_CPP:.cpp=.cli
   CLIS     = $(C_CLI)
   all:     $(CLIS)
   %.cli : %.cpp
        $(CPP) -P -x assembler-with-cpp $(INCLUDES) -o $@ $<

Bits
----
The Yang bits built-in type as defined in RFC 7950 Sec 9.7 provides a set of bit names. In the CLI, the names should be given in a white-spaced delimited list, such as ``"fin syn rst"``.

The RFC defines a "canonical form" where the bits appear ordered by their position in YANG, but Clixon validation accepts them in any order.

Given them in XML and JSON follows thus, eg XML::

   <flags>fin rst syn</flags>

Clixon CLI does not treat individual bits as "first-level objects". Instead it only validates the whole string of bit names. Operations (add/remove) are made atomically on the whole string.

Api-path-fmt
------------
The clixon CLI uses an internal meta-format called ``api_path_fmt`` which is used to generate api-paths, as described in Section :ref:`XML <clixon_xml>`.

An api-path-fmt extends an api-path with ``%`` flag characters (like ``printf``) as follows:

* %s: The value of a cligen-variable
* %k: The key of a YANG list

Example, an explicit clispec expansion variable could be::

  <name:string expand_dbvar("candidate","/interface=%s/%k")>

which could expand to ``/interface=eth0/mykey`` if "eth0" is given as the "name" variable and "mykey" is the YANG interface list key.
