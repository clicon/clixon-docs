Clixon and ConfD Examples
-------------------------

This section will describe how to create a minimal YANG specification
and use it together with both Clixon and ConfD. We will start with
adding a the new YANG model and feed Clixon and ConfD with some
NETCONF data which we later should be able to see in the respective
CLI:s.

This guide will assume that you have Clixon and/or ConfD installed and
running and we refer to their respective documentation for guideance
about installation and initial configuration.

YANG model
^^^^^^^^^^

We will start off with a minimal YANG model. The model consist of a
table with a list of parameters where each parameter have a name and a
value.

::

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

We will save the YANG model as example.yang

Installating the YANG model
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Clixon and ConfD have different ways of adding new models.

**Clixon:**

Here we assume that you have built and installed the example
application shipped with Clixon. If not it can be found in the source
tree under "example/".

In Clixon we can simply copy the file to the folder where Clixon expect
to find YANG models (ususally "/usr/local/share/clixon/"):

::

   $ sudo cp example.yang /usr/local/share/clixon/example.yang

We will also modify Clixons configuration, in our case
"/usr/local/etc/example.xml" to look something like this:

::

  <clixon-config xmlns="http://clicon.org/config">
    <CLICON_CONFIGFILE>/usr/local/etc/example.xml</CLICON_CONFIGFILE>
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
    <CLICON_CLI_GENMODEL_COMPLETION>1</CLICON_CLI_GENMODEL_COMPLETION>
    <CLICON_CLI_GENMODEL_TYPE>VARS</CLICON_CLI_GENMODEL_TYPE>
    <CLICON_XMLDB_DIR>/usr/local/var/example</CLICON_XMLDB_DIR>
    <CLICON_CLI_LINESCROLLING>0</CLICON_CLI_LINESCROLLING>
    <CLICON_STARTUP_MODE>init</CLICON_STARTUP_MODE>
    <CLICON_NACM_MODE>disabled</CLICON_NACM_MODE>
    <CLICON_STREAM_DISCOVERY_RFC5277>true</CLICON_STREAM_DISCOVERY_RFC5277>
    <CLICON_MODULE_LIBRARY_RFC7895>false</CLICON_MODULE_LIBRARY_RFC7895>
  </clixon-config>

After this is done, we can restart Clixons backend and the new model
should be present.
  
**ConfD:**

For ConfD it is slightly different. First we must compile
the model to ConfDs own FXS format and then move the FXS file to the
directory where ConfD can expect to find it and then restart ConfD.

To convert from YANG to FXS we use the "confdc" command:

::

   $ confdc -c example.yang

And then move the file:

::
   
   $ sudo cp example.fxs /opt/confd/etc/confd/

And finally after restarting ConfD the new model should be installed.


Testing with NETCONF
^^^^^^^^^^^^^^^^^^^^

To test all this we can use NETCONF. We will try to set a new
parameter with the name "test" and value 1234.

I'm running NETCONF over SSH, and to achieve this for both Clixon and
ConfD on the same machine I altered the SSH configuration to contain
the following lines:

::

  Subsystem confd_netconf /usr/local/bin/netconf-subsys
  Subsystem clixon_netconf /usr/local/bin/clixon_netconf -f /usr/local/etc/example.xml


We will use the XML below for both Clixon and ConfD, since it is
NETCONF it should work in the same way regardless of using Clixon or ConfD:

::

  <?xml version="1.0" encoding="UTF-8"?>
  <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <capabilities>
      <capability>urn:ietf:params:netconf:base:1.0</capability>
    </capabilities>
  </hello>
  ]]>]]>
  
  <?xml version="1.0" encoding="UTF-8"?>
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
  
  <?xml version="1.0" encoding="UTF-8"?>
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="2">
    <close-session/>
  </rpc>
  ]]>]]>

I save the XML as "example.xml" and use the following commands to test it:

::

   $ ssh -s 192.168.1.56 clixon_netconf < example.xml

And:

::

   $ ssh -s 192.168.1.56 confd_netconf < example.xml


If everything went fine, we should get a reply back saying that
everything went fine. We can ignore everything in the reply except for
the reply of the two messages:

::
  
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="1">
    <ok/>
  </rpc-reply>
  ]]>]]>
  
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="2">
    <ok/>
  </rpc-reply>
  ]]>]]>


And finally, we can validate from each of the CLIs that the configuration is available:

::

  root@debian10-clixon /> show configuration
  example {
      parameter {
          name test;
          value 1234;
      }
  }
