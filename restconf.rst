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
        (1)  +-------+    |    +----------+--------+  Netconf +----------+
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

The restconf daemon reads its initial config options from the configuration file on startup. The native http variant can read config options from the backend using the -b option as an alternative to reading everything from clixon options.

You can add plugins to the restconf daemon, where the primary usecase is authentication, using the ``ca_auth`` callback.


Configuration
-------------

The RESTCONF daemon can be configured (by autotools) as follows:
  --without-restconf      No RESTCONF
  --with-restconf=fcgi    RESTCONF using fcgi/ reverse proxy. This is default.
  --with-restconf=evhtp   RESTCONF using native http with libevhtp
  --with-wwwuser=<user>   Set www user different from ``www-data``

The restconf daemon can be started as root, but in that case drops privileges to wwwuser.
  
Config options
--------------
The configuration file options related to RESTCONF are as follows:

CLICON_RESTCONF_DIR
   Location of restconf .so plugins. Load all .so plugins in this dir as restconf code plugins.

CLICON_RESTCONF_PATH
   FCGI unix socket. Should be specified in webserver (only fcgi)

CLICON_RESTCONF_PRETTY
   RESTCONF return value is pretty-printed or not

CLICON_RESTCONF_CONFIG
   For evhtp, get restconf-specific configuration from backend on startup instead of config-file.

Evhtp local options
^^^^^^^^^^^^^^^^^^^

The following options apply only for evhtp and local config (ie ``CLICON_RESTCONF_CONFIG=false``). If the config is loaded from the backend, settings in the datastore is used instead, see `Native http`_.

CLICON_RESTCONF_IPV4_ADDR
   RESTCONF IPv4 socket address to bind

CLICON_RESTCONF_IPV4_ADDR
   RESTCONF IPv6 socket address to bind

CLICON_RESTCONF_HTTP_PORT
   RESTCONF non-ssl socket binding port

CLICON_RESTCONF_HTTPS_PORT
   RESTCONF SSL socket binding port
   
CLICON_SSL_SERVER_CERT
  SSL server cert for restconf https

CLICON_SSL_SERVER_KEY
  SSL server private key for restconf https

CLICON_SSL_CA_CERT
  SSL CA cert for client authentication

Fcgi stream options
^^^^^^^^^^^^^^^^^^^
The following options apply only for fcgi and streams:

CLICON_STREAM_DISCOVERY_RFC8040
  Enable monitoring information for the RESTCONF protocol from RFC 804 (only fcgi)

CLICON_STREAM_PATH  
  Stream path appended to CLICON_STREAM_URL to form stream subscription URL (only fcgi)


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

NGINX
-----
If you use FCGI, you need to configure a reverse-proxy, such as NGINX. A typical configuration is as follows::

  server {
    ...
    location / {
      fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
      include fastcgi_params;
    }
  }

where ``fastcgi_pass`` setting must match ``CLICON_RESTCONF_PATH``.

Native http
-----------
You need to have ``libevhtp`` installed. See :ref:`clixon_install`.

Configuration of native http has more options than reverse proxy, since it contains web-fronting parts, including socket(address, ports) and certificates, where these part of Nginx.

There are two ways to configure the socket and certificates of native http:

  1. Local configure (see `Evhtp local options`_) 
  2. Load configure options from clixon backend using clixon-restconf.yang (after loading initial config from config file).
     
In the case of (2), fields from clixon-restconf.yang is used by setting fields in the regular datastore, for example ::

   <restconf xmlns="https://clicon.org/restconf">
      <socket>
         <namespace>default</namespace>
         <address>0.0.0.0</address>
         <port>443</port>
         <ssl>true</ssl>
      </socket>
      <auth-type>client-certificate</auth-type>
      <server-cert-path>/etc/ssl/certs/clixon-server-crt.pem</server-cert-path>
      <server-key-path>/etc/ssl/private/clixon-server-key.pem</server-key-path>
      <server-ca-cert-path>/etc/ssl/certs/clixon-ca_crt.pem</server-ca-cert-path>
   </restconf>

These settings must be present in the running datastore `before` the
restconf daemon is started. This can be done via the startup datastore or
by editing the running config before restconf daemon.


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

Clixon defines an internal in-memory (not persistent) replay function controlled by the configure option above.  In this example, the retention is configured to 1 hour, i.e., the stream replay function will only save timeseries one hour, but if the restconf daemon is restarted, the hisstory will be lost.

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

