# The Stripes Connect API

<!-- ../okapi/doc/md2toc -l 2 api.md -->
* [Introduction](#introduction)
    * [Note](#note)
* [The Connection Manifest](#the-connection-manifest)
    * [Resource types](#resource-types)
        * [Local resources](#local-resources)
        * [REST resources](#rest-resources)
        * [Okapi resources](#okapi-resources)
    * [Example](#example)
* [Connecting the component](#connecting-the-component)
* [Error handling](#error-handling)
* [Using the connected component](#using-the-connected-component)
* [Appendices: for developers](#appendices-for-developers)
    * [Appendix A: how state is stored](#appendix-a-how-state-is-stored)
    * [Appendix B: unresolved issues](#appendix-b-unresolved-issues)
        * [One vs. Many](#one-vs-many)
        * [Metadata](#metadata)
        * [Object counts](#object-counts)
    * [Appendix C: worked-through example of connected component](#appendix-c-worked-through-example-of-connected-component)
    * [Appendix D: walk-through of state-object changes during a CRUD cycle](#appendix-d-walk-through-of-state-object-changes-during-a-crud-cycle)



## Introduction

Stripes Connect is one of the most important parts of the Stripes
toolkit for building FOLIO UIs. It provides the connection between the
UI and the underlying services -- most usually, Okapi (the FOLIO
middleware), though other RESTful web services are also supported.

A Stripes UI is composed of
[React](https://facebook.github.io/react/)
components. (You will need to learn at least the basics of React in
order to use Stripes.) Any component may use the services of Stripes
Connect to automate communication with back-end services. Such a
component is known as a "connected component".

In order to take advantage of Stripes Connect, a component must do two
things: declare a _manifest_, which describes what data elements it
wants to manage and how to link them to services; and call the
`connect()` method on itself.


### Note

This document describes the API as we wish it to be. The present
version of the code implements something similar to this, but not
identical. In what follows, additional notes mark such divergences.



## The Connection Manifest

The manifest is provided as a static member of the component class. It
is a JavaScript object in which the keys are the names of resources to
be managed, and the corresponding values are objects specifying how to
deal with them:

        static manifest = {
          'bibs': { /* ... */ },
          'items': { /* ... */ },
          'patrons': { /* ... */ }
        };


### Resource types

Each resource has several keys. The most important of these is `type`,
which determines how the associated data is treated. Currently, three
types are supported:

* `local`: a local resource (client-side only), which is not persisted
  by means of a service.
* `okapi`: a resource that is persisted by means of a FOLIO service
  mediated by Okapi.
* `rest`: a resource persisted by some RESTful service other than
  Okapi.

(In fact, the `okapi` type is merely a special case of `rest`, in
which defaults are provided to tailor the RESTful dialogues in
accordance with Okapi's conventions.)


#### Local resources

A local resource needs no configuration items -- not even an explicit
`type`, since the default type is `local`. So its configuration can
simply be specified as an empty object:

        static manifest = {
          'someLocalResource': {}
        }


#### REST resources

REST resources are configured by the following additional keys in
addition to `'type':'rest'`:

* `root`: the base URL of the service that persists the data.

* `path`: the path for this resource below the specified root. The
  path consists of one or more `/`-separated components: each
  component may be a literal, or a placeholder of the form `:`_name_,
  which is replaced at run-time by the value of the named property.

* `params`: A JavaScript object containing named parameters to be
  supplied as part of the URL. These are joined with `&` and appended
  to the path with a `?`.
  The root, path and params together make up the URL that is
  addressed to maintain the resource.

* `headers`: A JavaScript object containing HTTP headers: the keys are
  the header names and the values are their content.

* `records`: The name of the key in the returned JSON that contains
  the records. Typically the JSON response from a web service is not
  itself an array of records, but an object containing metadata about
  the result (result-count, etc.) and a sub-array that contains the
  actual records. The `records` item specifies the name of that
  sub-array within the top-level response object.

* `pk`: The name of the key in the returned records that contains
  the primary key. (Defaults to `id` for both REST and Okapi
  resources.)

* `clientGeneratePk`: a boolean indicating whether the client must
  generate a "sufficiently unique" private key for newly created
  records, or must accept one that is supplied by the service in
  response to a create request. Default: `true`.

* `fetch`: a component that adds a new record to an end-point would
  usually not need to pre-fetch from that resource. To avoid that, it
  can set this to false. Default: `true`.

In addition to these principal pieces of configuration, which apply to
all operations on the resource, these values can be overridden for
specific HTTP operations: the entries `GET`, `POST`, `PUT`, `DELETE`
and `PATCH`, if supplied, are objects containing configuration (using
the same keys as described above) that apply only when the specified
operation is used.

Similarly, the same keys provided in `staticFallback` will be used when
dynamic portions of the config are not satisfied by the current state.


#### Okapi resources

Okapi resources are REST resources, but with defaults set to make
connecting to Okapi convenient:

* `root`: defaults to a globally-configured address pointing to an
  Okapi instance.

* `headers`: are set appropriately for each HTTP operation to send the
  tenant-ID, specify that the POSTed or PUT body is JSON and expect
  JSON in response.


### Example

This manifest (from the Okapi Console component that displays the
health of running modules) defines two Okapi resources, `health` and
`modules`, providing paths for both of them that are interpreted
relative to the default root. In the modules response, the primary key
is the default, `id`; but in the health response, it is `srvcId`, and
the manifest must specify this.

        static manifest = {
          'health': {
            type: 'okapi',
            pk:   'srvcId',
            path: '_/discovery/health'
          },
          'modules': {
            type: 'okapi',
            path: '_/proxy/modules'
          }
        };



## Connecting the component

React components are classes that extend `React.Component`.  Instead
of using a React-component class directly -- most often by exporting
it -- use the result of passing it to the `connect()` method of
Stripes Connect.

For example, rather than

        export class Widget extends React.Component {
          // ...
        }

or

        class Widget extends React.Component {
          // ...
        }
        export Widget;

use

        import { connect } from 'stripes-connect';
        class Widget extends React.Component {
          // ...
        }
        export connect(Widget, 'stripes-module-name');

(At present, it is necessary to pass as a second argument the name of
the Stripes module that contains the connect component. We hope to
remove this requirement in future.)



## Error handling

By default, errors in communicating with a REST service such as Okapi
are signalled to the user in an alert-box that reports messages
such as:

> ERROR: in module 'okapi-console', operation 'CREATE' on resource
> 'modules' failed, saying: module
> 30c0046e-0967-4d3c-80fb-98c7c3ef2bb3: Missing dependency:
> 30c0046e-0967-4d3c-80fb-98c7c3ef2bb3 requires patrons: 1.1

This is on the basis that it's better to announce errors loudly than
to allow them to pass silently, But applications will often want to
handle errors in a more sophisticated way. This can be done by
installing an error-handler function in a component by providing it as
part of the component's manifest under the special key
`@errorHandler`.

The handler object is passed an error object which has the following
elements (all of type string):

* **module** -- the name of the Stripes module in which the error
  occurred.
* **op** -- the operation that Stripes Connect was trying to carry out
  **on behalf of the module: usually one of
  `FETCH`, `UPDATE`, `CREATE` or `DELETE`.
* **resource** -- the name of the resource within the component's
  manifest that Stripes Connect was trying to handle when the error
  occurred.
* **error** -- a description of what went wrong.

For example one might define an error-handler that logs the relevant
information to the JavaScript console, and install it in a component,
as follows:

        static handler(e) {
          console.log("TenantList ERROR: in module '" + e.module + "', " +
                      " operation '" + e.op + "' on " +
                      " resource '" + e.resource + "' failed, saying: " + e.error);
        }

        static manifest = {
          '@errorHandler': TenantList.handler,
          'tenants': {
            path: '_/proxy/tenants',
            type: 'okapi'
          }
        };



## Using the connected component

When a connected component is invoked, two properties are passed to
the wrapped component:

* `data`: contains the data associated with the resources in the
  manifest, as a JavaScript object whose keys are the names of
  resources. This is null if the data is pending and has not yet been
  fetched.

* `mutator`: a JavaScript object that enables the component to make
  changes to its resources. `mutator` is an object whose properties
  are named after the resources in the manifest. The corresponding
  values are themselves objects whose keys are HTTP methods and whose
  values are methods that perform the relevant CRUD operation using
  HTTP and update the internal representation of the state to match. A
  typical invocation would be something like
  `this.props.mutator['users'].POST(data)`.

The mutator methods optionally take a record as a parameter,
represented as a JavaScript object whose keys are fieldnames and whose
values contain the corresponding data. These records are used in the
obvious way by the POST, PUT and PATCH operations. For DELETE, the
record need only contain the `id` field, so that it suffices to call
`mutator.tenants.DELETE({ id: 43 })`.

<br/>
<br/>
<hr/>



## Appendices: for developers

These sections are only for developers working on Stripes
itself. Those working on _using_ Stripes to build a UI can ignore them.


### Appendix A: how state is stored

* All state is stored in a single branching structure, the _Redux
  store_. (Module creators should not need to know about details of
  Redux, and especially not about reducers, but this idea of a single
  state store is important nevertheless.)

* Data in this state structure consists of _resources_, each named by
  a string.

* Rather than each module having its own namespace within the
  structure, all modules' data is kept together in a single big
  table.

* To avoid different modules' same-named data from clashing, the code
  arranges that the keys in this table are composed of the module name
  and the resource name separated by an underscore:
  _moduleName_`_`_resourceName_

  * XXX In fact, the code that does this is the `stateKey()` methods of
    the various resource-types. That means (A) we need to be very
    careful that new resource-types also remember to do this; and (B)
    we probably made a mistake, and this should instead by done at a
    higher level in `stripes-connect/connect.js`.

* A component's resource names are defined by the keys in its _data
  manifest_. The value associated with each key is tied to the
  resource specified by its parameters -- for example, the `root` and
  `path` of a REST resource. In general, that value is a list of
  records: some components will deal only with a single record from
  that list.

  * XXX For example, `PatronEdit.js` deals only with a single record;
    but it works with the `patrons` resource, which is a list of
    records, and picks out the one it wants by using
    `patrons.find((p) =>  { return p._id === patronid })`.
    If I have understood this correctly, it looks like a grotesque
    inefficiency that will quickly become unworkable as we start to
    use large patron databases.

* In general, a Stripes module contains multiple connected
  components. The data manifest is specific per-component. Components
  may communicate with each other, or share cached data, by using the
  same data keys. It is the module author's responsibility to avoid
  inadvertent duplication of keys between unrelated components.

* Some components may exist in multiple simultaneous instances: for
  example, a list-of-records component may be designed such that a
  user may pop up a more than one full-record component to see the
  details of several records at once. In this case, the state keys are
  different because the records' IDs are included (due to the manifest
  path being of the form `/patrons/:patronid`, containing a
  placeholder.)

* For local resources, which are not persisted via a REST service such
  as Okapi, some means must be established whereby each individual
  datum is individually addressable. Only then can multiple instances
  of the same component that uses local storage co-exist.

  * For this reason, it may be worthwhile to prioritise the
    development of a page that has two instances of the Trivial
    module, and see that they can each maintain their own data.

  * Also to be done: a simple implementation of search preferences, as
    a model for how two components (SearchForm and SearchPrefernces)
    can deliberately share data.


### Appendix B: unresolved issues

#### One vs. Many

Right now there is no clear standard as to what data is returned by
Okapi services -- for example, a single record by `id` yields a single
record; but a query that matches a single record yields an array with
one element.

#### Metadata

We use the Redux Crud library to, among other things, generate
actions. It causes a number of compromises such as our needing to
clear out the Redux state at times, because it is designed for a
slightly different universe where there is more data re-use.

As part of that, it prefers to treat the responses as lists of records
that it can merge into its local store which makes having a top level
of metadata with an array property `patrons` or similar a bit
incompatible.

We currently pass it a key from the manifest (`records`) if it needs
to dig deeper to find the records. But that means we just discard the
top level metadata. It may soon be time to reimplement Redux Crud
ourselves and take full control of how data is being shuffled
around. Among other things, it would give the option to let our API be
more true to intent and transparently expose the data returned by the
REST request.

#### Object counts

Can we get a count of holds on an item? How does that API work and
does our above system mesh with it well enough to provide a pleasant
developer experience?


### Appendix C: worked-through example of connected component

This is in the separate document
[A component example: the PatronEdit component](https://github.com/folio-org/stripes-core/blob/master/doc/component-example.md)


### Appendix D: walk-through of state-object changes during a CRUD cycle

XXX To be written

These images may be useful:
* https://files.slack.com/files-pri/T047C3PCD-F2L37S7C2/pasted_image_at_2016_10_06_11_13_am.png
* https://files.slack.com/files-pri/T047C3PCD-F2L2RAH5E/pasted_image_at_2016_10_06_11_15_am.png
* https://files.slack.com/files-pri/T047C3PCD-F2L2WSHA4/pasted_image_at_2016_10_06_11_31_am.png

