.. _clixon_nacm:

====
NACM
====

Clixon implements the `Network Configuration Access Control Model
<http://www.rfc-editor.org/rfc/rfc8341.txt>`_ (NACM / RFC8341).
NACM rpc and datanode access validation is supported, not outgoing notifications.

NACM rules apply to all datastores.

Restrictions
============

Access notification authorization (Sec 3.4.6) is NOT implemented.

Data-node paths, eg ``<rule>...<path>ex:table/ex:parameter</path></rule>`` instance-identifiers are restricted to canonical namespace identifiers for both XML and JSON encoding. That is, if a symbol (such as ``table`` above) is a symbol in a module with prefix ``ex``, another prefix cannot be used, even though defined with a ``xmlns`` rule.

Config options
==============
The following configuration options are related to NACM:

CLICON_NACM_MODE
  NACM mode is either: `disabled`, `internal`, or `external`. Default: `disabled`.

CLICON_NACM_FILE
  If NACM mode is external, this file contains the NACM config.

CLICON_NACM_CREDENTIALS
  Verify NACM user credentials with unix socket peer credentials.  This means that a NACM user must match a UNIX user accessing ``CLIXON_SOCK``. Credentials are either: `none`, `exact` or `except`. Default: `except`.

CLICON_NACM_RECOVERY_USER
   RFC8341 defines a 'recovery session' as outside its scope. Clixon defines this user as having special admin rights to exempt from all access control enforcements.

CLICON_NACM_DISABLED_ON_EMPTY
   RFC 8341 defines enable-nacm as true by default. Since also write-default is deny by default it leads to that empty configs can not be edited. Default: `false`.
  
Mode
====

NACM rules are either internal or external. If external, rules are loaded from a separate file, specified by the option ``CLICON_NACM_FILE``.

If the NACM mode is internal, the NACM configuration is a part of the
regular candidate/running datastore. NACM rules are read from the
`running` datastore, ie they need to be committed. 

Since NACM rules are part of the config itself it means that there may
be bootstrapping issues. In particular, NACM default is `enabled` with
read/exec permit, and write `deny`. Loading an empty config therefore
leads to a "deadlock" where no user can edit the datastore.

Work-arounds include restarting the backend with a NACM config in the startup db, or using a `recovery user`_.

Access control
==============

NACM is implemented in the Clixon backend at:

* Incoming RPC (module-name/protocol-operation)
* Before modifying the data store (data create/delete/update)
* After retrieving data (data read)

User credentials
----------------
  
Access control relies on a user and groups. When an internal Clixon
client communicates with the backend, it piggybacks the name of the
user in the request, See :ref:`Internal netconf username
<clixon_misc>`::

  <rpc username="myuser"><get-config><source><running/></source></get-config></rpc>

The authentication of the username needs to be done in the client by either SSL certs (such as in :ref:`RESTCONF auth callback <clixon_restconf>`) or by SSH (as in NETCONF/CLI over SSH).

The Clixon backend can check credentials of the client if it uses a
UNIX socket (not IP socket) for internal communication between clients
and backend. In this way, a username claimed by a client can be verified against the UNIX user credentials.

The allowed values of `CLICON_NACM_CREDENTIALS` is:

* `none`: Do not match NACM user to any user credentials. Any user can pose as any other user. Set this for IP sockets, or do not use NACM.
* `exact`: Exact match between NACM user and unix socket peer user. 
* `except`: Exact match between NACM user and unix socket peer user, `except` for root and `wwwuser`. This is default.


Recovery user
=============

RFC 8341 defines a NACM emergency recovery session mechanism.  Clixon
implements a recovery user set by option
``CLICON_NACM_RECOVERY_USER``. If a client accesses the backend as
that user, all NACM rules will be bypassed. By default there is no such
user.

Moreover, this mechanism is controlled by `user credentials`_ which means
you can control who can act as the recovery user.

For example, by setting ``CLICON_NACM_CREDENTIALS`` to `except` the
RESTCONF daemon can make backend calls posing as the recovery user,
even though it runs as `www-data`.

Alternatively, ``CLICON_NACM_CREDENTIALS`` can be set to `exact` and
the recovery user as `root`, in which case only a netconf or cli
session running as root can make recovery operations.
