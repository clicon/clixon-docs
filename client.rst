.. _client_api:
.. sectnum::
   :start: 16
   :depth: 3

**********
Client API
**********

.. note::
   This section is not complete and outdated. A new Client API is developed as part of the `Clixon controller <https://clixon-controller-docs.readthedocs.io>`_.

Clixon's client API provides a way to communicate with the built-in
XML datastore. This can be used to fetch or manipulate configurations
handled by Clixon from any other application running on a host
system. For example, a deamon running on a host system may need to
read a configured value from the Clixon datastore.

Clixon integration normally uses dynamic plugins, but the client API
shown here is an alternative.

Below is a minimal example application which connects to Clixon to get
a value stored under "/table/parameter" in the XML store::

   #include <unistd.h>
   #include <stdio.h>
   #include <stdint.h>

   #include <clixon/clixon_client.h>

   int main(int argc, char **argv)
   {
       clixon_handle h = NULL;
       clixon_client_handle ch = NULL;

       uint32_t n = 0;

       if ((h = clixon_client_init("/usr/local/etc/clixon/example.xml")) == NULL)
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

Clixon data paths use full XPATHs::

   /table/parameter[name='a']/value

One can make the same index access as well (eg `[0]`). This means that one can make direct indexed accesses as an alternative to looping.

.. include:: ./client_examples.rst
