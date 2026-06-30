.. _clixon_migration:
.. sectnum::
   :start: 21
   :depth: 3

*********************
C API Migration Guide
*********************

This section documents breaking or deprecated C API changes in Clixon, organized
by release. Each entry describes what changed, why, and how to update application
code that uses Clixon as a library.

.. _migration_790:

7.9.0
=====

XML child iteration: ``xml_child_each()`` → ``xml_child_iter()``
----------------------------------------------------------------
The ``xml_child_each()`` iterator has been replaced by ``xml_child_iter()``,
which uses an integer index as loop state instead of a child pointer. This
enables internal optimizations to the child storage layout.

Change the loop variable declaration and iterator call as follows:

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - Old (``xml_child_each``)
     - New (``xml_child_iter``)
   * - .. code-block:: c

          cxobj *xc;
          xc = NULL;
          while ((xc = xml_child_each(xt, xc, CX_ELMNT))
                 != NULL) {
              /* ... */
          }

     - .. code-block:: c

          cxobj *xc;
          int    ix = 0;
          while ((xc = xml_child_iter(xt, &ix, CX_ELMNT))
                 != NULL) {
              /* ... */
          }

The key differences are:

* Replace the ``xc = NULL`` initializer with ``int ix = 0``
* Change ``xml_child_each(xt, xc, type)`` to ``xml_child_iter(xt, &ix, type)``
* The ``type`` argument (e.g. ``CX_ELMNT``, ``CX_ATTR``, ``CX_BODY``, or ``-1`` for all) is unchanged

After migrating, configure with ``--enable-xml-child-each-wrapper`` to catch any
remaining uses of the old API via a build-time deprecation wrapper.

XML body API: direct ``CX_BODY`` manipulation → ``xml_body_set/append/reset()``
--------------------------------------------------------------------------------
Three new functions replace the pattern of manually creating and manipulating
``CX_BODY`` child nodes to set the text content of an element:

- ``xml_body_set(xn, val)``    — set (replace) the body value on element ``xn``
- ``xml_body_append(xn, val)`` — append to the body value (creates body if needed)
- ``xml_body_reset(xn)``       — remove the body value from ``xn``

The read functions ``xml_body(xn)`` and ``xml_body_get(xn)`` are unchanged.

Replace direct ``CX_BODY`` patterns as follows:

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - Old pattern
     - New pattern
   * - .. code-block:: c

          cxobj *xb;
          xb = xml_new("body", xn, CX_BODY);
          xml_value_set(xb, val);

     - .. code-block:: c

          xml_body_set(xn, val);

   * - .. code-block:: c

          cxobj *xb;
          xb = xml_body_get(xn);
          xml_value_set(xb, val);

     - .. code-block:: c

          xml_body_set(xn, val);

   * - .. code-block:: c

          cxobj *xb;
          xb = xml_new("body", xn, CX_BODY);
          xml_value_append(xb, val);

     - .. code-block:: c

          xml_body_append(xn, val);

   * - .. code-block:: c

          xml_rm_children(xn, CX_BODY);

     - .. code-block:: c

          xml_body_reset(xn);

.. note::
   ``xml_value_set()`` and ``xml_value_append()`` remain valid for
   **attribute** (``CX_ATTR``) nodes and should not be replaced there.

The ``xml_new_body("name", parent, "val")`` helper function is unchanged and
continues to work correctly.

In addition to the write patterns above, a common **read** pattern must also
be migrated. Accessing the body via ``xml_find_value(x, "body")`` or
``xml_find(x, "body")`` relies on the internal ``CX_BODY`` child node being
named ``"body"``. In ``OPTMEM_XML_BODY`` mode that child no longer exists and
both calls return ``NULL``.  Replace with ``xml_body()``:

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - Old pattern
     - New pattern
   * - .. code-block:: c

          char *val;
          val = xml_find_value(x, "body");

     - .. code-block:: c

          char *val;
          val = xml_body(x);

   * - .. code-block:: c

          cxobj *xb;
          xb = xml_find(x, "body");
          if (xb)
              val = xml_value(xb);

     - .. code-block:: c

          char *val;
          val = xml_body(x);

.. note::
   Checking ``xml_child_nr(x) > 0`` as a proxy for "element has body text"
   is also broken in ``OPTMEM_XML_BODY`` mode.  Use ``xml_body(x) != NULL``
   instead.

   Similarly, checking ``xml_child_nr(x) == 1`` to detect a leaf element
   (expecting the single child to be a ``CX_BODY`` node) no longer works.
   To test whether an element is a leaf (text content, no child elements),
   use ``xml_child_nr_type(x, CX_ELMNT) == 0`` combined with
   ``xml_body(x) != NULL``.

After migrating, configure with ``--enable-optmem-xml-body`` to switch over to replacing all XML body objects with folding their value into the parent element.
