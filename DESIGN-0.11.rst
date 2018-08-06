Duct 0.11 Design Decisions
==========================

Preface
~~~~~~~

Duct 0.11 will introduce breaking changes to the project. This
document covers these changes in detail, and explains the reasons they
were implemented.

These changes have already been implemented in the alpha version,
which can be tested with::

  lein new duct-alpha <project> <hints...>


Modules
~~~~~~~

In 0.10
"""""""

In version 0.10 and below, modules were written:

.. code-block:: clojure

  (defmulti ig/init-key ::module [_ options]
    {:req [::precondition]
     :fn  (fn [config] config)})

Initiating the module key returns a map. The ``:fn`` key contains a
pure function that is used to transform the configuration. The
``:req`` key contains a collection of keys the module requires to be
present in the configuration. The required keys are used by Duct to
apply the modules in order of their dependencies.

The problem with this approach is that there is already a more
sophisticated mechanism for managing dependencies in Integrant. Rather
than having two separate systems for dependency ordering, it makes
more sense to use the same system for both modules and normal keys.

In 0.11
"""""""

In previous versions of Integrant, there was no way to add refs to a
key beyond changing the configuration. If we want to make a module
depend on another key, a way of adding refs automatically is required.

Integrant 0.7 introduces the ``prep-key`` multimethod for this
purpose, which is called by ``prep`` for each key before a
configuration is initiated. This allows refs to be added after the
configuration has been written, producing a dependency ordering.

From Duct version 0.11 onwards, modules are therefore written:

.. code-block:: clojure

  (defmulti ig/prep-key ::module [_ options]
    (assoc options ::requires (ig/ref ::precondition)))
                
  (defmulti ig/init-key ::module [_ options]
    (fn [config] config))

In the prep stage, a ref is added under a private ``::requires``
key. When the modules are initiated, this ensures that the
``::module`` key is initiated after the ``::precondition`` key.


Profiles
~~~~~~~~

In 0.10
"""""""

In previous versions of Duct, modules and normal Integrant keys
existed within the same configuration:

.. code-block:: clojure

  {:duct.core/project-ns foo
   :duct.module/example  {}}

However, this meant that the configuration had to be split into module
keys and non-module keys before it could be initiated. It also lead to
problems if module keys and non-module keys shared refs.

While it's convenient to have module and non-module keys in the same
configuration map, it also produces a leaky abstraction.


In 0.11
"""""""

The solution to this is to explicitly separate module keys from
non-module keys. The way Duct handles this in version 0.11 is to make
every key in the configuration a module. Non-module keys are placed
into profiles, like so:

.. code-block:: clojure

  {:duct.profile/base   {:duct.core/project-ns foo}
   :duct.module/example {}}

Profiles are just modules that meta-merge their value into the
configuration.

This results in a tiered structure with a clear separation between
tiers: a Duct configuration produces an Integrant configuration, which
is then used to create a running system of dependent components.

To ensure that profiles are run before any other module, we need a way
of defining a set of dependencies. Integrant 0.7 introduces refsets to
solve this problem. Refsets act like refs, except they produce a set
of all matching keys.

We can use refsets to ensure that any key derived from
``:duct/module`` must be applied after a key deriving from
``:duct/profile``. In ``duct.core`` there is the following definition
that does exactly that:

.. code-block:: clojure

  (defmethod ig/prep-key :duct/module [_ profile]
    (assoc profile ::requires (ig/refset :duct/profile)))

Between refs, refsets and keyword inheritance, we can set up
sophisticated but predictable dependency graphs.


Includes
~~~~~~~~

In 0.10
"""""""

In version 0.10 and below, includes were handled by a special key,
``:duct.core/include``:

.. code-block:: clojure

  {:duct.core/include ["example"]}

This will look for a resource named ``example.edn`` and meta-merge it
into the configuration.

There are two problems with this approach.

The first and most obvious issue is that it requires one key to have a
special function, one that isn't defined by a standard multimethod. It
cannot be a module because it's side-effectful.

The second issue is that it introduces new side-effects after the
configuration has been read. Ideally we want reading the configuration
to happen at the same step.

In 0.11
"""""""

In version 0.11 the ``:duct.core/include`` key is replaced with the
``#duct/include`` reader tag. The tag is replaced by the contents of
the referenced resource. If we want to merge it into the
configuration, we place it in a profile:

.. code-block:: clojure

  {:duct.profile/example #duct/include "example"]}

This ensures all the included configurations are read together by
``read-config``, and moves the complexities of merging into the
profiles.

This approach also allows smaller chunks of data to be included from
external files, rather than full configurations.


Summary
~~~~~~~

The changes represent an overall simplification of the module and
include system:

- Modules and normal components are separated.
- Modules no longer use their own dependency management.
- Merging is separated out into profiles.
- Including other configurations happens at read time.
- No keys with 'special' functionality.

In addition to the simplification, extra functionality has been added:

- ``prep-key`` removes the need for modules in simple cases
- ``refset`` allows for more sophisticated dependencies
