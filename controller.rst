.. _clixon_controller:
.. sectnum::
   :start: 15
   :depth: 3

**********
Controller
**********

.. note::
          The controller is still experimental

This section describes the clixon network controller, an extension of
Clixon to manage devices using NETCONF.

For information on developing network services, see also :ref:`Service development <clixon_services>`

Overview
========

Goals
-----
The Clixon network controller aims at providing a simple
network controller for NETCONF devices of different vendors, not only Clixon.

Further goals are:

- Programmable network services, with a Python API
- Multiple devices, with different YANG schemas using RFC 8528 yang schema mount
- Transactions with validate/commit/revert across groups of devices
- Scaling up to 100 devices.

Architecture
------------
.. image:: controller.jpg
   :width: 100%

The controller is built on the base of the `CLIgen/Clixon <https://clicon.org>`_ system, where
the controller semantics is implemented using plugins. The `backend`
is the core of the system controlling the datastores and accessing the
YANG models.

APIs
----
The `southbound API` uses only NETCONF over SSH to network
devices. There are no current plans to support other protocols for
device control.

The `northbound APIs` are YANG-derived Restconf, Autocli, Netconf, and
Snmp.  The controller CLI has two modes: operation and configure, with
an autocli configure mode derived from YANG.

A PyAPI module accesses configuration data via an `actions API`_. The
PyAPI module reads services configuration and writes device data. The
backend then pushes changes to the actual devices using a transaction
mechanism.

Installation
============
Packages
--------
Some packages are required. The following are example of debian-based packages::
  
  sudo apt install flex bison git make gcc libnghttp2-dev libssl-dev

  
Source
------
Check out the following GIT repos:

- `<https://github.com/clicon/cligen.git/>`_
- `<https://github.com/clicon/clixon.git/>`_
- `<https://github.com/clicon/clixon-controller.git/>`_
- `<https://github.com/clicon/clixon-pyapi.git/>`_

Building
--------
The source is built as follows.

Cligen
^^^^^^
::

  cd cligen
  ./configure
  make
  sudo make install

Clixon
^^^^^^
::
   
  cd clixon
  ./configure
  make
  sudo make install

Python API
^^^^^^^^^^
::

  # Build and install the package
  cd clixon-pyapi
  sudo -u clicon pip3 install -r requirements.txt
  sudo python3 setup.py install
  
Controller
^^^^^^^^^^
::
   
  cd clixon-controller
  ./configure
  make
  sudo make install
  sudo mkdir /usr/local/share/clixon/mounts/


Install
-------
Install the python code by copy::

  sudo cp clixon_server.py /usr/local/bin/

Add a new clicon user and install the needed Python packages,
the backend will start the Python server and drop the privileges
to this user::

  sudo useradd -g clicon -m clicon

Quick start
===========
Start example devices as containers::

  cd test
  ./start-devices.sh
  sudo ./copy-keys.sh

Start controller::

  sudo clixon_backend -f /usr/local/etc/controller.xml

Start the CLI and configure devices::

  clixon_cli -f /usr/local/etc/controller.xml -m configure
  set devices device clixon-example1 description "Clixon container"
  set devices device clixon-example1 conn-type NETCONF_SSH
  set devices device clixon-example1 addr 172.20.20.2
  set devices device clixon-example1 user root
  set devices device clixon-example1 enable true
  set devices device clixon-example1 yang-config VALIDATE
  set devices device clixon-example1 root
  commit local

Thereafter explicitly connect to the devices::

  clixon_cli -f /usr/local/etc/controller.xml
  connection open

Configuration
=============
The controller extends the clixon configuration file as follows:

``CLICON_CONFIG_EXTEND``
   The value should be `clixon-controller-config` making the controller-specific 

``CONTROLLER_ACTION_COMMAND``
   Should be set to the PyAPI binary with correct arguments
   The namespace is ="http://clicon.org/controller-config"

``CLICON_BACKEND_USER``
   Set to the user which the action binary (above) is used. Normally `clicon`

``CLICON_SOCK_GROUP``   
   Set to user group, ususally `clicon`

``CONTROLLER_PYAPI_MODULE_PATH``
   Path to Python code for PyAPI
   
``CONTROLLER_PYAPI_MODULE_FILTER``

``CONTROLLER_PYAPI_PIDFILE``
   
Example
-------
The following configuration file examplifies the configure options described above::

  <clixon-config xmlns="http://clicon.org/config">
  <CLICON_CONFIGFILE>/usr/local/etc/controller.xml</CLICON_CONFIGFILE>
  <CLICON_FEATURE>ietf-netconf:startup</CLICON_FEATURE>
  <CLICON_FEATURE>clixon-restconf:allow-auth-none</CLICON_FEATURE>
  <CLICON_CONFIG_EXTEND>clixon-controller-config</CLICON_CONFIG_EXTEND>
  <CONTROLLER_ACTION_COMMAND xmlns="http://clicon.org/controller-config">
        /usr/local/bin/clixon_server.py -F -f /usr/local/share/clixon/modules
  </CONTROLLER_ACTION_COMMAND>
  <CLICON_BACKEND_USER>clicon</CLICON_BACKEND_USER>
  <CLICON_SOCK_GROUP>clicon</CLICON_SOCK_GROUP>


CLI
===
This section desribes the CLI commands of the Clixon controller. A simple example is used to illustrate concepts.

Modes
-----
The CLI has two modes: operational and configure. The top-levels are as follows::
   
  > clixon_cli
  cli> ?
    configure             Change to configure mode
    connection            Change connection state of one or several devices
    debug                 Debugging parts of the system
    exit                  Quit
    processes             Process maintenance 
    pull                  Pull config from one or multiple devices
    push                  Push config to one or multiple devices
    quit                  Quit
    save                  Save running configuration to XML file
    services              Services operation
    shell                 System command
    show                  Show a particular state of the system   

  cli> configure 
  cli[/]# set ?
    devices               Device configuration
    processes             Processes configuration
    services              Placeholder for services                                                       
  cli[/]#


Devices
-------
Device configuration is separated into two domains:

1) Local information about how to access the device (meta-data)
2) Remote device configuration pulled from the device. 

The user must be aware of this distinction when performing `commit` operations.

Local device configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^
The local device configuration contains information about how to access the device::

   device clixon-example1 {
      description "Clixon example container";
      enabled true;
      conn-type NETCONF_SSH;
      user admin;
      addr 172.17.0.3;
      yang-config VALIDATE;
   }

A user makes a local commit and thereafter explicitly connects to a locally configured device::

  # commit local
  # exit
  > connection open

Remote device configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^
The remote device configuration is present under the `config` mount-point::

   device clixon-example1 {
      ... 
      config {
         interfaces {
            interface eth0 {
               mtu 1500;
            }
         }
      }
   }

The remote device configuration is bound to device-specific YANG models downloaded
from the device at connection time. 
   
Device naming
^^^^^^^^^^^^^
The local device name is used for local selection::

   device example1

Wild-cards (globbing) can be used to select multiple devices::

   device example*

Further, device-groups can be configured and accessed as a single entity::
  
   device-group all-examples

.. note::
          Device groups can be statically configured but not used in most operations
   
In the forthcoming sections, selecting `<devices>` means any of the methods described here.

Device state
^^^^^^^^^^^^
Examine device connection state using the show command::

   cli> show devices
   Name                    State      Time                   Logmsg                        
   =======================================================================================
   example1                OPEN       2023-04-14T07:02:07    
   example2                CLOSED     2023-04-14T07:08:06    Remote socket endpoint closed

There is also a detailed variant of the command with more information in XML::

   olof@zoomie> show devices detail 
   <devices xmlns="http://clicon.org/controller">
     <device>
       <name>example1</name>
       <description>Example container</description>
       <enabled>true</enabled>
       ...
  
(Re)connecting
^^^^^^^^^^^^^^
When adding and enabling one a new device (or several), the user needs to explicitly connect::

   cli> connection <devices> connect
   
The "connection" command can also be used to close, open or reconnect devices::

   cli> connection <devices> reconnect


Syncing from devices
--------------------
pull
^^^^
Pull fetches the configuration from remote devices and replaces any existing device config::

   cli> pull <devices>

The synced configuration is saved in the controller and can be used for diffs etc.


pull merge
^^^^^^^^^^
::
   
   cli> pull <devices> merge
   
This command fetches the remote device configuration and merges with the
local device configuration. use this command with care.

Services
--------
Network services are used to generate device configs.

Service process 
^^^^^^^^^^^^^^^^
To run services, the PyAPI service process must be enabled::

  cli# set services enabled true
  cli# commit local

To view or change the status of the service daemon::

  cli> service process ?
    restart
    start
    status
    stop
  
Example
^^^^^^^
An example service could be::

  cli> set service test 1 e* 1400

which adds MTU `1400` to all interfaces in the device config::

  interfaces {
    interface eth0{
      mtu 1400;
    }
    interface enp0s3{
      mtu 1400;
    }
  }

Service scripts are written in Python using the PyAPI, and are triggered by commit commands.

You can also trigger service scripts as follows::

  cli# services reapply

Editing
-------
Editing can be made by modifying services::

    cli# set services test 2 eth* 1500

Editing changes the controller candidate, changes can be viewed with::

   cli# show compare 
        services {
   +       test 2 {
   +          name eth*;
   +          mtu 1500;
   +       }
        }

Editing devices
^^^^^^^^^^^^^^^
Device configurations can also be directly edited::  

   cli# set devices device example1 config interfaces interface eth0 mtu 1500
       
Show and editinf commands can be made on multiple devices at once using "glob" patterns::

   cli> show config xml devices device example* config interfaces interface eth0
   example1:
   <interface>
      <name>eth0</name>
      <mtu>1500</mtu>
   </interface>
   example2:
   <interface>
      <name>eth0</name>
      <mtu>1500</mtu>
   </interface>

Modifications using set, merge and delete can also be applied on multiple devices::

   cli# set devices device example* config interfaces interface eth0 mtu 9600
   cli#

Commits
-------
This section describes `remote` commit, i.e., commit operations that have to do with modifying remote device configuration. See Section `devices`_ for how to make local commits for setting up device connections.

commit diff
^^^^^^^^^^^
Assuming a service has changed as shown in the previous secion, the
`commit diff` command shows the result of running the service
scripts modifying the device configs, but with no commits actually done::

   cli# commit diff
        services {
   +       test 2 {
   +          name eth*;
   +          add 1500;
   +       }
        }
        devices {
           device example1 {
              config {
                 interfaces {
                    interface eth0 {
   -                   mtu 1400;
   +                   mtu 1500;
                    }
                 }
              }
           }
           device example33 {
              config {
                 interfaces {
                    interface eth3 {
   -                   mtu 1400;
   +                   mtu 1500;
                    }
                 }
              }
           }
        }

Commit push
^^^^^^^^^^^
The changes can now be pushed and committed to the devices::

   cli# commit push  

If there are no services, changes will be pushed and committed without invoking any service handlers.

If the commit fails for any reason, the error is printed and the changes remain as prior to the commit call::
   
   cli# commit push
   Failed: device example1 validation failed
   Failed: device example2 out-of-sync

A non-recoverable error that requires manual intervention is shown as::

   cli# commit push
   Non-recoverable error: device example2: remote peer disconnected
   
To validate the configuration on the remote devices, use the following command::

   cli# validate push

If you want to rollback the current edits, use discard::

   cli# discard

One can also choose to not push the changes to the remote devices::

   cli# commit local

This is useful for setting up device connections. If a local commit is performed for remote device config, you need to make an explicit `push` as described in Section `Explicit push`_.

Limitations
^^^^^^^^^^^
The following combinations result in an error when making a remote commit:

1) No devices are present. However, it is allowed if no remote validate/commit is made. You may want to dryrun service python code for example even if no devices are present.
2) Local device fields are changed. These may potentially effect the device connection and should be made using regular netconf local commit followed by rpc connection-change, as described in Section `devices`_.
3) One of the devices is not in an OPEN state. Also in this case is it allowed if no remote valicate/commit is made, which means you can do local operations (like `commit diff`) even when devices are down.

Further, avoid doing BOTH local and remote edits simultaneously. The system detects local edits (according to (2) above) but if one instead  uses local commit, the remote edits need to be explicitly pushed

Compare and check
-----------------
The "show compare" command shows the difference between candidate and running, ie not committed changes.
A variant is the following that compares with the actual remote config::

   cli> show devices <devices> diff

This is acheived by making a "transient" pull that does not replace the local device config.

Further, the following command checks whether devices are is out-of-sync::

   cli> show devices <devices> check
   Failed: device example2 is out-of-sync

Out-of-sync means that a change in the remote device config has been made, such as a manual edit, since the last "pull".
You can resolve an out-of-sync state with the "pull" command.

Explicit push
-------------
There are also explicit sync commands that are implicitly made in
`commit push`. Explicit pushes may be necessary if local commits are
made (eg `commit local`) which needs an explicit push. Or if a new device has been off-line::

     cli> push <devices>

Push the configuration to the devices, validate it and then revert::

     cli> push <devices> validate 

            
Transactions
============
A basis of controller operation is the use of transactions. Clixon itself has underlying candidate/running datastore transactions. The controller expands the transaction concept to span multiple devices.
There are two such types of composite transactions:

1. `Device connect`: where devices are connected via NETCONF over ssh, key exchange, YANG retrieval and config pull
2. `Config push`: where a service is (optionally) edited, changed device config is pushed to remote devices via NETCONF.

.. image:: transaction.jpg
   :width: 100%
           
A `device connect` transaction starts in state `CLOSED` and if succesful stops in `OPEN`. there are multiple intermediate steps as follows (for each device):

1. An SSH session is created to the IP address of the device
2. An SSH login is made which requires:

   a) The device to have enabled a NETCONF ssh sub-system
   b) The public key of the controller to be installed on the device
   c) The public key of the device to be in the `known_hosts` file of the controller
3. A mutual NETCONF `<hello>` exchange
4. Get all YANG schema identifiers from the device using the ietf-netconf-monitoring schema.
5. For each YANG schema identifier, make a `<get-schema>` RPC call (unless already retrieved).
6. Get the full configuration of the device.

While a `device connect` operates on individual devices, the `config push` transaction operates on all devices. It starts in `OPEN` for all devices and ends in `OPEN` for all devices involved in the transaction:

1. The user edits a service definition and commits
2. The commit triggers PyAPI services code, which rewrites the device config
3. Alternatively, the user edits the device configuration manually
4. The updated device config is validated by the controller
5. The remote device is checked for updates, if it is out of sync, the transaction is aborted
6. The new config is pushed to the remote devices
7. The new config is validated on the remote devices
8. If validation succeeds on all remote devices, the new config is committed to all devices
9. The new config is retreived from the device and is installed on the controller
10. If validation is not successful, or only a `push validate` was requested, the config is reverted on all remote devices.

Use the show transaction command to get details about transactions::

   cli> show transaction
     <transaction>
        <tid>2</tid>
        <state>DONE</state>
        <result>FAILED</result>
        <description>pull</description>
        <origin>example1</origin>
        <reason>validation failed</reason>
        <timestamp>2023-03-27T18:41:59.031690Z</timestamp>
     </transaction>


YANG
====
The clixon-controller YANG has the following structure::

   module: clixon-controller
     +--rw processes
     |   +--rw services
     |     +--rw enabled              boolean
     +--rw services
     |   +--rw properties
     +--rw devices
     |   +--rw device-timeout         uint32
     |   +--rw device-group* [name]
     |   | +--rw name                 string
     |   +--rw device* [name]
     |     +--rw name                 string
     |     +--rw description?         string
     |     +--rw enabled?             boolean
     |     +--rw conn-type            connection-type
     |     +--rw user?                string
     |     +--rw addr?                string
     |     +--rw yang-config?         yang-config
     |     +--rw capabilities
     |     | +--rw capability*        string
     |     +--ro conn-state-timestamp yang:date-and-time
     |     +--ro sync-timestamp       yang:date-and-time
     |     +--ro logmsg               string
     |     +--rw config
     +--ro transactions
         +--ro transaction* [tid]
           +--ro tid                  uint64
     notifications:
       +---n services-commit
       +---n controller-transaction
     rpcs:
         +--config-pull
         +--controller-commit
         +--connection-change
         +--get-device-config
         +--transaction-error
         +--transaction-actions-done
         +--datastore-diff
  
The services section contains user-defined services not provided by
the controller.  A user adds services definitions using YANG `augment`. For example::

    import clixon-controller { prefix ctrl; }
    augment "/ctrl:services" {
        list myservice {
            ...

Actions API
===========
The controller provides an `actions API` which is a YANG-defined protocol for external action handlers, including the `PyAPI`.

The backend implements a tagging mechanism to keep track of what parts
of the configuration tree were created by which services.  In this
way, reference counts are maintained so that objects can be removed in
a correct way if multiple services create the same object.

There are some restrictions on the current actions API:

1. Only a single action handler is supported, which means that a single action handler handles all services.
2. The algorithm is not hierarchical, that is, if there is a tag on a device object, tags on children are not considered
3. No persistence: if the backend is restarted, tags are lost.


Service instance
----------------
A service extends the controller yang as described in the `YANG`_ section. For example, a service `ssh-users` may augment the original as follows::

   augment "/ctrl:services" {
      list ssh-users {   // YANG list
         key group;      // Single key
         leaf group {
            type string;
	 }
         list username {
            key name;
            leaf name{
               type string;
            }
            leaf ssh-key {
               type string;
            }
         }
      }
   }

The service must be on the following form:

1. The top-level is a YANG list (eg `ssh-users` above)
2. The list has a single key (eg `group` above)

The rest of the augmented service can have any form (eg `list username` above).
   
.. note::
        An augmented service must start with a YANG list with a single key

An example service XML for `ssh-users` is::

   <services xmlns="http://clicon.org/controller">
     <ssh-users xmlns="urn:example:test">
        <group>ops</group>
        <username>
           <name>eric</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
        <username>
           <name>alice</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
     </ssh-users>
     <ssh-users xmlns="urn:example:test">
        <group>devs</group>
        <username>
           <name>kim</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
        <username>
           <name>alice</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
     </ssh-users>
   </services>

The actions protocol defines a service instances as::

  <list>  |  <list>[<key>='<value>']

From the example YANG above, examples of service instances of `ssh-users` are::

  ssh-users
  ssh-users[group='ops']
  ssh-users[group='devs']

where the first identifies all `ssh-users` instances and the other two
identifies the specific instances given above

Device config
-------------
The service definition is input to changing the device config, where the actual change is made by
Python code in the PyAPI.

A device configuration could be as follows (inspired by openconfig)::

  container users {
     description "Enclosing container list of local users";
     list user {
        key "username";
        description "List of local users on the system";
        leaf username {
            type string;
            description "Assigned username for this user";
        }
        leaf ssh-key {
            type string;
            description "SSH public key for the user (RSA or DSA)";
        }
     }
  }

Tags
----
An action handler tags device configuration objects it creates with the name of the service instances
using the `cl:creator` YANG extension.  This is used to track which instance created
an object and acts as a reference count when removing objects.  An object may have several tags if it is created by more than one service instance.

In the following example, three device objects are tagged with service instances in one device, as follows:

.. table:: `Device A with service-instance tags`
   :widths: auto
   :align: left

   =============  =======================
   Device object  Service-instance
   =============  =======================
   eric           ssh-users[group='ops']
   alice          ssh-users[group='devs']
   kim            ssh-users[group='ops'],
                  ssh-users[group='devs']
   =============  =======================

where device objects `eric` and `alice` are created by service instance `ops` (more precisely `ssh-users[group='ops']`) and `devs` respectively, and `kim` is created by both.

Suppose that service instance `ops` is deleted, then all device objects tagged with `ops` are deleted:

.. table:: `Device A after removal of ops`
   :widths: auto
   :align: left
            
   =============  =======================
   Device object  Service-instance
   =============  =======================
   alice          ssh-users[group='devs']
   kim            ssh-users[group='devs']
   =============  =======================

Note that `kim` still remains since it was created by both ops and devs.

Note also that this example only considers a single device `A`. In reality there are many more devices.

Example python
--------------
An example PyAPI script takes the service ssh-users definition and creates users on the actual devices, for example::

    for instance in root.services.users:
        for user in instance.username:
            username = ssh-users.name.cdata
            ssh_key = ssh-users.ssh_key.cdata
            for device in root.devices.device:
                new_user = Element("user",
                                   attributes={
                                       "cl:creator": "users[group='ops']",
                                       "nc:operation": "merge",
                                       "xmlns:cl": "http://clicon.org/lib"})
                new_user.create("name", cdata=username)
                new_user.create("authentication")
                new_user.authentication.create("ssh-rsa")
                new_user.authentication.ssh_rsa.create("name", cdata=ssh_key)
                device.config.configuration.system.login.add(new_user)


Algorithm
---------
The algorithm for managing device objects using tags is as follows. Consider a commit operation where some services have changed by adding, deleting or modifying service -instances:

  1. The controller makes a diff of the candidate and running datastore and identifies all changed services-instances
  2. For all changed service-instances S:
    
    - For all device nodes D tagged with that service-instance tag:

      - If S is the only tag, delete D
      - Otherwise, delete the tag, but keep D

  3. The controller sends a notification to the PYAPI including a list of modified service-instances S
  4. The PyAPI creates device objects based on the service instances S, merges with the datastore and commits
  5. The controller makes a diff between the modified datastore and running and pushes to the devices

The algorithm is stateless in the sense that the PyAPI recreates all
objects of the modified service-instances. If a device object is not
created, it is considered as deleted by the controller. Keeping track
of deleted or changed service-instances is done only by the
controller.
     
Protocol
--------
The following diagram shows an overview of the action protocol::

     Backend                           Action handler
        |                                  |
        + <--- <create-subscription> ---   +
        |                                  |
        +  --- <services-commit> --->      +
        |                                  |
        + <---   <edit-config>   ---       +
        |            ...                   |
        + <---   <edit-config>   ---       +
        |                                  |
        + <---  <trans-actions-done> ---   +
        |                                  |
        |          (wait)                  |
        +  --- <services-commit> --->      +
        |            ...                   |           
           
where each message will be described in the following text.
        
Registration
^^^^^^^^^^^^
An action handler registers subscriptions of service commits by using RFC 5277
notification streams::

    <create-subscription>
       <stream>service-commit</stream>
    </create-subscription>

Notification
^^^^^^^^^^^^
Thereafter, controller notifications of type `service-commit` are sent
from the backend to the action handler every time a
`controller-commit` RPC is initiated with an `action` component. This
is typically done when CLI commands `commit push`, `commit diff` and
others are made.

An example of a `service-commit` notification is the following::

    <services-commit>
       <tid>42</tid>
       <source>candidate</source>
       <target>actions</target>
       <service>ssh-users[group='ops']</service>
       <service>ssh-users[group='devs']</service>
    </services-commit>

In the example above, the transaction-id is `42` and the services definitions are read from
the `candidate` datastore. Updated device edits are written to the `actions` datastore.

The notification also informs the action server that two service instances have changed.

A special case is if `no` service-instance entries are present. If so, it means
`all` services in the configuration should be re-applied.


Editing
^^^^^^^
In the following example, the PyAPI adds an object in the device configuration tagged with the service instance `ssh-users[group='ops']`::

  <edit-config>
    <target><actions xmlns="http://clicon.org/controller"/></target>
    <config>
      <devices xmlns="http://clicon.org/controller">
        <device>
          <name>A</name>
          <config>
            <users xmlns="urn:example:users" xmlns:cl="http://clicon.org/lib" nc:operation="merge">
              <user cl:creator="ssh-users[group='ops']">
                <username>alice</username>>
                <ssh-key>ssh-rsa AAA...</ssh-key>
              </user>
          </users>
          </config>
        </device>
      </devices>
    </config>
  </edit-config>

Note that the action handler needs to make a `get-config` to read the
service definition.  Further, there is no information about what
changes to the services have been made. The idea is that the action
handler reapplies a changed service and the backend sorts out any
deletions using the tagging mechanism.

Finishing
^^^^^^^^^
When all modifications are done, the action handler issues a `transaction-actions-done` message to the backend::

    <transaction-actions-done xmlns="http://clicon.org/controller">
      <tid>42</tid>
    </transaction-actions-done>

After the `done` message has been sent, no further edits are made by
the action handler, it waits for the next notification.

The backend, in turn, pushes the edits to the devices, or just shows
the diff, or validates, depending on the original request parameters.

Error
^^^^^
The action handler can also issue an error to abort the transaction. For example::
  
    <transaction-error>
      <tid>42</tid>
      <origin>pyapi</origin>
      <reason>No connection to external server</reason>
    </transaction-error>

In this case, the backend terminates the transaction and signals an error to the originator, such as a CLI user.
    
Another source of error is if the backend does not receive a `done`
message. In this case it will eventually timeout and also signal an error.
