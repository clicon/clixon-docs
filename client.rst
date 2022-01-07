.. _client_api:
.. sectnum::
   :start: 14
   :depth: 3

**********
Client API
**********

Clixon's client API provides a simple way to communicate with the
built-in XML datastore. This can be used to fetch or manipulate
configurations handled by Clixon from any other application
running on a host system. For example, a deamon running on a host system may need
to read a configured value from the Clixon datastore.

Clixon integration normally uses dynamic plugins, but the client API
shown here is an alternative.

Below, simple examples for both Clixon and Cisco ConfD are shown and compared
with the aim of replacing one with the other.


Comparison between Clixon and ConfD
===================================
Cisco ConfD is a well known configuration manager which also
maintains the configuration in its built-in datastore. Just like
Clixon, ConfD offers an API for configuration access and manipulation.

Applications that are integrated using ConfDs CDB API can be converted
to use Clixon instead. Below is a minimal example application
which connects to ConfD to get a value stored under
"/table/parameter" in the XML store.

ConfD example:
::

   #include <unistd.h>
   #include "confd_lib.h"
   #include "confd_cdb.h"

   #define CONFD_IPC_ADDR "127.0.0.1"
   #define CONFD_IPC_PORT  11004

   int main(int argc, char **argv)
   {
       int s = 0;
       u_int32_t n = 0;
       struct sockaddr_in addr;

       addr.sin_addr.s_addr = inet_addr(CONFD_IPC_ADDR);
       addr.sin_family = AF_INET;
       addr.sin_port = htons(CONFD_IPC_PORT);

       confd_init("server", stderr, CONFD_DEBUG);

       if ((s = socket(PF_INET, SOCK_STREAM, 0)) < 0){
	   confd_fatal("socket failed\n");
	   return -1;
       }

       if (cdb_connect(s, CDB_READ_SOCKET, (struct sockaddr*)&addr, sizeof(struct sockaddr_in)) < 0) {
	   cdb_close(s);
	   return -1;
       }

       if (cdb_start_session(s, CDB_RUNNING) != CONFD_OK){
	   cdb_close(s);
	   return -1;
       }

       if (cdb_get_u_int32(s, &n, "/table/parameter[0]/value") != CONFD_OK)
	   return -1;

       printf("Response: %d\n", n);

       cdb_end_session(s);
       cdb_close(s);

       return 0;
   }


To use Clixon instead, the following needs to change but
most of the functions are similar:

- confd_init        -> clixon_client_init
- cdb_connect       -> clixon_client_connect
- cdb_start_session -> Not needed for Clixon
- cdb_get_u_int32   -> clixon_client_get_uint32
- cdb_end_session   -> clixon_client_disconnect
- cdb_close         -> clixon_client_terminate


The above example re-written for Clixon looks as follows:

::

   #include <unistd.h>
   #include <stdio.h>
   #include <stdint.h>

   #include <clixon/clixon_client.h>

   int main(int argc, char **argv)
   {
       clixon_handle h = NULL;
       clixon_client_handle ch = NULL;

       uint32_t n = 0;

       if ((h = clixon_client_init("/usr/local/etc/example.xml")) == NULL)
	   return -1;

       if ((ch = clixon_client_connect(h, CLIXON_CLIENT_NETCONF)) == NULL)
	   return -1;

       if (clixon_client_get_uint32(ch, &n, "urn:example:clixon", "/table/parameter[name='a']/value") < 0)
	   return -1;

       printf("Response: %n\n", u);

       clixon_client_disconnect(ch);
       clixon_client_terminate(h);

       return 0;
   }

Tne difference between Clixon and ConfD is that Clixon data paths use full XPATHs::

   /table/parameter[name='a']/value

One can make the same index access as in ConfD paths as well (eg
`[0]`). This means that one can make direct indexed accesses as an alternative to looping.

.. include:: ./client_examples.rst
