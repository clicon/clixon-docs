.. _clixon_configuration:

Configuration
=============

The clixon configuration file is an XML file modelled by YANG. By
default, it is installed in `/usr/local/etc/clixon.xml`.  The YANG
specification for the configuration is `clixon-config.yang
<https://github.com/clicon/clixon/blob/master/yang/clixon/clixon-config%402019-06-05.yang>`_. All Clixon processes (backend, cli, netconf, restconf) use the same
config file, although some configuration options are not valid for all processes.

Please consult the YANG spec directly if you want detailed description of config options.

Example
-------

The following is the configuration file of the `hello world` example:
::
   
   <clixon-config xmlns="http://clicon.org/config">
     <CLICON_FEATURE>*:*</CLICON_FEATURE>
     <CLICON_CONFIGFILE>/usr/local/etc/example.xml</CLICON_CONFIGFILE>
     <CLICON_YANG_DIR>/usr/local/share/clixon</CLICON_YANG_DIR>
     <CLICON_YANG_MODULE_MAIN>clixon-hello</CLICON_YANG_MODULE_MAIN>
     <CLICON_CLI_MODE>hello</CLICON_CLI_MODE>
     <CLICON_CLISPEC_DIR>/usr/local/lib/hello/clispec</CLICON_CLISPEC_DIR>
     <CLICON_SOCK>/usr/local/var/hello.sock</CLICON_SOCK>
     <CLICON_BACKEND_PIDFILE>/usr/local/var/hello.pidfile</CLICON_BACKEND_PIDFILE>
     <CLICON_XMLDB_DIR>/usr/local/var/hello</CLICON_XMLDB_DIR>
     <CLICON_STARTUP_MODE>init</CLICON_STARTUP_MODE>
     <CLICON_MODULE_LIBRARY_RFC7895>false</CLICON_MODULE_LIBRARY_RFC7895>
   </clixon-config>

Specification
-------------
Some options (of approximately 50) described below of `clixon-config.yang <https://github.com/clicon/clixon/blob/master/yang/clixon/clixon-config%402019-06-05.yang>`_ are the following (descriptions are skipped):
::
   
    container clixon-config {
	leaf CLICON_CONFIGFILE{
	    type string;
	}
	leaf CLICON_YANG_MAIN_DIR {
	    type string;
	}
        leaf-list CLICON_FEATURE {
	   type string;
        }
	leaf CLICON_YANG_REGEXP {
	    type regexp_mode;
	    default posix;
	}

The option `CLICON_CONFIGFILE` is special, it must be available
before the configuration file is found (see `Finding the
configuration`_), which means that the value in the file is a no-op.

Finding the configuration
-------------------------

There are several ways to change where Clixon finds it config file, in priority order:

  - Provide `-f <FILE>` option when starting a program (eg `clixon_backend -f FILE`)
  - Provide `--with-configfile=FILE` when configuring
  - Provide `--with-sysconfig=<dir>` when configuring. Then FILE is `<dir>/clixon.xml`
  - Provide `--sysconfig=<dir>` when configuring. Then FILE is `<dir>/etc/clixon.xml`
  - If none of the above: FILE is `/usr/local/etc/clixon.xml`

Runtime modification
--------------------

You can modify clixon options at runtime by using the `-o` option to
modify the configuration specified in the configuration file. For
example, add `usr/local/share/ietf` to the list of directories where yang files are searched for:
::

  clixon_cli -o CLICON_YANG_DIR=/usr/local/share/ietf

Features
--------
`CLICON_FEATURE` is a list of values, describing how Clixon supports feature.
Clixon have three hardcoded features:

- ietf-netconf:candidate (RFC6241 8.3)
- ietf-netconf:validate (RFC6241 8.6)
- ietf-netconf:xpath (RFC6241 8.9)

Supplying the `-o` option will add a value to the list (not replace existing).  For example, if `-o CLICON_FEATURE=ietf-netconf:startup` is given at startup, the following options would be present in the configuration:
::
   
      <CLICON_FEATURE>ietf-netconf:startup</CLICON_FEATURE>
      <CLICON_FEATURE>*:*</CLICON_FEATURE>
      
(Which does not really make sense since `*:*` enables all features anayway.)


Finding YANG files
------------------
The example have two options for finding Yang files:
::
   
     <CLICON_YANG_DIR>/usr/local/share/clixon</CLICON_YANG_DIR>
     <CLICON_YANG_MODULE_MAIN>clixon-hello</CLICON_YANG_MODULE_MAIN>
     
which means that Yang files are searched for in
`/usr/local/share/clixon` and that the module `clixon-hello` should be
loaded. Note:

- `clixon-hello.yang` must be present in `/usr/local/share/clixon`
- Clixon itself may load several YANG files as part of the system startup, such as `clixon-config.yang`. These must all reside in the list of `CLICON_YANG_DIR`:s.
- When a Yang file is loaded, it may contain references to other Yang files (eg using `import` and `include`). They must also be found in the list of `CLICON_YANG_DIR`:s.

The following configuration file options control the loading of Yang files:

- `CLICON_YANG_DIR` -  A list of directories (yang dir path) where Clixon searches for module and submodules.
- `CLICON_YANG_MAIN_FILE` - Load a specific Yang module given by a file. 
- `CLICON_YANG_MODULE_MAIN` - Specifies a single module to load. The module is searched for in the yang dir path.
- `CLICON_YANG_MODULE_REVISION` : Specifies a revision to the main module. 
- `CLICON_YANG_MAIN_DIR` - Load all yang modules in this directory.

Note that the special `YANG_INSTALLDIR` autoconf configure option, by default `/usr/local/share/clixon` should be included in the yang dir path for Clixon system files to be found.

You can combine the options, however, if there are different variants
of the same module, more specific options override less
specific. The precedence of the options are as follows:

- `CLICON_YANG_MAIN_FILE`
- `CLICON_YANG_MODULE_MAIN`
- `CLICON_YANG_MAIN_DIR`

Note that using `CLICON_YANG_MAIN_DIR` Clixon may find several files
containing the same Yang module. Clixon will prefer the one without a
revision date if such a file exists. If no file has a revision date,
Clixon will prefer the newest.

Default values
--------------

`CLICON_YANG_REGEXP` which is not present in the `hello world` is an example of a configuration option with a default value of `posix`. More clearly, it could be provided in the file as a comment:
::

   <!--CLICON_YANG_REGEXP>posix</CLICON_YANG_REGEXP-->
