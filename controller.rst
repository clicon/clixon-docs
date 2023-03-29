.. _clixon_controller:
.. sectnum::
   :start: 15
   :depth: 3

**********
Controller
**********

This section describes the clixon network controller, an extension of
Clixon to manage devices using NETCONF.

The aim of the clixon network controller is to provide a simple
network controller for NETCONF devices (not only Clixon).

Further, the controller has the following features:

- Network services, with a Python API
- Multiple devices, with different YANG schemas
- Transactions with validate/commit/revert across groups of devices
- RFC 8528 yang schema mount allows multiple yang schemas (per device)
- Scaling up to 100 devices.


Architecture
============

.. image:: controller.jpg
   :width: 100%

The controller is built using the regular CLIgen/Clixon system, where
the controller semantics is implemented using plugins. The `backend`
is the core of the system controlling the datastores and accessing the
YANG models.

The southbound API is NETCONF over SSH to network devices.

The northbound APIs are YANG-derived Restconf, Autocli, Netconf, and
Snmp.  The controller CLI has two modes: operation and configure, with
an autocli configure mode derived from YANG.

A PyAPI module is used to implement user-specific network services
which interpret services configuration into device configuration,
which are pushed to the actual devices using a transaction mechanism.

Transactions
------------
.. image:: transaction.jpg
   :width: 100%

A typical controller session works as follows:

1. A `sync pull` retrieves the configurations from the devices into the devices section of the controller config.
2. The user edits a service defintion and commits
3. The commit triggers the PyAPI services code, which rewrites the device config
4. The updated device config is validated by the controller
5. The updated device config is pushed to the devices, and if successful committed by the controller

In the clixon controller the `sync push` of the device config works by a transaction mechanism involving the controller and a set of remote devices:

1. Check: The remote device is checked for updates, if it is out of sync, the transaction is aborted
2. Edit: The new config is pushed to the remote devices
3. Validate: The new config is validated on the remote devices
4. Wait: wait for all validations to succeed
5. Commit: If validation succeeds, the new config is committed on all devices
6. Discard: If validation is not successful, or only a `sync push validate` was requested, the config is reverted on all remote devices.
        
Use the show transaction command to get more info why a transaction failed::

   cli> show transaction
     <transaction>
        <tid>2</tid>
        <state>DONE</state>
        <result>FAILED</result>
        <description>sync pull</description>
        <origin>example1</origin>
        <reason>validation failed</reason>
        <timestamp>2023-03-27T18:41:59.031690Z</timestamp>
     </transaction>

There are two error levels:

- FAILED: validation error or other recoverable error
- ERROR: uneciverable error where manual resolving is necessary
  
CLI
===
This section desribes the CLI commands of the Clixon controller. A simple example is used to illustrate concepts.

Modes
-----
The CLI has two modes: operational and configure. The top-levels are as follows::
   
  > clixon_cli
  cli> ?
    configure             Change to configure mode
    connection            Reconnect one or several devices in closed state
    debug                 Debugging parts of the system
    exit                  Quit  
    quit                  Quit
    save                  Save running configuration to XML file
    services              Services operation
    shell                 System command
    show                  Show a particular state of the system
    sync                  Read the config of one or several devices.
  cli> configure 
  cli[/]# set ?
    devices               Device configurations
    generic               Generic controller settings
    services              Placeholder for services                                                       
  cli[/]#

Devices
-------

Device naming
^^^^^^^^^^^^^
A device has a name which can be used to select it::

  device example1

Wild-cards (globbing) can be used to select multiple devices::

  device example*

Further, device-groups can be configured and accessed as a single entity::
  
  device-group all-examples
  
In the forthcoming sections, selecting `<devices>` means any of the methods described here.

Device configuration
^^^^^^^^^^^^^^^^^^^^
The controller manipulates device configuration, according to YANG models downloaded from the device at start time. A very simple device configuration data example is::

  interfaces {
    interface eth0;
    interface enp0s3;
  }

Syncing from devices
--------------------
sync pull
^^^^^^^^^
Sync pull fetches the configuration from remote devices and replaces any existing device config::

   cli> sync pull <devices>

The synced configuration is saved in the controller and can be used for diffs etc.


sync pull merge
^^^^^^^^^^^^^^^
::
   
   cli> sync pull <devices> merge
   
This command fetches the remote device configuration and merges with the
local device configuration. use this command with care.

Services
--------
Network services are used to generate device configs.

An example service could be::

  cli> set service test 1 e* 1400;

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

Editing
-------
Editing can be made by modifying services::

    cli# set services test 2 eth* 1500

Editing changes the controller candidate, changes can be viewed::

   cli# show compare 
        services {
   +       test 2 {
   +          name eth*;
   +          mtu 1500;
   +       }
        }

Device configurations can also be directly edited::  

   cli# set devices device example1 config interfaces interface eth0 mtu 1500
       
Validates and commits
---------------------

commit dry-run
^^^^^^^^^^
Assuming a service has changed as shown in the previous secion, the
`commit dry-run` command shows the result of running the service
scripts modifying the device configs, but with no commits actually done::

   cli# commit dry-run
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

Commit
^^^^^^
The changes can now be pushed and committed to the devices::

   cli# commit push  

If the commit fails for any reason, the error is printed and the changes are discarded::
   
   cli# commit push
   Failed: device example1 validation failed
   Failed: device example2 out-of-sync

A non-recoverable error that requires manual intervention is shown as::

   cli# commit push
   Non-recoverable error: device example2: remote peer disconnected
   
One can also choose to not push the changes to the remote devices::

   cli# commit local

One can also chose to validate the configuration on the remote devices::

   cli# validate push

Compare
-------
There are several compare commands related to device configuration to
check differences in device config state between the remote (actual)
device, and the controller's device config. This can be used to
determine if the device configs are out-of-sync.

There are three basic device configs used in these comparisons:

- `Running`: The local controller device config. This is updated by a `sync pull`, commit, or rollback.
- `Synced`: The cached device config retrieved in the most recent `sync pull` command. A successful `sync push` also updates this config.
- `Remote`: The actual configuration of the remote device. If this is different from `Synced`, the device is `out-of-sync` and is always checked when doing `sync push`.

Operations that affect the device configs::

  Remote      -- sync pull -->       Running/Synced
  Running     -- sync push -->       Synced/Remote
  Running     -- commit/rollback --> Running

Note that there is also local `show compare` command in
configure mode showing the differences between the local controller
candidate and local running which is only useful in editing (eg services).

show compare <devices> running synced
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Show comparison between `Running` and `Synced`. It shows any differences made to the device config by the controller, either due to manual edits, or by triggering services scripts.

show compare <devices> synced remote
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Show comparison between the most recent sync of a device with the
actual remote config. If there is a diff, it has been made by a third
part, and the device is `out-of-sync`.

You can also use: `check <devices> synced remote` to get a yes/no answer on whether the device is out of sync.


show compare <devices> running remote
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Show comparison between running device config on the controller and remote
device configuration. This is only different from running-synced of a remote party has changed the config on the remote device.

Sync push
---------
There are also explicit sync commands that are implictly made in
`commit push`. Explicit pushes may be necessary if local commits are
made (eg `commit local`) which needs an explicit push. Or if a new device has been off-line::

     cli> sync push <devices>

* Validate. Push the configuration to the devices and validate it::

     cli> sync push <devices> validate 

YANG
====

The clixon-controller YANG has the following structure::

   module: clixon-controller
     +--rw services
     |   +--rw service* [name]
     +--rw generic
     |   +--rw device-timeout         uint32
     +--rw devices
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
     |     +--rw root
     +--ro transactions
         +--ro transaction* [tid]
           +--ro tid                  uint64
     notifications:
       +---n services-commit
       +---n controller-transaction

     rpcs:
         +--sync-pull
         +--sync-push
         +--services-apply
         +--connection-change
         +--get-device-config
         +--transaction-error
  
In short, the configuration part of the YANG is separated into
`services` and `devices`.

The services section contains user-defined services not provided by
the controller.  A user adds services definitions using YANG `augment`. For example::

    import clixon-controller { prefix ctrl; }
    augment "/ctrl:services" {
        list myservice {
            ...
            
