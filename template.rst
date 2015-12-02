..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
 The title of your blueprint
=============================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/ironic-inspector/+spec/example

Introduction paragraph -- why are we doing anything?

Some notes about using this template:

* Your spec should be in ReSTructured text, like this template.

* Please wrap text at 79 columns.

* The filename in the git repository must match the launchpad URL, for
  example a URL of:
  https://blueprints.launchpad.net/ironic-inspector/+spec/awesome-thing
  must be named awesome-thing.rst

* Please do not delete any of the sections in this template.  If you have
  nothing to say for a whole section, just write: None

* For help with syntax, see http://sphinx-doc.org/rest.html

* To test out your formatting, build the docs using tox, or see:
  http://rst.ninjs.org

* If you would like to provide a diagram with your spec, ascii diagrams are
  required.  http://asciiflow.com/ is a very nice tool to assist with making
  ascii diagrams.  The reason for this is that the tool used to review specs is
  based purely on plain text.  Plain text will allow review to proceed without
  having to look at additional files which can not be viewed in gerrit.  It
  will also allow inline feedback on the diagram itself.

Problem description
===================

A detailed description of the problem:

* For a new feature this might be use cases. Ensure you are clear about the
  actors in each use case: End User, Admin User, Deployer, or another Service

* For a major reworking of something existing it would describe the
  problems in that feature that are being addressed.


Proposed change
===============

Here is where you cover the change you propose to make in detail. How do you
propose to solve this problem?

If this is one part of a larger effort make it clear where this piece ends. In
other words, what's the scope of this effort?

Include where in the ironic-inspector tree hierarchy this will reside.

Alternatives
------------

This is an optional section, where it does apply we'd just like a demonstration
that some thought has been put into why the proposed approach is the best one.

Data model impact
-----------------

Changes which require modifications to the data model often have a wider impact
on the system.  The community often has strong opinions on how the data model
should be evolved, from both a functional and performance perspective. It is
therefore important to capture and gain agreement as early as possible on any
proposed changes to the data model.

Questions which need to be addressed by this section include:

* What new data objects and/or database schema changes is this going to
  require?

* What database migrations will accompany this change?

* How will the initial set of new data objects be generated? For example, if
  you need to take into account existing instances, or modify other existing
  data, describe how that will work.

HTTP API impact
---------------

Describe changes to the HTTP API.

Each API method which is either added or changed should have the following

* Specification for the method

  * A description of what the method does, suitable for use in user
    documentation.

  * Method type (POST/PUT/GET/DELETE/PATCH)

  * Normal http response code(s)

  * Expected error http response code(s)

    * A description for each possible error code should be included.
      Describe semantic errors which can cause it, such as
      inconsistent parameters supplied to the method, or when a
      resource is not in an appropriate state for the request to
      succeed. Errors caused by syntactic problems covered by the JSON
      schema definition do not need to be included.

  * URL for the resource

  * Parameters which can be passed via the url, including data types

  * JSON schema definition for the body data if allowed

  * JSON schema definition for the response data if any

* Does the API microversion need to increment?

* Example use case including typical API samples for both data supplied
  by the caller and the response

* Is this change discoverable by clients? Not all clients will upgrade at the
  same time, so this change must work with older clients without breaking them.

Note that the schema should be defined as restrictively as possible. Parameters
which are required should be marked as such and only under exceptional
circumstances should additional parameters which are not defined in the schema
be permitted.

Reuse of existing predefined parameter types is highly encouraged.

Client (CLI) impact
-------------------

Typically, but not always, if there are any REST API changes, there are
corresponding changes to python-ironic-inspector-client. If so, what does
the user interface look like. If not, describe why there are REST API changes
but no changes to the client.

Performance and scalability impact
----------------------------------

Describe any potential performance impact on the system, for example
how often will new code be called, and is there a major change to the calling
pattern of existing code.

Describe any potential scalability impact on the system, for example any
increase in network, RPC, or database traffic, or whether the feature
requires synchronization across multiple services.

Security impact
---------------

Describe any potential security impact on the system.

Deployer impact
---------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example, a flag that other hardware drivers might want to
  implement as well)? Are the default values appropriate for production?
  Provide an explanation of why these defaults are reasonable.

* Is this a change that takes immediate effect after it's merged, or is it
  something that has to be explicitly enabled?

* If this change adds a new service that deployers will be required to run,
  how would it be deployed? Describe the expected topology, for example,
  what network connectivity the new service would need, what service(s) it
  would interact with, how many should run relative to the size of the
  deployment, and so on.

* Please state anything that those doing continuous deployment, or those
  upgrading from the previous release, need to be aware of. Also describe
  any plans to deprecate configuration values or features.

* If your proposal includes any changes to the REST API, describe how existing
  clients will continue to function when interacting with an upgraded API
  server.

* Describe what testing you will be adding to ensure that backwards
  compatibility is maintained.

* If deprecating an existing feature or API, describe the deprecation plan, and
  for how long compatibility will be maintained.

Developer impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the hooks API, discussion of how
  other hooks would implement the feature is required. Describe how
  existing hooks will continue to function after the change.


Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  <launchpad-id or None>

Can optionally can list additional ids if they intend on doing
substantial implementation work on this blueprint.

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

- Include specific references to specs and/or blueprints in ironic-inspector,
  or in other projects, that this one either depends on or is related to.

- Does this feature require any new library dependencies or code otherwise not
  included in OpenStack? Or does it depend on a specific version of library?


Testing
=======

Please discuss how the change will be tested. We especially want to know what
tempest tests will be added. It is assumed that unit test coverage will be
added so that doesn't need to be mentioned explicitly, but discussion of why
you think unit tests are sufficient and we don't need to add more tempest
tests would need to be included.


References
==========

Please add any useful references here. You are not required to have any
reference. Moreover, this specification should still make sense when your
references are unavailable. Examples of what you could include are:

* Links to mailing list or IRC discussions

* Links to notes from a summit session

* Links to relevant research, if appropriate

* Related specifications as appropriate (e.g.  if it's an EC2 thing, link the
  EC2 docs)

* Anything else you feel it is worthwhile to refer to
