.. _clixon_configuration:

Configuration
=============

Clixon options are stored in an XML configuration file. The default
configuration file is installed in /usr/local/etc/clixon.xml. The example
configuration file is installed at /usr/local/etc/example.xml. The
YANG specification for the configuration file is clixon-config.yang.

Clixon by default finds its configuration file at `/usr/local/etc/clixon.xml`. However, you can modify this location as follows:

  - Provide `-f <FILE>` option when starting a program (eg `clixon_backend -f FILE`)
  - Provide `--with-configfile=FILE` when configuring
  - Provide `--with-sysconfig=<dir>` when configuring. Then FILE is `<dir>/clixon.xml`
  - Provide `--sysconfig=<dir>` when configuring. Then FILE is `<dir>/etc/clixon.xml`
  - If none of the above: FILE is `/usr/local/etc/clixon.xml`

You can modify clixon options at runtime by using the `-o` option to modify the configuration specified in the configuration file. Options that are leafs are overwritten, whereas options that are leaf-lists are added to.

Example, add the "usr/local/share/ietf" directory to the list of directories where yang files are searched for:
::

  clixon_cli -o CLICON_YANG_DIR=/usr/local/share/ietf

Yang files
----------
Yang files contain the configuration specification. A Clixon
application loads yang files and clixon itself loads system yang
files. When Yang files are loaded modules are imported and submodules
are included.

The following configuration file options control the loading of Yang files:

- `CLICON_YANG_DIR` -  A list of directories (yang dir path) where Clixon searches for module and submodules.
- `CLICON_YANG_MAIN_FILE` - Load a specific Yang module given by a file. 
- `CLICON_YANG_MODULE_MAIN` - Specifies a single module to load. The module is searched for in the yang dir path.
- `CLICON_YANG_MODULE_REVISION` : Specifies a revision to the main module. 
- `CLICON_YANG_MAIN_DIR` - Load all yang modules in this directory.

Note that the special `YANG_INSTALLDIR`, by default `/usr/local/share/clixon` should be included in the yang dir path for Clixon system files to be found.

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

Yang features
-------------
Yang models have features, and parts of a specification can be
conditional using the if-feature statement. In Clixon, features are
enabled in the configuration file using <CLICON_FEATURE>.

The example below shows enabling a specific feature; enabling all features in module; and enabling all features in all modules, respectively:
::
   
      <CLICON_FEATURE>ietf-routing:router-id</CLICON_FEATURE>
      <CLICON_FEATURE>ietf-routing:*</CLICON_FEATURE>
      <CLICON_FEATURE>*:*</CLICON_FEATURE>

Features can be probed by using RFC 7895 Yang module library which provides
information on all modules and which features are enabled.

Clixon have three hardcoded features:

- :candidate (RFC6241 8.3)
- :validate (RFC6241 8.6)
- :xpath (RFC6241 8.9)

You can select the startup feature by including it in the config file:
::

      <CLICON_FEATURE>ietf-netconf:startup</CLICON_FEATURE>

(or just `ietf-netconf:*`).
