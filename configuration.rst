.. _clixon_configuration:

Configuration
=============

Clixon configuration files are XML files modeled by YANG. By
default, the main config file is installed in ``/usr/local/etc/clixon.xml``, but can be changed by the ``-f <file>`` command-line option.

The YANG specification for Clixon configuration is `clixon-config.yang
<https://github.com/clicon/clixon/blob/master/yang/clixon/clixon-config.yang>`_. This configuration file is updated regularly.

Normally, all Clixon processes (backend, cli, netconf, restconf) use
the same configuration, although some options are not valid for all
processes.

Please consult the `YANG spec <https://github.com/clicon/clixon/blob/master/yang/clixon/clixon-config.yang>`_ directly if you want detailed description of config options.

Example
-------

The following is the configuration file of an example:
::
   
   <clixon-config xmlns="http://clicon.org/config">
     <CLICON_CONFIGFILE>/usr/local/etc/clixon.xml</CLICON_CONFIGFILE>
     <CLICON_CONFIGDIR>/usr/local/etc/clixon.d</CLICON_CONFIGDIR>
     <CLICON_FEATURE>*:*</CLICON_FEATURE>
     <CLICON_YANG_DIR>/usr/local/share/clixon</CLICON_YANG_DIR>
     <CLICON_YANG_MODULE_MAIN>clixon-hello</CLICON_YANG_MODULE_MAIN>
     <CLICON_CLI_MODE>hello</CLICON_CLI_MODE>
     <CLICON_CLISPEC_DIR>/usr/local/lib/hello/clispec</CLICON_CLISPEC_DIR>
     <CLICON_SOCK>/usr/local/var/hello.sock</CLICON_SOCK>
     <CLICON_BACKEND_PIDFILE>/usr/local/var/hello.pidfile</CLICON_BACKEND_PIDFILE>
     <CLICON_XMLDB_DIR>/usr/local/var/hello</CLICON_XMLDB_DIR>
     <CLICON_STARTUP_MODE>init</CLICON_STARTUP_MODE>
     <CLICON_MODULE_LIBRARY_RFC7895>false</CLICON_MODULE_LIBRARY_RFC7895>
     <restconf>
        <enable>true</enable>
     </restconf>
   </clixon-config>

The option ``CLICON_CONFIGFILE`` is special, it must be available before
the configuration file is found (see `Loading the configuration`_),
which means that the value in the file is a no-op.

The ``restconf`` clause defines RESTCONF configuration options as described in the :ref:`clixon_restconf` section.

Loading the configuration
-------------------------

There are several ways to manage how Clixon finds its configuration files, in priority order:

  1. Start a clixon program with the ``-f <FILE>`` option. For example::

	clixon_backend -f FILE

  2. At install time, Use the ``--with-configfile=FILE`` option to configure a default location::

	./configure -f FILE

  3. At install time: ``./configure --with-sysconfig=<dir>`` when configuring. Then FILE is ``<dir>/clixon.xml``
  4. At install time: ``./configure --sysconfig=<dir>`` when configuring. Then FILE is `<dir>/etc/clixon.xml`
  5. If none of the above: FILE is ``/usr/local/etc/clixon.xml``

The following options control the Clixon configuration:

``CLICON_CONFIGFILE``
   The configure file itself. Due to bootstrapping reasons, its value is meaningless in a file but can be useful for documentation purposes.
``CLICON_CONFIGDIR``
   A directory of extra configuration files loaded after the main configuration. It can alsp be specified using the ``-E <dir>`` command-line option. These extra configuration files are read in alphabetical order after the main configuration as follows:

   * leaf values are overwritten
   * leaf-list values are appended
   

Runtime modification
--------------------

You can modify clixon options at runtime by using the `-o` option to
modify the configuration specified in the configuration file. For
example, add `usr/local/share/ietf` to the list of directories where yang files are searched for::

  clixon_cli -o CLICON_YANG_DIR=/usr/local/share/ietf

Features
--------
`CLICON_FEATURE` is a list of values, describing how Clixon supports feature.

A value is specified as one of the following:

- `<module>:<feature>` : enable a specific feature in a specific module
- `*:*` : enable all features in all modules
- `<module>:*` : enable all features in the specified module
- `*:<feature>` : enable the specific feature in all modules.

Example:: 

      <CLICON_FEATURE>ietf-netconf:startup</CLICON_FEATURE>
      <CLICON_FEATURE>ietf-netconf:*</CLICON_FEATURE>
      <CLICON_FEATURE>*:*</CLICON_FEATURE>
      
Supplying the `-o` option adds a value to the feature list.
      
Clixon have three hardcoded features:

- ietf-netconf:candidate (RFC6241 8.3)
- ietf-netconf:validate (RFC6241 8.6)
- ietf-netconf:xpath (RFC6241 8.9)


Finding YANG files
------------------
The example have two options for finding Yang files::
   
     <CLICON_YANG_DIR>/usr/local/share/clixon</CLICON_YANG_DIR>
     <CLICON_YANG_MODULE_MAIN>clixon-hello</CLICON_YANG_MODULE_MAIN>
     
which means that Yang files are searched for in ``/usr/local/share/clixon`` and that module ``clixon-hello`` is loaded. Note:

- ``clixon-hello.yang`` must be present in ``/usr/local/share/clixon``
- Clixon itself may load several YANG files as part of the system startup, such as ``clixon-config.yang``. These must all reside in the list of ``CLICON_YANG_DIR``:s.
- When a Yang file is loaded, it may contain references to other Yang files (eg using ``import`` and ``include``). They must also be found in the list of ``CLICON_YANG_DIR``:s.

The following configuration file options control the loading of Yang files:

``CLICON_YANG_DIR``
   A list of directories (yang dir path) where Clixon searches for module and submodules.
``CLICON_YANG_MAIN_FILE``
   Load a specific Yang module given by a file. 
``CLICON_YANG_MODULE_MAIN``
   Specifies a single module to load. The module is searched for in the yang dir path.
``CLICON_YANG_MODULE_REVISION``
   Specifies a revision to the main module. 
``CLICON_YANG_MAIN_DIR``
   Load all yang modules in this directory.

Note that the special ``YANG_INSTALLDIR`` autoconf configure option, by default ``/usr/local/share/clixon`` should be included in the yang dir path for Clixon system files to be found.

You can combine the options, however, if there are different variants
of the same module, more specific options override less
specific. The precedence of the options are as follows:

1. ``CLICON_YANG_MAIN_FILE``
2. ``CLICON_YANG_MODULE_MAIN``
3. ``CLICON_YANG_MAIN_DIR``

Note that using ``CLICON_YANG_MAIN_DIR`` Clixon may find several files
containing the same Yang module. Clixon will prefer the one without a
revision date if such a file exists. If no file has a revision date,
Clixon will prefer the newest.

Default values
--------------

``CLICON_YANG_REGEXP`` which is not present in the ``hello world`` is an example of a configuration option with a default value of ``posix``::

   <CLICON_YANG_REGEXP>posix</CLICON_YANG_REGEXP>
