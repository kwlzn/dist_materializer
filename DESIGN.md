# Goals

The goal of this project is to provide a pip-compatible index server that can materialize source dependencies from a monorepo in a way that delegates control to pip. This affords for consumption of deps from a monorepo in a consistent way that is also mutable and does not require a local copy of the repo.

This provides seamless interop with standard python dependency management tooling which affords for easier consumption of source dependencies in a myriad of environments such as Jupyter Notebooks, IDEs, GCP's Dataflow & AI Platform and so on.

# Theory of Operation

## Namespace Mapping

Installation via e.g. pip uses a namespace mapping with a magic, system-defined prefix and a dot notation that maps to monorepo address space.

For example, the equivalent of e.g.:

```
./pants binary src/python/some/group:cool_package
```

Given a magic prefix of e.g. "monorepo" - would be installed as e.g.:

```
pip install monorepo.src.python.some.group.cool_package
```

## Materialization

Materialization relies on a build system plugin that can emit only intransitive wheels for a given target. Within this intransitive wheel, dependencies are emitted in the wheel metadata that follow the same projection mapping for source deps and use direct deps for 3rdparty. This allows the resolver (e.g. pip) to recurse to complete the transitive set of dependencies and minimizes per-package build costs for lower latency at the index level. This also allows pip to fully manage the transitive environment.

## Indexing and Versioning

Given that source dependencies in a monorepo don't have a version beyond a git sha, all source dependency package versions in this system use a fixed version of 0.0.0 with a PEP440 local version identifier that encodes the git sha. When materialized wheels emit their own projected dependencies, they're always pinned to the same git sha for sha-level consistency (as one would expect/rely on in a monorepo). This ensures that upgrades and other mutations are always consistent with the state of the monorepo at a particular git sha (including branches).

The "latest" version on the index always points to the master sha, such that an unpinned e.g. `pip install monorepo.x.y.z` always consumes master. (TODO: confirm `pip install --upgrade` always tracks "latest" vs manual version incrementalism).

Using the above example, to install `src/python/some/group:cool_package` at master you would run:

```
pip install monorepo.src.python.some.group.cool_package
```

And then to install that at some specific sha, you would run:

```
pip install monorepo.src.python.some.group.cool_package==0.0.0#<sha>
```

# Components

## Index Builder

This component maintains a local copy of the monorepo, does a periodic pull and maintains a global definition of "latest" as the master sha. It also maintains prior recent versions of all git shas in an in-memory store to support sha-specific installation.

## Index Server

This component is an HTTP service that speaks the python index protocol. It reads from the in-memory store maintained by the index builder for low latency serving and invokes the Build System Adapter to materialize wheels for serving. This will leverage it's own caching for latency optimization and will likely need distributed workers for parallelism. To accomodate for build system latency in the request path, the goal will be to utilize intransitive builds, speculative builds as well as e.g. HTTP 503 or similar with a Retry-After header. It is unclear if pip supports this degree of retry robustness currently, but if not an upstream contribution to pip itself might be needed to accomodate.

## Build System Adapter

This component is a configurable subsystem that knows how to invoke your build system (Bazel, Pants, etc) and can produce intransitive wheel builds in accordance with the namespace projection mechanics. Your build system must be capable of providing intransitive wheel builds to support this, which often requires a build system plugin to be installed in the monorepo.
