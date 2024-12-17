.. _clixon_events:
.. sectnum::
   :start: 14
   :depth: 3

*******************
Event notifications
*******************

Overview
========

Clixon implements RFC 5277 NETCONF Event Notifications.

There are no pre-existing notification streams in Clixon, an application needs to set them up.

The main example illustrates an EXAMPLE stream notification that triggers every 5s.

Configure options
-----------------
CLICON_STREAM_DISCOVERY_RFC5277
   Enable event stream discovery as described in RFC 5277. This is the main option and should be ``true``.
CLICON_STREAM_DISCOVERY_RFC8040
   Enable monitoring information, including streams, for RESTCONF
CLICON_STREAM_URL
   URL for locating event stream
CLICON_STREAM_RETENTION
   Retention for stream replay buffers in seconds
CLICON_STREAM_PUB
   NCHAN event publication. Obsolete

Restrictions
------------
Limited replay support

Components
==========

This section describes which components are involved to add a new event notification stream in Clixon.

To create a new event notification, you need to define the following:

1. `YANG`: Define a schema event spec
2. `Backend`: and send server-side event messages
3. `CLI`: Subscribe to the event-stream, read and display events

YANG
----
The first step is to add an event notification in YANG. The notification statement is a part of the `YANG specification <https://www.rfc-editor.org/rfc/rfc7950.html#section-7.16>`_

The main example defines an event notification following event which is also used in RFC8040 and RFC5277::

   notification event {
      description "Example notification event.";
      leaf event-class {
         type string;
         description "Event class identifier.";
      }
      container reportingEntity {
         description "Event specific information.";
         leaf card {
            type string;
         }
      }
      leaf severity {
         type string;
      }
   }

You can add any info to an event notifcation, the fields above are just examples.

Backend
-------
The server-side first declares an event-stream, for example event ``my-event``::

   stream_add(h, "my-event", "Example notification event, 0, NULL)

Then, code needs to be added to generate event. This is highly application dependent.

Typically, events are either timer-based or event-based.

The main example uses a timer-based event, that is, a periodic timer is set that generates a periodic event.

One can also create an event-based event that triggers on things like commits or logs. Typically this code is added in applicatin user code.

You create a NETCONF notification event in XML, for example::

  <my-event xmlns="urn:myuri...">
    <data-for-this-event>....
  </my-event>

And then, assuming ``msg`` is a string variable with the notification, you send it with::

   stream_notify(h, "my-event", msg)

CLI
---
You need to register a new socket "s"::

   int s;
   clicon_rpc_create_subscription(h, "my-event", NULL, &s)


Save the asynchronous socket "s" and read from it separately.

There are some examples on how to do this in the clixon main example and the controller, eg here: https://github.com/clicon/clixon-controller/blob/9a3ef161106207f6a49f3fbc14e50841f20f50e3/src/controller_cli_callbacks.c#L431
But it depends on how you want the CLI to behave.

Netconf
-------
To start a notification stream via netconf::

   <rpc><create-subscription xmlns="urn:ietf:params:xml:ns:netmod:notification"><stream>EXAMPLE</stream></create-subscription></rpc>]]>]]>
   <rpc-reply><ok/></rpc-reply>]]>]]>
   <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0"><eventTime>2019-01-02T10:20:05.929272</eventTime><event><event-class>fault</event-class><reportingEntity><card>Ethernet0</card></reportingEntity><severity>major</severity></event></notification>]]>]]>


Restconf
--------
An example using curl::

  curl  -X GET -H "Accept: text/event-stream" -H "Cache-Control: no-cache" -H "Connection: keep-alive" https://thehost/streams/


Main example
============

The main example has an integrated CLI event notification. To try out::

  clixon_cli -f /usr/local/etc/clixon/example.xml
  cli> notify
  cli> event-class fault;
  reportingEntity {
    card Ethernet0;
  }
  severity major;

  cli> no notify
  cli>
