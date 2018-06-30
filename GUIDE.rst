Guide to the Duct Framework
===========================

Preface
~~~~~~~

Duct is a data-driven framework for writing server-side applications
in the Clojure_ programming language. This guide is intended as an
in-depth explanation of how to use Duct, and will focus on web
applications in particular.

This guide assumes you have a working knowledge of Clojure code, and
that you have Leiningen_ installed. A basic understanding of Ring_
will also be useful, but not required.

.. _Clojure:   https://clojure.org/
.. _Leiningen: https://leiningen.org/
.. _Ring:      https://github.com/ring-clojure/ring


Getting Started
~~~~~~~~~~~~~~~

The most straightforward way of getting started is to use the Duct
Leiningen template. Duct can be used to build many types of
server-side applications, but for the purposes of this guide we'll be
building a web service backed by a SQLite_ database.

Creating the Project
""""""""""""""""""""

At the shell, run::

  $ lein new duct todo +api +ataraxy +sqlite

This produces the output::

  Generating a new Duct project named todo...
  Run 'lein duct setup' in the project directory to create local config files.

The parameters prefixed by ``+`` are profile hints, which are used to
tell the template we want to build a web service (``+api``), using the
Ataraxy_ routing library (``+ataraxy``), against a SQLite database
(``+sqlite``).

If you want to see what profile hints there are available, you can
run::

  $ lein new :show duct

For now, let's change directory into the ``todo`` project that has
been created::

  $ cd todo

Then run the local setup::

  $ lein duct setup

This creates four files that should be kept out of source control::

  Created profiles.clj
  Created .dir-locals.el
  Created dev/resources/local.edn
  Created dev/src/local.clj

If you're using Git_, then these files are already added to your
``.gitignore`` file. If you're using another version control system,
then you'll need to manually update your ignore files.
  
.. _SQLite:  https://sqlite.org/
.. _Ataraxy: https://github.com/weavejester/ataraxy
.. _Git:     https://git-scm.com/


Starting the REPL
"""""""""""""""""

Duct development is orientated around the REPL. It's recommended that
you use an editor with REPL integration, such as Cursive_, Emacs_ with
CIDER_, Vim_ with `fireplace.vim`_, or Atom_ with `Proto REPL`_.
However, this guide doesn't require editor integration, and the
instructions will assume you're working directly from the command
line.

So start the REPL with::

  $ lein repl

At the prompt, we'll first load the development environment:

.. code-block:: clojure

  user=> (dev)
  :loaded
  dev=>

This isn't loaded automatically, as errors in the development could
cause the REPL not to start.

Once we're in the ``dev`` namespace we can start the application:

.. code-block:: clojure

  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

The web server has been started on port 3000. Lets check it's running
by sending it a HTTP request. This can be done from the command line
with the standard curl_ or wget_ tools, but I prefer HTTPie_ for
testing web services::

  $ http :3000
  HTTP/1.1 404 Not Found
  Content-Length: 21
  Content-Type: application/json; charset=utf-8
  Date: Wed, 06 Dec 2017 11:27:22 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "error": "not-found"
  }

We get a "not found" response, but this is expected as we've yet to
add any routes to the application.

.. _Cursive:       https://cursive-ide.com/
.. _Emacs:         https://www.gnu.org/software/emacs/
.. _CIDER:         https://github.com/clojure-emacs/cider
.. _Vim:           http://www.vim.org/
.. _fireplace.vim: https://github.com/tpope/vim-fireplace
.. _Atom:          https://atom.io/
.. _Proto Repl:    https://atom.io/packages/proto-repl
.. _curl:          https://curl.haxx.se/
.. _wget:          https://www.gnu.org/software/wget/
.. _HTTPie:        https://httpie.org/


Configuration
~~~~~~~~~~~~~

Duct applications are built around an edn_ configuration file. This
defines the structure and dependencies of the application. In the
project we're writing in this guide, the configuration is located at:
``resources/todo/config.edn``.


Adding a Static Route
"""""""""""""""""""""

Let's take a look at the configuration file:

.. code-block:: edn

  {:duct.core/project-ns  todo
   :duct.core/environment :production

   :duct.module/logging {}
   :duct.module.web/api {}
   :duct.module/sql {}

   :duct.module/ataraxy
   {}}

We're going to start by adding in a static index route, and to do that
we're going to add to the ``:duct.module/ataraxy`` key, since Ataraxy
is our router:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"] [:index]}

This connects a route ``[:get "/"]`` with a result ``[:index]``. The
Ataraxy module automatically looks for a Ring handler in the
configuration with a matching name to pair with the result. Since the
result key is ``:index``, the handler key is ``:todo.handler/index``.
Let's add in a configuration entry with that name:

.. code-block:: edn

  [:duct.handler.static/ok :todo.handler/index]
  {:body {:entries "/entries"}}

This time we're using a vector as the key; in Duct parlance, this is
known as a *composite key*. Composite keys inherit the properties of
all the keywords contained in them; because the vector contains the
key ``:duct.handler.static/ok``, the configuration entry produces a
static handler.

Let's apply this change to the application. Go to back to the REPL and
run:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

This reloads the configuration and any changed files. When we send a
HTTP request to the web server, we now get the expected response::

  $ http :3000
  HTTP/1.1 200 OK
  Content-Length: 22
  Content-Type: application/json; charset=utf-8
  Date: Wed, 06 Dec 2017 13:28:52 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "entries": "/entries"
  }

.. _edn: https://github.com/edn-format/edn

Adding a Database Migration
"""""""""""""""""""""""""""

We want to begin adding more dynamic routes, but before we can we need
to create our database schema. Duct uses Ragtime_ for migrations, and
each migration is defined in the configuration.

Add two more keys to the configuration:

.. code-block:: edn

  :duct.migrator/ragtime
  {:migrations [#ig/ref :todo.migration/create-entries]}

  [:duct.migrator.ragtime/sql :todo.migration/create-entries]
  {:up ["CREATE TABLE entries (id INTEGER PRIMARY KEY, content TEXT)"]
   :down ["DROP TABLE entries"]}

The ``:duct.migrator/ragtime`` key contains an ordered list of
migrations. Individual migrations can be defined by including
``:duct.migrator.ragtime/sql`` in a composite key. The ``:up`` and
``:down`` options contains vectors of SQL to execute; the former to
apply the migration, the latter to roll it back.

To apply the migration we run ``reset`` again at the REPL:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/applying :todo.migration/create-entries#b34248fc
  :resumed

Suppose after applying the migration we change our mind about the
schema. We could write another migration, but if we haven't committed
the code or deployed it to production it's often more convenient to
edit the migration we have.

Let's change the migration and rename the ``content`` column to
``description``:

.. code-block:: edn

  [:duct.migrator.ragtime/sql :todo.migration/create-entries]
  {:up ["CREATE TABLE entries (id INTEGER PRIMARY KEY, description TEXT)"]
   :down ["DROP TABLE entries"]}

Then ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/rolling-back :todo.migration/create-entries#b34248fc
  :duct.migrator.ragtime/applying :todo.migration/create-entries#5c2bb12a
  :resumed

The old version of the migration is automatically rolled back, and the
new version of the migration applied in its place.

.. _Ragtime: https://github.com/weavejester/ragtime

Running Database Migrations in Production
"""""""""""""""""""""""""""""""""""""""""

We can easily run migrations in production::

  $ lein run :duct/migrator

If you are using Heroku for deployment, this can easily be added to the release phase via your Procfile::

  web: java -jar target/sstandalone.jar
  release: lein run :duct/migrator

Adding a Query Route
""""""""""""""""""""

Now that we have a database table, it's time to write some routes to
query it. To do this, we're going to use a library called
``duct/handler.sql``, which should be added to the ``:dependencies``
key in your ``project.clj`` file:

.. code-block:: clojure

  [duct/handler.sql "0.3.1"]

Your dependencies should now look something like:

.. code-block:: clojure

  :dependencies [[org.clojure/clojure "1.9.0-RC1"]
                 [duct/core "0.6.1"]
                 [duct/handler.sql "0.3.1"]
                 [duct/module.logging "0.3.1"]
                 [duct/module.web "0.6.3"]
                 [duct/module.ataraxy "0.2.0"]
                 [duct/module.sql "0.4.2"]
                 [org.xerial/sqlite-jdbc "3.20.1"]]

Adding dependencies is one of the few times we have to restart the
REPL. So first we exit:

.. code-block:: clojure

  dev=> (exit)
  Bye for now!

Then we restart::

  $ lein repl

And start the application running again:

.. code-block:: clojure

  user=> (dev)
  :loaded
  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

We can now turn back to the project configuration. Let's start by
adding a new Ataraxy route:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]}

As before, the result ``[:entries/list]`` needs to be paired with an
appropriately named Ring handler. The Ataraxy module expects this
handler to be named ``:todo.handler.entries/list``, so we'll use that
name, along with the ``:duct.handler.sql/query`` key:

.. code-block:: edn

  [:duct.handler.sql/query :todo.handler.entries/list]
  {:sql ["SELECT * FROM entries"]}

Once the handler is defined in the configuration, we can ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

Then we check the route by sending a HTTP request to it::

  $ http :3000/entries
  HTTP/1.1 200 OK
  Content-Length: 2
  Content-Type: application/json; charset=utf-8
  Date: Thu, 07 Dec 2017 10:13:34 GMT
  Server: Jetty(9.2.21.v20170120)

  []

We get a valid, though empty response. This makes sense, as we've yet
to populate the ``entries`` table with any data.


Adding an Update Route
""""""""""""""""""""""

Next we'd like to add a route that updates the database. Again we're
going to be making use of the ``duct/handler.sql`` library, but both
the route and handler are going to be more complex.

First, the new route:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]}

The new Ataraxy route not only matches the method and URI of the
request, it also destructures the request body and places the
description of the todo entry into the result.

When we come to write the associated handler, we need some way of
getting the information from the result. Ataraxy places the result
into the ``:ataraxy/result`` key on the request map, so we can
destructure the request to find the description of the new entry:

.. code-block:: edn

  [:duct.handler.sql/insert :todo.handler.entries/create]
  {:request {[_ description] :ataraxy/result}
   :sql     ["INSERT INTO entries (description) VALUES (?)" description]}

Next we ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

And test::

  $ http post :3000/entries description="Write Duct guide"
  HTTP/1.1 201 Created
  Content-Length: 0
  Content-Type: application/octet-stream
  Date: Thu, 07 Dec 2017 11:29:46 GMT
  Server: Jetty(9.2.21.v20170120)


  $ http get :3000/entries
  HTTP/1.1 200 OK
  Content-Length: 43
  Content-Type: application/json; charset=utf-8
  Date: Thu, 07 Dec 2017 11:29:51 GMT
  Server: Jetty(9.2.21.v20170120)

  [
      {
          "description": "Write Duct guide",
          "id": 1
      }
  ]

We can now have the bare bones of a useful application.


Becoming More RESTful
"""""""""""""""""""""

We can now GET and POST to lists of entries for our Todo application,
but ideally we'd also like to DELETE particular entries as well. In
order to do that, each entry needs to have a distinct URI.

Let's start by adding some hypertext references to our list handler:

.. code-block:: edn

  [:duct.handler.sql/query :todo.handler.entries/list]
  {:sql   ["SELECT * FROM entries"]
   :hrefs {:href "/entries/{id}"}}

The ``:hrefs`` option allows hypertext references to be added to the
response using `URI templates`_. If we ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main dev user)
  :resumed

And test::

  $ http :3000/entries
  HTTP/1.1 200 OK
  Content-Length: 63
  Content-Type: application/json; charset=utf-8
  Date: Thu, 07 Dec 2017 21:13:20 GMT
  Server: Jetty(9.2.21.v20170120)

  [
      {
          "description": "Write Duct guide",
          "href": "/entries/1",
          "id": 1
      }
  ]

We can see that each list entry now has a new key. Let's write two new
Ataraxy routes:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]

   [:get    "/entries/" id] [:entries/find    ^int id]
   [:delete "/entries/" id] [:entries/destroy ^int id]}

These routes show how we can pull data out of the URI, and coerce it
into a new type.

The routes require associated handlers. As before, we'll make use of
the `duct/handler.sql` library, using the `query-one` and `execute`
handler types:

.. code-block:: edn

  [:duct.handler.sql/query-one :todo.handler.entries/find]
  {:request {[_ id] :ataraxy/result}
   :sql     ["SELECT * FROM entries WHERE id = ?" id]
   :hrefs   {:href "/entries/{id}"}}

  [:duct.handler.sql/execute :todo.handler.entries/destroy]
  {:request {[_ id] :ataraxy/result}
   :sql     ["DELETE FROM entries WHERE id = ?" id]}


We also want to improve the entry creation route and give it a
`Location` header to the resource it creates:

.. code-block:: edn

  [:duct.handler.sql/insert :todo.handler.entries/create]
  {:request  {[_ description] :ataraxy/result}
   :sql      ["INSERT INTO entries (description) VALUES (?)" description]
   :location "/entries/{last_insert_rowid}"}

The `last_insert_rowid` is a resultset column specific to
SQLite. Other databases will return the generated row ID in different
ways.

With all that done we `reset`:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :resumed

And test::

  $ http :3000/entries/1
  HTTP/1.1 200 OK
  Content-Length: 61
  Content-Type: application/json; charset=utf-8
  Date: Sat, 09 Dec 2017 12:59:05 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "description": "Write Duct guide",
      "href": "/entries/1",
      "id": 1
  }

  $ http delete :3000/entries/1
  HTTP/1.1 204 No Content
  Content-Type: application/octet-stream
  Date: Sat, 09 Dec 2017 12:59:12 GMT
  Server: Jetty(9.2.21.v20170120)


  $ http :3000/entries/1
  HTTP/1.1 404 Not Found
  Content-Length: 21
  Content-Type: application/json; charset=utf-8
  Date: Sat, 09 Dec 2017 12:59:18 GMT
  Server: Jetty(9.2.21.v20170120)

  {
      "error": "not-found"
  }

  $ http post :3000/entries description="Continue Duct guide"
  HTTP/1.1 201 Created
  Content-Length: 0
  Content-Type: application/octet-stream
  Date: Sat, 09 Dec 2017 13:18:46 GMT
  Location: http://localhost:3000/entries/1
  Server: Jetty(9.2.21.v20170120)

.. _URI templates: https://tools.ietf.org/html/rfc6570


Code
~~~~

So far we've seen how the configuration can be leveraged to produce
applications in Duct. This works well when our needs are modest, but
for most applications we're going to have to knuckle down and write
some code.

While defining handlers using data has advantages, it's important not
to take this too far. Treat the configuration as the skeleton of your
application, and the code as the muscles and organs that drive it.


Adding Users
""""""""""""

So far our application has been the single-user variety. Let's change
that by adding a ``users`` table. First we'll add a reference to a new
migration in the configuration:

.. code-block:: edn

  :duct.migrator/ragtime
  {:migrations [#ig/ref :todo.migration/create-entries
                #ig/ref :todo.migration/create-users]}

Then create the migration:

.. code-block:: edn

  [:duct.migrator.ragtime/sql :todo.migration/create-users]
  {:up ["CREATE TABLE users (id INTEGER PRIMARY KEY, email TEXT UNIQUE, password TEXT)"]
   :down ["DROP TABLE users"]}

And ``reset`` to apply the new migration:

.. code-block:: clojure

  dev=> (reset)
  :reloading ()
  :duct.migrator.ragtime/applying :todo.migration/create-users#66d6b1f8
  :resumed

Now that we have a table to hold our users, we next need to provide a
way for people to sign up to our web service. We could write a handler
for this with the ``duct/handler.sql`` library, but good security
practice tells us that we should avoid writing passwords directly to
the database.

Instead, we'll be writing our own handler function, one that secures
the password with a `key derivation function`_ or KDF. To do this, we
first need to introduce a new dependency to the project file:

.. code-block:: clojure

  [buddy/buddy-hashers "1.3.0"]

This is the library that we'll use to supply our KDF. Once the
dependency is in place, exit the REPL:

.. code-block:: clojure

  dev=> (exit)
  Bye for now!

Then restart::

  $ lein repl

And start the application:

.. code-block:: clojure

  user=> (dev)
  :loaded
  dev=> (go)
  :duct.server.http.jetty/starting-server {:port 3000}
  :initiated

Next we want to add in an additional Ataraxy route that allows users
to be created:

.. code-block:: edn

  :duct.module/ataraxy
  {[:get "/"]        [:index]
   [:get "/entries"] [:entries/list]

   [:post "/entries" {{:keys [description]} :body-params}]
   [:entries/create description]

   [:get    "/entries/" id] [:entries/find    ^int id]
   [:delete "/entries/" id] [:entries/destroy ^int id]

   [:post "/users" {{:keys [email password]} :body-params}]
   [:users/create email password]}

And we next write the handler configuration:

.. code-block:: edn

  :todo.handler.users/create
  {:db #ig/ref :duct.database/sql}

You'll notice that this isn't a composite key; we're not using
existing functionality, but instead we're going to write our own
method.

You might also notice that we're also including a reference to the
database. All SQL database keys in Duct inherit from
``:duct.database/sql``, so by using that key in the reference we're
telling Duct to find the first available SQL database.

You may wonder why the ``duct.handler.sql`` keys didn't include a
database key. This is because they all inherit from the
``:duct.module.sql/requires-db`` keyword, which is a indicator to the
``:duct.module/sql`` module to automatically insert the reference. We
could also do this, but for now we'll keep the reference explicit.

It's now finally time to write the handler. The namespace of the
keyword is ``todo.handler.users``, so we'll use that as the namespace
for the code. Create a new file ``src/todo/handler/users.clj`` and add
a namespace declaration:

.. code-block:: clojure

  (ns todo.handler.users
    (:require [ataraxy.response :as response]
              [buddy.hashers :as hashers]
              [clojure.java.jdbc :as jdbc]
              duct.database.sql
              [integrant.core :as ig]))

Naturally we need ``buddy.hashers`` for our KDF, and we need
``clojure.java.jdbc`` because we're accessing the database. The
``integrant.core`` namespace is necessary because we're writing an
Integrant multimethod, but the purpose of ``ataraxy.response`` and
``duct.database.sql`` might be less obvious.

Let's create the function to insert the new user into the database,
and return the ID of the newly created row:

.. code-block:: clojure

  (defprotocol Users
    (create-user [db email password]))

  (extend-protocol Users
    duct.database.sql.Boundary
    (create-user [{db :spec} email password]
      (let [pw-hash (hashers/derive password)
            results (jdbc/insert! db :users {:email email, :password pw-hash})]
        (-> results ffirst val))))

If you're new to Duct, you might be surprised that we're using a
protocol here. Why not just write a function? Why are we writing a
protocol, then implementing it against this mysterious
``duct.database.sql.Boundary`` type?

The answer is that we *could* use a function, and it would certainly
save us a few lines, but by using a protocol we gain the capability to
mock out the database for testing or development. Duct provides an
empty 'boundary' record, ``duct.database.sql.Boundary``, for this
purpose. This is why we need to require the ``duct.database.sql``
namespace, or the record will not be loaded.

Finally, we write the ``init-key`` method for our keyword:

.. code-block:: clojure

  (defmethod ig/init-key ::create [_ {:keys [db]}]
    (fn [{[_ email password] :ataraxy/result}]
      (let [id (create-user db email password)]
        [::response/created (str "/users/" id)])))

Ataraxy allows a vector to be returned instead of the usual Ring
response map. This is both a convenience, and an abstraction. Ataraxy
will turn this into a ``201 Created`` response map for you.

Let's ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (todo.main todo.handler.users dev user)
  :resumed

Then test it out::

  $ http post :3000/users email=bob@example.com password=hunter2
  HTTP/1.1 201 Created
  Content-Length: 0
  Content-Type: application/octet-stream
  Date: Mon, 11 Dec 2017 14:10:31 GMT
  Location: http://localhost:3000/users/1
  Server: Jetty(9.2.21.v20170120)

We don't have any way of visualizing this information yet, so we need
to take a look at the database.

.. _key derivation function: https://en.wikipedia.org/wiki/Key_derivation_function


Querying the Database
"""""""""""""""""""""

During development we likely want to query the database to ensure that
the code we write is inserting the correct data. To make this process
easier, we'll be adding to the ``dev`` namespace in
``dev/src/dev.clj``.

First, we want to require the ``clojure.java.jdbc`` namespace:

.. code-block:: clojure

  [clojure.java.jdbc :as jdbc]

Next we want a way of getting a database connection. Duct stores the
running system in the ``system`` var during development. This allows
us to write a simple function to retrieve a JDBC database spec:

.. code-block:: clojure

  (defn db []
    (-> system (ig/find-derived-1 :duct.database/sql) val :spec))

Now that we can get the database, we can add a small function to help
us query it:

.. code-block:: clojure

  (defn q [sql]
    (jdbc/query (db) sql))

Once these changes are made, we ``reset``:

.. code-block:: clojure

  dev=> (reset)
  :reloading (dev)
  :resumed

Then try querying our ``users`` table:

.. code-block:: clojure

  dev=> (q "SELECT * FROM users")
  ({:id 1,
    :email "bob@example.com",
    :password
    "bcrypt+sha512$f4c1bc592ecd1869d0bf802f7c8f6e36$12$19a9ae3ed9118cb6cbfcd8c4a31aadb6b00162288b1fce50"})

That certainly looks correct. We have an ID, email and an hashed password.
