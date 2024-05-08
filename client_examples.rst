Clixon and ConfD Examples
=========================

This describes how to create a minimal YANG specification and use it
together with Clixon. First, a YANG model is added. Then, NETCONF
messages are sent to Clixon which is thereafter accessed in the Clixon CLI.

This guide assumes that you have Clixon installed and running. Please
refer to the respective for installation and initial configuration.

YANG model
----------
The YANG model consists of a table with a list of parameters where each
parameter have a name and a value::

  module example {
      yang-version 1.1;
      namespace "urn:example:clixon";
      description
        "Tiny example to be used with Clixon and ConfD.
           ";
      revision 2021-01-26 {
        description "Added example.";
      }

      container example{
        list parameter{
            key name;
            leaf name{
                type string;
            }
            leaf value{
                type string;
            }
        }
      }
  }

The YANG model is saved as `example.yang`.

Installating the YANG model
---------------------------

It is assumed that the example application shipped with Clixon is
installed. If not it can be found in the source tree under
"example/main".

The YANG file `example.yang` is copied to the folder where Clixon expects
to find YANG models (usually `/usr/local/share/clixon`)::

   $ sudo cp example.yang /usr/local/share/clixon/example.yang

The Clixons configuration should look something like::

  <clixon-config xmlns="http://clicon.org/config">
    <CLICON_CONFIGFILE>/usr/local/etc/clixon/example.xml</CLICON_CONFIGFILE>
    <CLICON_FEATURE>ietf-netconf:startup</CLICON_FEATURE>
    <CLICON_YANG_DIR>/usr/local/share/clixon</CLICON_YANG_DIR>
    <CLICON_YANG_MODULE_MAIN>example</CLICON_YANG_MODULE_MAIN>
    <CLICON_CLI_MODE>example</CLICON_CLI_MODE>
    <CLICON_BACKEND_DIR>/usr/local/lib/example/backend</CLICON_BACKEND_DIR>
    <CLICON_NETCONF_DIR>/usr/local/lib/example/netconf</CLICON_NETCONF_DIR>
    <CLICON_RESTCONF_DIR>/usr/local/lib/example/restconf</CLICON_RESTCONF_DIR>
    <CLICON_CLI_DIR>/usr/local/lib/example/cli</CLICON_CLI_DIR>
    <CLICON_CLISPEC_DIR>/usr/local/lib/example/clispec</CLICON_CLISPEC_DIR>
    <CLICON_SOCK>/usr/local/var/example/example.sock</CLICON_SOCK>
    <CLICON_BACKEND_PIDFILE>/usr/local/var/example/example.pidfile</CLICON_BACKEND_PIDFILE>
    <CLICON_XMLDB_DIR>/usr/local/var/example</CLICON_XMLDB_DIR>
    <CLICON_CLI_LINESCROLLING>0</CLICON_CLI_LINESCROLLING>
    <CLICON_STARTUP_MODE>init</CLICON_STARTUP_MODE>
    <CLICON_NACM_MODE>disabled</CLICON_NACM_MODE>
    <CLICON_STREAM_DISCOVERY_RFC5277>true</CLICON_STREAM_DISCOVERY_RFC5277>
  </clixon-config>

After this is done, the Clixon backend can be restarted,  and the new model should be present.
  

Testing with NETCONF
--------------------
The next step is to modify configuration values with NETCONF. A new
`test` parameter is added with value `1234`.

In the example, NETCONF is running over SSH. The SSH configuration needs to contain
the following line::

  Subsystem netconf /usr/local/bin/clixon_netconf -f /usr/local/etc/clixon/example.xml

The following NETCONF operation is used::

  <?xml version="1.0" encoding="UTF-8"?>
  <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <capabilities>
      <capability>urn:ietf:params:netconf:base:1.1</capability>
    </capabilities>
  </hello>
  ]]>]]>
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
    <edit-config>
      <target>
        <running/>
      </target>
      <config>
        <table xmlns="urn:example:clixon">
          <parameter>
            <name>test</name>
            <value>1234</value>
  	</parameter>
        </table>
      </config>
    </edit-config>
  </rpc>
  ]]>]]>
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="2">
    <commit/>
  </rpc>
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="2">
    <close-session/>
  </rpc>
  ]]>]]>

The XML is saved as "example.xml" and use the following commands to test it::

   $ ssh 192.168.1.56 -s netconf < example.xml

If everything went fine, a reply is returned saying OK::
  
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
    <ok/>
  </rpc-reply>
  ]]>]]>
  
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="2">
    <ok/>
  </rpc-reply>
  ]]>]]>


Finally, the config can be viewed from the CLI::

  root@debian10-clixon /> show configuration
  example {
      parameter {
          name test;
          value 1234;
      }
  }
