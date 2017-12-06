Duct
====

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


Adding a Route
""""""""""""""

Now that we have a running project, we can start making changes to
it. Duct applications are built around an edn_ configuration file,
which in this project is located at ``resources/todo/config.edn``:

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

This time we're using a vector as the key. Vector keys inherit the
properties of all the keywords contained in them; because the vector
contains the key ``:duct.handler.static/ok``, the configuration entry
produces a static handler.

Let's apply this change to the application. Go to back to the REPL and
run:

.. code-block:: clojure

  user=> (reset)
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
