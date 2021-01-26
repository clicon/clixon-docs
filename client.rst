.. _client_api:

Client API
==========


Clixons client API offers a simple way to communicate with the
built-in XML datastore. This can be used to fetch or manipulate
configuration which is handled by Clixon from any other application
running on the host system.


For example, there can be a deamon running on the host system and we
want that deamon to read a certain configured value from the
datastore. Normally we would use the plugin API, but there is also the
option to use the client API to achieve that in a fast and simple way.

Below we will use examples from both Clixon and Cisco ConfD and see
how we can achieve the same thing using the two of them, and also show
how Clixon can be a replacement for ConfD with minimal effort.



Comparison between Clixon and ConfD
-----------------------------------

Cisco ConfD is another well known configuration manager which also
maintains the configuration in its buil-in datastore. Just like
Clixon, ConfD offers an API for configuration access and manipulation.

Applications that are integrated using ConfDs CDB API can easly be
converted to use Clixon instead. Below we have a minimal example
application which will connect to ConfD and get a value which is
stored under "/table/parameter" in the XML store.

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


To use Clixon instead, there are very few things we have to change and
most of the functions are nearly identical, if we want to convert the
above application to use Clixon instead, we will change very few things:

- confd_init        -> clixon_client_init
- cdb_connect       -> clixon_client_connect
- cdb_start_session -> Not needed for Clixon
- cdb_get_u_int32   -> clixon_client_get_uint32
- cdb_end_session   -> clixon_client_disconnect
- cdb_close         -> clixon_client_terminate


And the above example re-written for Clixon would look like this:

::

   #include <unistd.h>
   #include <stdio.h>
   #include <stdint.h>

   #include <clixon/clixon_log.h>
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

One difference between Clixon and ConfD is that when we handle data
Clixon will use an XPATH:

::

   "/table/parameter[name='a']/value"

This means that we don't have to know where in a list to find a
specific item, the XPATH can look that up for us and we can avoid
writing code for iterating the list of possible parameter values.
