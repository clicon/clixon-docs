.. _clixon_nacm:

====
NACM
====

Clixon implements the `Network Configuration Access Control Model
<http://www.rfc-editor.org/rfc/rfc8341.txt>`_ (NACM / RFC8341).
NACM rpc and datanode access validation is supported, but not outigoing notifications.

Config options
==============
A couple of config options control files related to the backend, as follows:

CLICON_NACM_MODE
  NACM mode is either: disabled, internal, or external. Default: disabled.

CLICON_NACM_FILE
  If NACM mode is external, this file contains the NACM config.

CLICON_NACM_CREDENTIALS
  Verify NACM user credentials with unix socket peer cred.  This means nacm user must match unix user accessing the backend socket. Credentials are either: none, exact or except. Default: `except`.

CLICON_NACM_RECOVERY_USER
   RFC8341 defines a 'recovery session' as outside its scope. Clixon defines this user as having special admin rights to exempt from all access control enforcements.

CLICON_NACM_DISABLED_ON_EMPTY
   RFC 8341 defines enable-nacm as true by default. Since also write-default is deny by default it leads to that empty configs can not be edited. Default: `false`.
  
Access control
==============

NACM is implemented in the Clixon backend at:

* Incoming RPC (module-name/protocol-operation)
* When (before) modifying the data store (data create/delete/update)
* When (after) retreiving data (data read)

Users
-----
  
Access control relies on a user and groups. When an internal Clixon
client communicates with the backend, it piggybacks the name of the
user in the request, See :ref:`Internal netconf username
<clixon_misc>`::

  <rpc username="myuser"><get-config><source><running/></source></get-config></rpc>

The authentication of the username itself is primarily assumed to be
done in the client by either SSL certs (such as in RESTCONF) or by ssh (as
in NETCONF over ssh or CLI over ssh).

Credentials
-----------
The Clixon backend can check credentials of the client if it uses a
UNIX socket (not IP socket) for internal communication between clients
and backend. In this way, a username claimed by a client can be verified against the UNIX user credentials.

The allowed values of `CLICON_NACM_CREDENTIALS` is:

* `none`: Dont match NACM user to any user credentials. Any user can pose as any other user. Set this for IP sockets, or dont use NACM.
* `exact`: Exact match between NACM user and unix socket peer user. Except for root that can pose as any user.
* `except`: Exact match between NACM user and unix socket peer user, except for root and www user (restconf). This is default.

Internal vs External mode
=========================

NACM rules are either internal or external. If external, rules are loaded from a separate NACM conmfig file, as specified by the option `CLICON_NACM_FILE`.

Internal
--------
If the NACM mode is internal, the NACM configuration is a part of the
regular candidate/running datastore. NACM rules are read from
`running`.

Since NACM rules are part of the config itself it means that there may
be bootstrapping issues. In particular, NACM default is `enabled` with
read/exec permit, and write `deny`. Loading an empty config therefore
leads to a "deadlock" where no user can edit the datastore.

Work-arounds include restarting the backend with a NACM config in the startup db, or using a `recovery user`_.

Recovery user
=============

RFC 8341 defines a NACM emergency recovery session mechanism.  Clixon implements a "recovery user" set by option `CLICON_NACM_RECOVERY_USER`. If you access the backend (ie in an internal NETCONF session) as that user, you will bypass all NACM rules. By default there is no such user.

Moreover, this mechanism is controlled by `credentials`_ which means
you cannot mimic the recovery user since the backend controls which
actual peer user the client is logged in as.  However, for RESTCONF
you need to be able to proxy as a different user, which means there is
an exception for root and wwwuser.

In other words, the restconf daemon can make backend calls posing as
other (eg auhenticated) users.  The `CLICON_NACM_RECOVERY_USER` could be
set to root, for example, or to an authenticated user, or keep the
default non-setting in which case you need to restart the backend with
a proper startup datastore.
