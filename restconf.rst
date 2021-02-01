.. _clixon_restconf:

RESTCONF
========

.. This is a comment
   
Clixon supports two RESTCONF compile-time variants: *FCGI* and *Native*. 
   
Architecture
------------
::

                                      +---------------------------------+
                                      |        clixon options           |
                                      +---------------------------------+
                                          |- XML                   |- XML
                        FCGI              v                        v
        (1)  +-------+    |    +----------+--------+   IPC    +----------+
 User  <-->  | nginx |  <--->  | restconf | plugin |    |     | backend  |
             +-------+         | daemon   |--------+  <--->   | daemon   |
 User  <-------------------->  |          | plugin |          |          |
        (2)     "native"       +----------+--------+          +----------+

The restconf deamon provides a http/https RESTCONF interface to the
Clixon backend.  It comes in two variants, as shown by (1) and (2) in the figure above:

  1. A reverse proxy (such as NGINX) and fastCGI where web and restconf function is separated
  2. Native http using libevhtp, which combines a web server and restconf handler.

The restconf daemon communicates with the backend using
internal netconf over the ``CLIXON_SOCK``. If FCGI is used, there is also a FCGI socket specified by ``CLICON_RESTCONF_PATH``.

The restconf daemon reads its initial config options from the configuration file on startup. The native http variant can read config options from the backend as an alternative to reading everything from clixon options.

You can add plugins to the restconf daemon, where the primary usecase is authentication, using the ``ca_auth`` callback.


Configuration
-------------

The RESTCONF daemon can be configured (by autotools) as follows:
  --without-restconf      No RESTCONF
  --with-restconf=fcgi    RESTCONF using fcgi/ reverse proxy. This is default.
  --with-restconf=evhtp   RESTCONF using native http with libevhtp
  --with-wwwuser=<user>   Set www user different from ``www-data``

Command-line options
--------------------

The restconf daemon have the following command-line options:
  -h              Help
  -D <level>      Debug level
  -f <file>       Clixon config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default. Filename is given after -f: -lf<file>.
  -d <dir>        Specify backend plugin directory (default: none)
  -p <dir>        Yang directory path (see CLICON_YANG_DIR)
  -b <dir>        Specify XMLDB database directory
  -a <family>     Internal backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal socket domain path or IP addr (see -a)(default: /usr/var/hello.sock)
  -r              Do not drop privileges if run as root\n"
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

Options
^^^^^^^
The configuration file options related to RESTCONF common to both fcgi and evhtp are the following:

CLICON_RESTCONF_DIR
   Location of restconf .so plugins. Load all .so plugins in this dir as restconf code plugins.

CLICON_RESTCONF_PATH
   FCGI unix socket. Should be specified in webserver (only fcgi)

CLICON_RESTCONF_PRETTY
   RESTCONF return value is pretty-printed or not

Starting
--------
The restconf daemon (``clixon_restconf``) can be started as root, but in that case drops privileges to wwwuser, unless the ``-r`` command-line option is used.

You can start the restconf daemon in several ways:

  1. `systemd` as described in :ref:`clixon_install`
  2. `internally` using the `process-control` RPC (see below)
  3. `docker` mechanisms, see the docker container docs

Internal start
^^^^^^^^^^^^^^
For starting restconf internally, you need to enable ``CLICON_BACKEND_RESTCONF_PROCESS`` option

Thereafter, you can either use the ``clixon-restconf.yang`` configuration or use the ``clixon-lib.yang`` process control RPC:s to start/stop/restart the daemon or query status.

The algorithm for starting and stopping the clixon-restconf internally is as follows:

  1. on RPC start, if enable is true, start the service, if false, error or ignore it
  2. on RPC stop, stop the service 
  3. on backend start make the state as configured
  4. on enable change, make the state as configured  

Example 1, using netconf `edit-config` to start the process::

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
      
FCGI mode
---------
This section describes the RESTCONF FCGI mode using NGINX.

NGINX
^^^^^
If you use FCGI, you need to configure a reverse-proxy, such as NGINX. A typical configuration is as follows::

  server {
    ...
    location / {
      fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
      include fastcgi_params;
    }
  }

where ``fastcgi_pass`` setting must match ``CLICON_RESTCONF_PATH``.

Fcgi stream options
^^^^^^^^^^^^^^^^^^^
The following options apply only for fcgi mode and notification streams:

CLICON_STREAM_DISCOVERY_RFC8040
  Enable monitoring information for the RESTCONF protocol from RFC 804 (only fcgi)

CLICON_STREAM_PATH  
  Stream path appended to CLICON_STREAM_URL to form stream subscription URL (only fcgi)

Native http mode
----------------
This only applies if you have chosen ``--with-restconf=evhtp``.

You need to have ``libevhtp`` installed. See :ref:`clixon_install`.

Configuration of native http has more options than reverse proxy, since it contains web-fronting parts, including socket(address, ports) and certificates, where these part of Nginx. These options are defined in in ``clixon-restconf.yang``.

There are two ways to configure the socket and certificates of native http:

  1. Local configure (clixon-config), where clixon-restconf.yang options can be included.
  2. From clixon backend as a second step after loading initial config from clixon-config.
     
In the case of (1) example HTTP on port 80 (note multiple sockets can be configured)::

  <clixon-config xmlns="http://clicon.org/config">
     <CLICON_CONFIGFILE>/usr/local/etc/clixon.xml</CLICON_CONFIGFILE>
     ...
     <restconf>
        <enable>true</enable>
        <auth-type>password</auth-type>
        <socket>
           <namespace>default</namespace>
           <address>0.0.0.0</address>
           <port>80</port>
           <ssl>false</ssl>
        </socket>
     </restconf>
  </clixon-config>

In the case of (2) example with ssl ::

   <restconf xmlns="https://clicon.org/restconf">
      <enable>true</enable>
      <auth-type>client-certificate</auth-type>
      <server-cert-path>/etc/ssl/certs/clixon-server-crt.pem</server-cert-path>
      <server-key-path>/etc/ssl/private/clixon-server-key.pem</server-key-path>
      <server-ca-cert-path>/etc/ssl/certs/clixon-ca_crt.pem</server-ca-cert-path>
      <socket>
         <namespace>default</namespace>
         <address>0.0.0.0</address>
         <port>443</port>
         <ssl>true</ssl>
      </socket>
      <socket>
         <namespace>myns</namespace>
         <address>0.0.0.0</address>
         <port>443</port>
         <ssl>true</ssl>
      </socket>
   </restconf>

In the latter case, ``clixon_restconf.yang`` should be imported, and these settings must be
present in the running datastore `before` the restconf daemon is
started. This can be done via the startup datastore or by editing the
running config before restconf daemon.

The example also includes a socket that listen to another network namespace ``myns``, which must exist and have networking setup so that a socket can be bound.

Plugin callbacks
----------------
Restconf plugins implement callbacks, some are same as for :ref:`backend plugins <clixon_backend>`. Most important is the ``auth`` callback where user authentication can be implemented.

init
   Clixon plugin init function, called immediately after plugin is loaded into the restconf daemon.
start
   Called when application is started and initialization is complete, and after drop privileges.
exit
   Called just before plugin is unloaded 
extension
  Called at parsing of yang modules containing an extension statement.
auth
  Called by restconf on each incoming request to check credentials and return username. This is done after cert validation, if any. For example, http basic authentication, oauth2 or just matching client certs with username can be implemented here.


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

RESTCONF streams
----------------

Clixon has an experimental RESTCONF event stream implementations following
RFC8040 Section 6 using Server-Sent Events (SSE).  Currently this is implemented in FCGI/Nginx only (not evhtp).

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

