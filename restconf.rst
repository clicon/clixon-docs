.. _clixon_restconf:

RESTCONF
========

.. This is a comment
   
Clixon supports two RESTCONF compile-time variants: *FCGI* and *Native*. 
   
Architecture
------------

 .. image:: restconf1.jpg
   :width: 100%

The restconf deamon provides a http/https RESTCONF interface to the
Clixon backend.  It comes in two variants, as shown in the figure above:

  1. Native http, which combines a web and restconf server.
  2. A reverse proxy (such as NGINX) and fastCGI where web and restconf function is separated

The restconf daemon communicates with the backend using
internal netconf over the ``CLIXON_SOCK``. If FCGI is used, there is also a FCGI socket specified by ``CLICON_RESTCONF_PATH``.

The restconf daemon reads its initial config options from the configuration file on startup. The native http variant can read config options from the backend as an alternative to reading everything from clixon options.

You can add plugins to the restconf daemon, where the primary usecase is authentication, using the ``ca_auth`` callback.


Installation
------------

The RESTCONF daemon can be configured for compile-time (by autotools) as follows:
  --without-restconf      No RESTCONF
  --with-restconf=fcgi    RESTCONF using fcgi/ reverse proxy. This is default. Note fcgi feature must be set
  --with-restconf=native  RESTCONF using native http with libevhtp

After that perform system-wide compilation::

    make && sudo make install
  
Command-line options
--------------------

The restconf daemon have the following command-line options:
  -h              Help
  -D <level>      Debug level
  -f <file>       Clixon config file
  -E <dir>        Extra configuration directory
  -l <option>     Log on (s)yslog, std(e)rr, std(o)ut or (f)ile. Syslog is default. If foreground, then syslog and stderr is default. Filename is given after -f as follows: ``-lf<file>``.
  -p <dir>        Yang directory path (see CLICON_YANG_DIR)
  -b <dir>        Specify XMLDB database directory
  -a <family>     Internal backend socket family: UNIX|IPv4|IPv6
  -u <path|addr>  Internal socket domain path or IP addr (see -a)(default: /usr/var/hello.sock)
  -r              Do not drop privileges if run as root
  -W <user>       Run restconf daemon as this user, drop according to CLICON_RESTCONF_PRIVILEGES
  -o <option=value>  Give configuration option overriding config file (see clixon-config.yang)

Note that the restconf daemon can be started as root, drops privileges to `wwwuser`, unless the ``-r`` command-line option is used.

Configuration
-------------
The following RESTCONF configuration options can be defined in the clixon configuration file:

CLICON_RESTCONF_DIR
   Location of restconf .so plugins. Load all .so plugins in this dir as restconf code plugins.
CLICON_BACKEND_RESTCONF_PROCESS
   Start restconf daemon internally from backend daemon. The restconf daemon reads its config from the backend running datastore. 
CLICON_ANONYMOUS_USER
   If RESTCONF authentication auth-type=none then use this user

Config file
^^^^^^^^^^^

The remaining options are defined in `clixon-restconf.yang` where they are defined locally within the clixon config file::

  <CLICON_FEATURE>clixon-restconf:fcgi</CLICON_FEATURE>
  <CLICON_BACKEND_RESTCONF_PROCESS>false/CLICON_BACKEND_RESTCONF_PROCESS>
  <clixon-config xmlns="http://clicon.org/config">
     <restconf>
        <enable>true</enable>
	<fcgi-socket>/wwwdata/restconf.sock</fcgi-socket>
     </restconf>
   </clixon-config>

Datastore
^^^^^^^^^

Alternatively if ``CLICON_BACKEND_RESTCONF_PROCESS`` is set, the restconf configuration is::

  <CLICON_FEATURE>clixon-restconf:fcgi</CLICON_FEATURE>
  <CLICON_BACKEND_RESTCONF_PROCESS>false/CLICON_BACKEND_RESTCONF_PROCESS>

And the detailed restconf is defined in the regular running datastore by adding something like::

   <restconf xmlns="http://clicon.org/restconf">
      <enable>true</enable>
      <fcgi-socket>/wwwdata/restconf.sock</fcgi-socket>
   </restconf>
   
In the latter case, the restconf daemon reads its config from the running datastore on startup. 

.. note::
      If ``CLICON_BACKEND_RESTCONF_PROCESS`` is enabled, the restconf config must be in the regular datastore.
   
The restconf options are as follows:

enable
   Enable the RESTCONF daemon. If disabled, the restconf daemon will not start
auth-type
   Authentication method (see `auth types`_)
debug
   Enable to debug on syslog
pretty
   Restconf vallues are pretty printed by default. Disable to turn this off

Features
^^^^^^^^
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
^^^^^^^^^^
The RESTCONF daemon uses the following authentication types:

none
   Messages are not authenticated and set to the value of ``CLICON_ANONYMOUS_USER``. A callback can revise this behavior. Note, must set `allow-auth-none` feature.
client-cert
   Set to authenticated and extract the username from the SSL_CN parameter. A callback can revise this behavior.
user
   User-defined behaviour as implemented by the `auth callback`_. Typically done by basic auth, eg HTTP_AUTHORIZATION header, and verify password

FCGI mode
^^^^^^^^^
Applies if clixon is configured with ``--with-restconf=fcgi``. Fcgi-specific config options are:

fcgi-socket
   Path to FCGI unix socket. This path should be the same as specific in fcgi reverse proxy

Native mode
^^^^^^^^^^^
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
^^^^^^^^
Configure a single HTTP on port 80 in the default config file::

  <clixon-config xmlns="http://clicon.org/config">
     <CLICON_CONFIGFILE>/usr/local/etc/clixon.xml</CLICON_CONFIGFILE>
     ...
     <restconf>
        <enable>true</enable>
        <auth-type>user</auth-type>
        <socket>
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

SSL Certificates
^^^^^^^^^^^^^^^^
If you use native RESTCONF you may want to have server/client
certs. If you use FCGI, certs are configured according to the reverse
proxy documentation, such as NGINX. The rest of this section applies to native restconf only.

If you already have certified server certs, ensure ``CLICON_SSL_SERVER_CERT`` and ``CLICON_SSL_SERVER_KEY`` points to them.

If you do not have them, you can generate self-signed certs, for example as follows::

   openssl req -x509 -nodes -newkey rsa:4096 -keyout /etc/ssl/private/clixon-server-key.pem -out /etc/ssl/certs/clixon-server-crt.pem -days 365

You can also generate client certs (not shown here) using ``CLICON_SSL_CA_CERT``. Example using client certs and curl for client `andy`::
  
   curl -Ssik --key andy.key --cert andy.crt -X GET https://localhost/restconf/data/example:x

Starting
--------
You can start the RESTCONF daemon in several ways:

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
----------------
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
^^^^^^^^^^^^^

The role of the authentication callback is, given a message (its headers) and authentication type, determine if the message passes authentication and return an associated user.

The auth callback is invoked after incoming processing, including cert validation, if any, but before relaying the message to the backend for NACM checks and datastore processing.

If the message is not authenticated, an error message is returned with
tag: `access denied` and HTTP error code `401 Unauthorized`.
   
There are default handlers for TLS client certs and for "none" authentication. But other variants, such as http basic authentication, oauth2 or the remapping of client certs to NACM usernames, can be implemented by this callback

If the message is authenticated, a user is associated with the message. This user can be derived from the headers or mapped in an application-dependent way. This user is used internally in Clixon and sent via the IPC protocol to the backend where it may be used for NACM authorization.

The signature of the auth callback is as follows::

  int ca_auth(clicon_handle h, void *req, clixon_auth_type_t auth_type, char **authp);

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
----
This section describes the RESTCONF FCGI mode using NGINX.

If you use FCGI, you need to configure a reverse-proxy, such as NGINX. A typical configuration is as follows::

  server {
    ...
    location / {
      fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
      include fastcgi_params;
    }
  }

where ``fastcgi_pass`` setting must match ``CLICON_RESTCONF_PATH``.

RESTCONF streams
----------------

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
^^^^^^^^^^^^^^^^^^^
The following options apply only for fcgi mode and notification streams:

CLICON_STREAM_DISCOVERY_RFC8040
  Enable monitoring information for the RESTCONF protocol from RFC 804 (only fcgi)

CLICON_STREAM_PATH  
  Stream path appended to CLICON_STREAM_URL to form stream subscription URL (only fcgi)
