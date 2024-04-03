.. _clixon_restconf:
.. sectnum::
   :start: 11
   :depth: 3

********
RESTCONF
********

.. This is a comment
   
Clixon supports two RESTCONF compile-time variants: *FCGI* and *Native*. 
   
Architecture
============

 .. image:: restconf1.jpg
   :width: 100%

The restconf deamon provides a http/https RESTCONF interface to the
Clixon backend.  It comes in two variants, as shown in the figure above:

  1. Native http, which combines a HTTP and Restconf server. Further, HTTP configuration is made using Clixon.
  2. A reverse proxy (such as NGINX) and FastCGI where web and restconf function is separated. NGINX is used to make all HTTP configuration.

The restconf daemon communicates with the backend using
internal netconf over the ``CLIXON_SOCK``. If FCGI is used, there is also a FCGI socket specified by ``fcgi-socket`` in ``clixon-config/restconf``.

The restconf daemon reads its initial config options from the configuration file on startup. The native http variant can read config options from the backend as an alternative to reading everything from clixon options.

You can add plugins to the restconf daemon, where the primary usecase is authentication, using the ``ca_auth`` callback.

Note that there is some complexity in the configuration of the
different variants of native Clixon restconf involving HTTP/1 vs
HTTP/2, TLS vs plain HTTP, client cert vs basic authentication and
external vs internal daemon start.

Further, ALPN is used to select http/1 or http/2 in HTTPS.

Installation
============
The RESTCONF daemon can be configured for compile-time (by autotools) as follows:

  --disable-http1         Disable native http/1.1 (ie http/2 only)
  --disable-nghttp2       Disable native http/2 using libnghttp2 (ie http/1 only)
  --with-restconf=native  RESTCONF using native http. (DEFAULT)
  --with-restconf=fcgi    RESTCONF using fcgi/ reverse proxy.
  --without-restconf      No RESTCONF

After that perform system-wide compilation::

    make && sudo make install
  
Command-line options
====================
The restconf daemon have the following command-line options:
  -h              Help
  -V              Show version and exit
  -D <level>      Debug level
  -f <file>       Clixon config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut, (n)one or (f)ile. Syslog is default. If foreground, then syslog and stderr is default.
  -C <format>     Dump configuration options on stdout after loading and quit. Format is one of xml|json|text
  -p <dir>        Add Yang directory path (see CLICON_YANG_DIR)
  -y <file>       Load yang spec file (override yang main module)
  -a <family>     Internal backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal socket domain path or IP addr (see -a)
  -r              Do not drop privileges if run as root
  -W <user>       Run restconf daemon as this user, drop according to ``CLICON_RESTCONF_PRIVILEGES``
  -R <xml>        Restconf configuration in-line overriding config file
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

Note that the restconf daemon started as root, drops privileges to `wwwuser`, unless the ``-r`` command-line option is used, or ``CLICON_RESTCONF_PRIVILEGES`` is defined.


Configuration options
=====================
The following RESTCONF configuration options can be defined in the clixon configuration file:

CLICON_RESTCONF_DIR
   Location of restconf .so plugins. Load all .so plugins in this dir as restconf code plugins.
CLICON_RESTCONF_INSTALLDIR
   Path to dir of clixon-restconf daemon binary as used by backend if started internally
CLICON_RESTCONF_STARTUP_DONTUPDATE
   Disable automatic update of startup on restconf edit operations
   This is not according to standard RFC 8040 Sec 1.4.
CLICON_RESTCONF_USER
   Run clixon_restconf daemon as this user, default is `www-data`.
CLICON_RESTCONF_PRIVILEGES
   Restconf daemon drop privileges mode, one of: none, drop_perm, drop_temp
CLICON_RESTCONF_HTTP2_PLAIN
   Enable plain (non-tls) HTTP/2.
CLICON_BACKEND_RESTCONF_PROCESS
   Start restconf daemon internally from backend daemon. The restconf daemon reads its config from the backend running datastore. 
CLICON_ANONYMOUS_USER
   If RESTCONF authentication auth-type=none then use this user
CLICON_RESTCONF_API_ROOT
   RESTCONF API root path as defined in RFC 8040, default is `/restconf`
CLICON_NOALPN_DEFAULT
   Fallback if no ALPN for https. valid values are "http/1.1" and "http/2"

More more documentation of the options, see the source YANG file.
   
Advanced config
===============
Apart from options, there is also structured restconf data primarily for native mode encapsulated with ``<restconf>...</restconf>`` as defined in ``clixon-restconf.yang``. 

The first-level fields of the advanced restconf structure are the following:

enable
   Enable the RESTCONF daemon. If disabled, the restconf daemon will not start
auth-type
   Authentication method (see `auth types`_)
debug
   Enable debug
log-destination
   Either syslog or file (/var/log/clixon_restconf.log)
pretty
   Restconf vallues are pretty printed by default. Disable to turn this off

The advanced config can be given using three different methods

  1. inline - as command-line option using ``-R``
  2. config-file - as part of the regular config file
  3. datastore - committed in the regular running datastore

Inline
------
When starting the restconf daemon, structured data can be directly given as a command-line option::

  -R <restconf xmlns="http://clicon.org/restconf"><enable>true</enable></restconf>

Config file
-----------
The restconf config can also be defined locally within the clixon config file, such as::

  <CLICON_FEATURE>clixon-restconf:fcgi</CLICON_FEATURE>
  <CLICON_BACKEND_RESTCONF_PROCESS>false</CLICON_BACKEND_RESTCONF_PROCESS>
  <restconf>
      <enable>true</enable>
      <fcgi-socket>/wwwdata/restconf.sock</fcgi-socket>
   </restconf>

Datastore
---------
Alternatively if ``CLICON_BACKEND_RESTCONF_PROCESS`` is set, the restconf configuration is::

  <CLICON_FEATURE>clixon-restconf:fcgi</CLICON_FEATURE>
  <CLICON_BACKEND_RESTCONF_PROCESS>false</CLICON_BACKEND_RESTCONF_PROCESS>

And the detailed restconf is defined in the regular running datastore by adding something like::

   <restconf xmlns="http://clicon.org/restconf">
      <enable>true</enable>
      <fcgi-socket>/wwwdata/restconf.sock</fcgi-socket>
   </restconf>
   
In the latter case, the restconf daemon reads its config from the running datastore on startup. 

.. note::
      If ``CLICON_BACKEND_RESTCONF_PROCESS`` is enabled, the restconf config must be in the regular datastore.

Features
--------
The Restconf config has two features:

fcgi
   The restconf server supports the fast-cgi reverse proxy mode. Set this if fcgi/nginx is used.
allow-auth-none
   Authentication supports a `none` mode.

Example, add this in the config file to enable fcgi::

   <clixon-config xmlns="http://clicon.org/config">
      ...
      <CLICON_FEATURE>clixon-restconf:fcgi</CLICON_FEATURE>
   
Auth types
----------
The RESTCONF daemon uses the following authentication types:

none
   Messages are not authenticated and set to the value of ``CLICON_ANONYMOUS_USER``. A callback can revise this behavior. Note, must set `allow-auth-none` feature.
client-cert
   Set to authenticated and extract the username from the SSL_CN parameter. A callback can revise this behavior.
user
   User-defined behaviour as implemented by the `auth callback`_. Typically done by basic auth, eg HTTP_AUTHORIZATION header, and verify password

FCGI mode
---------
Applies if clixon is configured with ``--with-restconf=fcgi``. Fcgi-specific config options are:

fcgi-socket
   Path to FCGI unix socket. This path should be the same as specific in fcgi reverse proxy

Need also fcgi feature enabled: `features`_
   
Native mode
-----------
Applies if clixon is configured with ``--with-restconf=native``. 
Native specific config options are:

server-cert-path
   Path to server certificate file
server-key-path
   Path to server key file
server-ca-cert-path
   Path to server CA cert file
socket
   List of server sockets that the restconf daemon listens to with the following fields:
socket namespace
   Network namespace
socket address
   IP address to bind to
socket port
   TCP port to bind to
socket ssl
   If true: HTTPS; if false: HTTP protocol

Examples
--------
Configure a single HTTP on port 80 in the default config file::

  <clixon-config xmlns="http://clicon.org/config">
     <CLICON_CONFIGFILE>/usr/local/etc/clixon.xml</CLICON_CONFIGFILE>
     ...
     <restconf>
        <enable>true</enable>
        <auth-type>user</auth-type>
        <socket>
	   <description>HTTP listen</description>
           <namespace>default</namespace>
           <address>0.0.0.0</address>
           <port>80</port>
           <ssl>false</ssl>
        </socket>
     </restconf>
  </clixon-config>

Configure two HTTPS listeners in two different namespaces::

   <restconf xmlns="https://clicon.org/restconf">
      <enable>true</enable>
      <auth-type>client-certificate</auth-type>
      <server-cert-path>/etc/ssl/certs/clixon-server-crt.pem</server-cert-path>
      <server-key-path>/etc/ssl/private/clixon-server-key.pem</server-key-path>
      <server-ca-cert-path>/etc/ssl/certs/clixon-ca_crt.pem</server-ca-cert-path>
      <socket>
	 <description>HTTPS listen</description>
         <namespace>default</namespace>
         <address>0.0.0.0</address>
         <port>443</port>
         <ssl>true</ssl>
      </socket>
      <socket>
      	 <description>HTTPS listen on myns</description>
         <namespace>myns</namespace>
         <address>0.0.0.0</address>
         <port>443</port>
         <ssl>true</ssl>
      </socket>
   </restconf>

SSL Certificates
----------------
If you use native RESTCONF you may want to have server/client
certs. If you use FCGI, certs are configured according to the reverse
proxy documentation, such as NGINX. The rest of this section applies to native restconf only.

If you already have certified server certs, ensure ``CLICON_SSL_SERVER_CERT`` and ``CLICON_SSL_SERVER_KEY`` points to them.

If you do not have them, you can generate self-signed certs, for example as follows::

   openssl req -x509 -nodes -newkey rsa:4096 -keyout /etc/ssl/private/clixon-server-key.pem -out /etc/ssl/certs/clixon-server-crt.pem -days 365

You can also generate client certs (not shown here) using ``CLICON_SSL_CA_CERT``. Example using client certs and curl for client `andy`::
  
   curl -Ssik --key andy.key --cert andy.crt -X GET https://localhost/restconf/data/example:x

Starting
========
You can start the RESTCONF daemon in several ways:

  1. `systemd` , externally started
  2. `internally` using the `process-control` RPC (see below)
  3. `docker` mechanisms, see the docker container docs

Start with Systemd
------------------
The Restconf service can be installed at, for example, /etc/systemd/system/example_restconf.service::
   
   [Unit]
   Description=Starts and stops an example clixon restconf service on this system
   Wants=example.service
   After=example.service
   [Service]
   Type=simple
   User=root
   Restart=on-failure
   ExecStart=/usr/local/sbin/clixon_restconf -f /usr/local/etc/example.xml
   [Install]
   WantedBy=multi-user.target

Internal start
--------------
For starting restconf internally, you need to enable ``CLICON_BACKEND_RESTCONF_PROCESS`` option. See Section `datastore`_.

Thereafter, you can either use the ``clixon-restconf.yang`` configuration or use the ``clixon-lib.yang`` process control RPC:s to start/stop/restart the daemon or query status.

The algorithm for starting and stopping the clixon-restconf internally is as follows:

  1. on RPC start, if enable is true, start the service, if false, error or ignore it
  2. on RPC stop, stop the service 
  3. on backend start make the state as configured
  4. on enable change, make the state as configured  

Example 1, using netconf `edit-config` to start the process::

  <?xml version="1.0" encoding="UTF-8"?>
  <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
     <capabilities><capability>urn:ietf:params:netconf:base:1.1</capability></capabilities>
  </hello>]]>]]>
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="10">
     <edit-config>
        <default-operation>merge</default-operation>
	<target><candidate/></target>
        <config>
	   <restconf xmlns="http://clicon.org/restconf">
	      <enable>true</enable>
	   </restconf>
	</config>
     </edit-config
  </rpc>
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="10">
     <ok/>
  </rpc-reply>
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="11">
     <commit/>
    </rpc>
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="10">
     <ok/>
  </rpc-reply>
  
Example 2, using netconf RPC to restart the process::

  <?xml version="1.0" encoding="UTF-8"?>
  <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
     <capabilities><capability>urn:ietf:params:netconf:base:1.1</capability></capabilities>
  </hello>]]>]]>
  <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="10">
     <process-control xmlns="http://clicon.org/lib">
        <name>restconf</name>
	<operation>restart</operation>
     </process-control>
  </rpc>
  <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="10">
     <pid xmlns="http://clicon.org/lib">1029</pid>
  </rpc-reply>

Note that the backend daemon must run as root (no lowering of privileges) to use this feature.

Plugin callbacks
================
Restconf plugins implement callbacks, some are same as for :ref:`backend plugins <clixon_backend>`.

init
   Clixon plugin init function, called immediately after plugin is loaded into the restconf daemon.
start
   Called when application is started and initialization is complete, and after drop privileges.
exit
   Called just before plugin is unloaded 
extension
  Called at parsing of yang modules containing an extension statement.
auth
  See `auth callback`_

Auth callback
-------------
The role of the authentication callback is, given a message (its headers) and authentication type, determine if the message passes authentication and return an associated user.

The auth callback is invoked after incoming processing, including cert validation, if any, but before relaying the message to the backend for NACM checks and datastore processing.

If the message is not authenticated, an error message is returned with
tag: `access denied` and HTTP error code `401 Unauthorized`.
   
There are default handlers for TLS client certs and for "none" authentication. But other variants, such as http basic authentication, oauth2 or the remapping of client certs to NACM usernames, can be implemented by this callback

If the message is authenticated, a user is associated with the message. This user can be derived from the headers or mapped in an application-dependent way. This user is used internally in Clixon and sent via the IPC protocol to the backend where it may be used for NACM authorization.

The signature of the auth callback is as follows::

  int ca_auth(clixon_handle h, void *req, clixon_auth_type_t auth_type, char **authp);

where:  

h
   Clixon handle
req
   Per-message request www handle to use with restconf_api.h
auth-type
   Specifies how the authentication is made and what default value 
authp
   NULL if credentials failed, otherwise malloced string of authentoicated user

The return value is one of:

- -1: Fatal error, close socket
- 0: Ignore, undecided, not handled, same as no callback. Fallback to default handler.
- 1: OK see authp parameter whether the result is authenticated or not, and the associated user.
 
If there are multiple callbacks, the first result which is not "ignore" is returned. This is to allow for different callbacks registering different classes, or grouping of authentication.
  
The main example contains example code.



FCGI
====
This section describes the RESTCONF FCGI mode using NGINX.

You need to configure the following:

  1. Configure clixon with ``--with-restconf=fcgi``
  2. Restconf config in the Clixon config file
  3. Reverse proxy configuration
  4. Start the restconf daemon (see `starting`_)

Restconf config
---------------

The restconf daemon can be started in several ways as described in Section `auth types`_. In all cases however, the configuration is simpler than in native mode. For example::

   <clixon-config xmlns="http://clicon.org/config">
      ...
      <CLICON_FEATURE>clixon-restconf:fcgi</CLICON_FEATURE>
      <restconf>
         <enable>true</enable>
         <fcgi-socket>/wwwdata/restconf.sock</fcgi-socket>
      </restconf>
   </clixon-config>

Reverse proxy config
--------------------     
If you use FCGI, you need to configure a reverse-proxy, such as NGINX. A typical configuration is as follows::

  server {
    ...
    location / {
      fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
      include fastcgi_params;
    }
  }

where ``fastcgi_pass`` setting should match  by ``fcgi-socket`` in ``clixon-config/restconf``.


Callhome
========
Clixon supports RESTCONF callhome according to `RFC 8071: NETCONF Call Home and RESTCONF Call Home <http://www.rfc-editor.org/rfc/rfc8071.txt>`_ using native RESTCONF and TLS and server/client certs.

Overview
--------
::

    device/server (n)                            client      (n)
  +-----------------+    (1) connect     +---------------------+
  |                 |  --------------->  |                     |
  | clixon-restconf |     (2) TLS        |   callhome-client   |  
  |                 | <---------------   |                     |
  |                 |     (3) data       |                     |
  |                 | <---------------   |                     |
  +-----------------+                    +---------------------+
          | (4) IPC
          v                             
  +-----------------+
  | clixon-backend  |
  +-----------------+                   

The operation of RESTCONF callhome is as follows:
  1. The RESTCONF server initiates a TCP connection to a client, either persistently or periodically
  2. The client sets up a TLS connection to the server using the existing TCP session
  3. The client sends data as HTTP requests over TLS to the server
  4. The RESTCONF server receives data, authenticates the client-cert, transforms the request to NETCONF and sends it internally to the clicon backend.
  5. Status replies are returned to the client

.. note::
   Clixon does not implement client-side call-home functionality, only server-side
     
Callback clients
----------------
A server may configure multiple HTTP callback clients, for fault-tolerance purposes, for example.

A callback "controller" client typically serves multiple servers.

Bootstrap controller
^^^^^^^^^^^^^^^^^^^^
A typical callback client scenario is a bootstrap controller. The HTTP requests
by the client are stored waiting for an inital connect from a
minimally deployed server. The controller may then store pre-configured
configurations. In such a scenario, the server needs at least the
following initial information:

   1. A callback IP address
   2. A server cert
   3. A CA to validate the client cert OR a list of exactly matching client-certs

Once the controller accepts a connection from a server, it may send
RESTCONF requests over HTTP and fully configure the new server.

Such a controller can also be used for more "intelligent"
configuration as well, such as setting up tunnels or other
configurations spanning multiple servers.

The controller may also provide an interactive CLI or GUI for example once a connection is established.

Description
-----------
The callhome function is "internal" in the sense that it is
integrated in Clixon and uses the openssl lib and extends the regular
"listen" RESTCONF functionality.

The callhome mechanism is a server-side implementation. There is an
example client-side implementation
(`util/clixon_restconf_callhome_client.c`) which is not a part of the
actual Clixon code. A user needs to write a client to use this
functionality.

The existing clixon-restconf YANG has been extended to support
callhome. The ietf restconf server draft (`<https://datatracker.ietf.org/doc/html/draft-ietf-netconf-restconf-client-server-26>`_)
which is used as a basis for the extensions. While not complying to the draft's
structure, the YANG fields covering callhome are similar. Please see
the draft for detailed description of scenarios and configuration
fields.

The callhome features include:

* `Persistent` and `periodic` connection types, ie continuous callhome attempts, or at specific time intervals.
* Periodic connections support `idle-timeout`, ie close the TCP connection if no traffic after timeout.
* `Max-attempts` reconnect strategy, ie how many times to retry a connect attempt before timeout

Setup
-----
A callhome session is setup by adding a ``call-home`` section to a native RESTCONF socket declaration. For example::

     <restconf xmlns="https://clicon.org/restconf">
      <enable>true</enable>
      <auth-type>client-certificate</auth-type>
      <server-cert-path>/etc/ssl/certs/clixon-server-crt.pem</server-cert-path>
      <server-key-path>/etc/ssl/private/clixon-server-key.pem</server-key-path>
      <server-ca-cert-path>/etc/ssl/certs/clixon-ca_crt.pem</server-ca-cert-path>
      <socket>
         <description>callhome session</description>
         <namespace>default</namespace>
         <address>12.13.14.15</address>
         <port>4336</port>
         <ssl>true</ssl>
	 <call-home>                
	   <!-- ... call-home section ... -->
	 </call-home>
      </socket>

Some notes for callhome sockets:

1. The ``address`` field denotes a remote client, not the server which is the case for "listen" sockets.
2. The default ``port`` for RESTCONF callhome is `4336`
3. A callhome socket must have ``ssl`` enabled
4. Client certs must be used as ``auth-type``
5. You can mix regular "listen" sockets with "callhome" sockets.
6. You can have multiple (concurrent) callhome sockets.

Persistent connection
---------------------
If the callhome session is persistent, the server tries to hold the connection open at all times. The default re-connect strategy is 1 second.

Example socket configuration::

   <call-home>
      <connection-type>
         <persistent/>
      </connection-type>
   </call-home>

Periodic connection
-------------------
Periodic call-home sessions try to establish a callhome connections at regular intervals, such as once a minute, or once a day.

Example periodic configuration::

   <call-home>
      <connection-type>
         <periodic>
  	    <period>3600</period>
	    <idle-timeout>60</idle-timeout>
	  </periodic>
      </connection-type>
      <reconnect-strategy>
         <max-attempts>3</max-attempts>
      </reconnect-strategy>
   </call-home>

Notes:

  1. The `period` is in seconds, while the draft uses minutes. The example therefore shows a period of one hour.
  2. The `idle-timeout` field means that the TCP session is closed by the server if no data is sent in 1 minute.
  3. The `max-attempts` setting means that every start of period the server makes several attempts to reconnect. If all attempts fail, it waits another period before re-trying.
  4. If the connection is active when the new period starts, no reconnect attempt is made.

HTTP data
=========

As an extension to the native restconf implementation, Clixon provides a
limited http-data server for displaying web pages. To use the http-data feature, you need
to configure clixon with ``--with-restconf=native`` and also enable
the ``http-data`` feature as described below.

.. note::
      The http data feature is limited and can only be used for very simple web content

Configuration options
---------------------
First, you also need to define ``<CLICON_FEATURE>clixon-restconf:http-data</CLICON_FEATURE>`` to enable http-data.

The following configuration options can be defined in the clixon configuration file:

CLICON_HTTP_DATA_ROOT
   Directory in the local file system where http-data files are searched for. Soft links, ``..``, ``~`` etc are not followed. Default is ``/var/www``.
CLICON_HTTP_DATA_PATH
   Prefix to match with URI to match for http-data. This path is appended to ``CLICON_HTTP_DATA_ROOT`` to find a matching file. Default is ``/``. Note that the restconf match prefix is ``/restconf``.

Example
^^^^^^^
The following is an example of a config file for setting up an http-data service at ``/var/www/data``::

  <clixon-config xmlns="http://clicon.org/config">
    <CLICON_FEATURE>clixon-restconf:http-data</CLICON_FEATURE>
    <CLICON_HTTP_DATA_ROOT>/var/www</CLICON_HTTP_DATA_ROOT>
    <CLICON_HTTP_DATA_PATH>/data</CLICON_HTTP_DATA_PATH>
    ...

An example curl call could be::

    curl -X GET -H 'Accept: text/html' http://localhost/data/

The call will retrieve the file at ``/var/www/data/index.hmtl``.
    
Features and limitation
-----------------------

The http server is very limited in functionality:

  * No dynamic pages, ie backend scripts, only static pages are supported.
  * Operations are limited to GET, HEAD, and OPTIONS
  * Query parameters are not supported
  * Indata is not supported
  * Supported media is: html, css, js, fonts, and some images. All other content is default `octet-stream`

All applicable features of the native restconf implementation are available for the http-data as well. This includes http/1 and http/2 and TLS using openssl.

There is support for URI path internal redirect to a file called
`index.html`. This can be changed by compile-time option ``HTTP_DATA_INTERNAL_REDIRECT``.

RESTCONF streams
================
Clixon has an experimental RESTCONF event stream implementations following
RFC8040 Section 6 using Server-Sent Events (SSE).  Currently this is implemented in FCGI/Nginx only (not native).

.. note::
        RESTCONF streams are experimental and only implemented for FCGI.

Example: set the Clixon configuration options::

  <CLICON_STREAM_PATH>streams</CLICON_STREAM_PATH>
  <CLICON_STREAM_URL>https://example.com</CLICON_STREAM_URL>
  <CLICON_STREAM_RETENTION>3600</CLICON_STREAM_RETENTION>

In this example, the stream ``example`` is accessed with ``https://example.com/streams/example``.

Clixon defines an internal in-memory (not persistent) replay function controlled by the configure option above.  In this example, the retention is configured to 1 hour, i.e., the stream replay function will only save timeseries one hour, but if the restconf daemon is restarted, the history will be lost.

In the Nginx configuration, add the following to extend the nginx configuration file with the following statements (for example)::

	location /streams {
	    fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
	    include fastcgi_params;
 	    proxy_http_version 1.1;
	    proxy_set_header Connection "";
        }

An example of a stream access is as follows::

  curl -H "Accept: text/event-stream" -s -X GET http://localhost/streams/EXAMPLE
  data: <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0"><eventTime>2018-11-04T14:47:11.373124</eventTime><event><event-class>fault</event-class><reportingEntity><card>Ethernet0</card></reportingEntity><severity>major</severity></event></notification>
  data: <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0"><eventTime>2018-11-04T14:47:16.375265</eventTime><event><event-class>fault</event-class><reportingEntity><card>Ethernet0</card></reportingEntity><severity>major</severity></event></notification>

You can also specify start and stop time. Start-time enables replay of existing samples, while stop-time is used both for replay, but also for stopping a stream at some future time::

   curl -H "Accept: text/event-stream" -s -X GET http://localhost/streams/EXAMPLE?start-time=2014-10-25T10:02:00&stop-time=2014-10-25T12:31:00

Fcgi stream options
-------------------
The following options apply only for fcgi mode and notification streams:

CLICON_STREAM_DISCOVERY_RFC8040
  Enable monitoring information for the RESTCONF protocol from RFC 8040 (only fcgi)

CLICON_STREAM_PATH  
  Stream path appended to CLICON_STREAM_URL to form stream subscription URL (only fcgi)
