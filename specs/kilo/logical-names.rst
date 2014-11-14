..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Ironic Logical Names
====================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/ironic/+spec/logical-names

Everything that is tracked in Ironic is done via UUID.  This isn't very
friendly to humans.  We should support nodes being referenced by a logical
name, in addition to the current method of being able to refer to them via
UUID.  This should be supported in the REST API and in our command-line client.


Problem description
===================

Operators, and other humans, that use Ironic find it awkward and error-prone
to use UUIDs to refer to entities that are tracked by Ironic.

However computers, and extremely geekly people, prefer to track things by the
canonical identifier - the UUID.

While humans are more likely to use the command-line tools, computers
are more likely to use the REST API.  However the opposite is also true,
consequently both the API and command-line client require updating to
support logical names for nodes.

It's useful to be able to assign semantic meaning to nodes - for example
datacentre location (i.e. 'DC6'), or node function (i.e. 'database') - to
assist operators in managing nodes in their network. The semantic identifier
of a node is analogous to the hostname for the node, and may indeed correlate.

An example of this association might be the logical name 'DC6-db-17' is
associated with a node with UUID '9e592cbe-e492-4e4f-bf8f-4c9e0ad1868f'.  In
all interactions with ironic, the node's UUID or logical name can be used to
identify the specific node.

Proposed change
===============

We propose adding a new concept to ironic, that being the <logical name>,
which can be used interchangeably with the <node uuid>.  Everywhere a
<node uuid> can be specified, we should be able to instead specify a
<logical name>, if such an association exists for that node.

There should be a 1:1 mapping between a <logical name> and a <node uuid>.  We
can consider a <logical name> as an alias for a <node uuid>.

To support this, new APIs need to be provided to:
1) Create a <logical name> and associate it with a <node uuid>,
2) List all the <logical name> to <node uuid> associations,
3) Modify the <logical name> associated with a <node uuid> (which has the
side-effect of deleting the previous <logical name>), and
4) Delete the association between a <logical name> and a <node uuid> (which
also has the side effect of deleting the <logical name>).

This needs to be supported at the REST API and at the python-ironicclient.


In addition, we should at this time also provide an `ironic.bash_completion`
drop-in script for the BASH shell that allows tab completion in specifying
nodes.  Both the current UUIDs, and the proposed logical names, should be
supported.


Alternatives
------------
Some benefit from just implementing bash completion for nodes specified by
UUID is gained, however this is insufficient as it would not allow semantic
tagging of nodes to ease operators.


Data model impact
-----------------

What is a logical name?
~~~~~~~~~~~~~~~~~~~~~~~
This change introduces the concept of a <logical name> to ironic so that a
human readable name can be associated with nodes.  This <logical name> should
be hostname safe, that is, the node logical name should also be usable as the
hostname for the instance.  For this to be true, the following references
should be used to define what is a valid <logical name>: [wikipedia:hostname],
[RFC952] and [RFC1123].

In simple english, what this means is that <logical names>s can be between
1 and 63 characters long, with the valid characters being [a-z0-9] and '-',
except that a <logical name> cannot begin or end with a '-'.

As a regular expression, this can be represented as:
<logical name> == [a-z0-9]([a-z0-9\-]{0,61}[a-z0-9]|[a-z0-9]{0,62})?


Where do we store the association?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The association between logical-name and UUID will need to be stored in
a table in the database.  Both logical-names, and UUIDs, will need to be
protected via a UNIQUE constraint.

Logical names and UUIDs will only be added to this new table once the
association between them is made.  There will be no prep-population of UUIDs
to the table.

As this is a new table, no database migration is required.


REST API impact
---------------
To implement logical names a number of changes are required.  These are:

New APIs
~~~~~~~~

* POST /v1/nodes/associate

  * This is a new API which will associate a <logical_name> with a <node uuid>.

  * URL parameters: None

  * JSON body: {"logical_name": "node_uuid"}

  * Return Codes:

    * If the call is successful, HTTP 200 ('OK') should be returned.

    * If <node uuid> is not found, HTTP 404 ('Not Found') should be returned.

    * If <logical_name> is not valid, according to the validation rules in
      this spec, HTTP 406 ('Not Acceptable') should be returned.

    * If <logical_name> is not unique, or the <node uuid> already has an
      <logical name> associated with it, HTTP 409 ('Conflict') should be
      returned.

  * Response Body: None

* PUT /v1/nodes/associate

  * This is a new API which will modify the <logical_name> associated with a
    <node UUID>.

  * URL parameters: None

  * JSON body: {"logical_name": "node_uuid"}

  * Return Codes:

    * If the call is successful, HTTP 200 ('OK') should be returned.

    * If <node uuid> is not found, HTTP 404 ('Not Found') should be returned.

    * If <logical_name> is not valid, according to the validation rules in
      this spec, HTTP 406 ('Not Acceptable') should be returned.

    * If <logical_name> is not unique, HTTP 409 ('Conflict') should be
      returned.

  * Response Body: None

* GET /v1/nodes/associate

  * This is a new API which will return all the <logical_name> to <node UUID>
    associations.

  * URL parameters: None

  * JSON body: None

  * Return Codes:

    * If the call is successful, HTTP 200 ('OK') should be returned.

    * If there is a problem retrieving the list of associations, an HTTP 400
      ('Bad Request') will be returned

  * Response Body: None

* DELETE  /v1/nodes/associate

  * This is a new API which will delete the association between a
    <logical_name> and a <node UUID>.  The <logical_name> will no longer
    have any meaning in ironic.

  * URL parameters: None

  * JSON body: {"logical_name": "name"} or {"node_uuid": "uuid"}
    Note: only one or the other is allowed to be specified.

  * Return Codes:

    * If the call is successful, HTTP 200 ('OK') should be returned.

    * If <node uuid> or <logical name> is not found, HTTP 404 ('Not Found')
      should be returned.

    * If both <logical_name> and <node_uuid> are specified, HTTP 409
      ('Conflict') should be

  * Response Body: None


Existing APIs
~~~~~~~~~~~~~
In addition to these new APIs, a number of existing APIs will need to be
modified to support the use of <logical_name>.

The following APIs will add in a new JSON body parameter named "logical_name":
* DELETE /v1/nodes
* PATCH /v1/nodes
* GET /v1/nodes/validate

The following APIs will add in a new response body field named "logical_name":
* GET /v1/nodes

The following new APIs will reflect the existing node_uuid version of this
API except that they will specify logical_name instead of node_uuid in the URL:
* GET /v1/nodes/(node_logical_name)
* PUT /v1/nodes/(node_logical_name)/maintenance
* DELETE /v1/nodes/(node_logical_name)/maintenance
* GET /v1/nodes/(node_logical_name)/management/boot_device
* PUT /v1/nodes/(node_logical_name)/management/boot_device
* GET /v1/nodes/(node_logical_name)/management/boot_device/supported
* GET /v1/nodes/(node_logical_name)/states
* PUT /v1/nodes/(node_logical_name)/states/power
* PUT /v1/nodes/(node_logical_name)/states/provision
* GET /v1/nodes/(node_logical_name)/states/console
* PUT /v1/nodes/(node_logical_name)/states/console
* POST /v1/nodes/(node_logical_name)/vendor_passthru

RPC API impact
--------------
None

Driver API impact
-----------------
None

Nova driver impact
------------------
This change as specified here is wholely contained with ironic itself.  It is
most probably beneficial to expose the concept of a logical name to outside
ironic for use in the Nova API.

If required, this will be addressed in an independent spec.

Security impact
---------------
None

Other end user impact
---------------------
If Horizon allows a user to enter a node UUID, and validates it as conforming
to a particular regex, then this will most likely require change to support
either a <node uuid> or <logical name>.

python-ironicclient
~~~~~~~~~~~~~~~~~~~
In each sub-command in python-ironicclient where a node UIUD can be specified,
we will need to be able to support a logical name in its place.  Please see
the detailed changes in the REST API section for an idea of the scope of
change required.

Additionally a number of new sub-commands to python-ironicclient will be
required to manage logical names, specifically:

1. ironic add-association <logical-name> <node UUID>

This command associates <node UUID> with the unique <logical name>.

If <node UUID> is already associated with a logical name, an error will be
returned alongside no change to the node's data.

If <logical name> is already in use, an error will be returned alongside no
change to the node's data.

2. ironic list-associations

This command lists all associations between <node UUID>s and <logical-name>s.

3. ironic delete-association <logical-name>|<node UUID>

This command deletes the association belonging to the node specified by the
supplied <logical name> or <node UUID>.  If the supplied <logical-name> or
<node UUID> has no association, then an error will be returned.

bash command-line completion
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
To support command-line logical name tab completion, a `bash_completion`
option to python-ironicclient to provide a list of valid nodes.

Scalability impact
------------------
None

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  mrda - Michael Davies <michael@the-davies.net>

Work Items
----------
1. New database table for logical_name and node uuid association
2. REST API additions and modifications
3. python-ironicclient additions and modifications
4. bash completion addition

Dependencies
============
None

Testing
=======
Unit testing will be sufficient to verify the veracity of this change

Upgrades and Backwards Compatibility
====================================
None

Documentation Impact
====================
Online documentation for both the Ironic API and python-ironicclient will need
to be updated to accompany this change.

References
==========
The need for this change was discussed at the Kilo Summit in Paris
(ref https://etherpad.openstack.org/p/kilo-ironic-making-it-simple)

* [wikipedia:hostname] - http://en.wikipedia.org/wiki/Hostname

* [RFC952] - http://tools.ietf.org/html/rfc952

* [RFC1123] - http://tools.ietf.org/html/rfc1123

