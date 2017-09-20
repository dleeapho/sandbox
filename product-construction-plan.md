Product Construction Plan
=========================

Situation
=========

Dependency Isolation
--------------------

Product dependencies are the build outputs that each component repo produces and
consumes. The aggregate composition of these dependencies is what composes the
whole product. Toolsets are a dependency but are distinct from product
dependencies since they can be thought to be external to the composition of the
product; toolsets are used to build each component repo. Today the whole of the
.NET Core product is built on a per-repo basis and the whole emerges from the
parts through manual coordination of the dependent packages to be consumed and
the packages to produce (the uptake of product dependency is essentially an
opt-in for the consuming repo). Dependency isolation makes the assembling of any
release candidate ad hoc and indeterminate from a systems perspective.
Nevertheless, dependency isolation valuable when repos wish to favour repo
isolation, say in the early stages of development leading up to a product
release.

To summarize, repo-local efficiency is an outcome of the relative autonomy of
each repo to consume the version of its dependency as it sees fit. However the
isolation afforded by this model becomes a hindrance to assembling the release
candidates of the whole product as it approaches stabilization and integration.

We need a parallel mechanism whereby we can build the product as a whole that
automatically flows the latest version of dependencies within the repos that
constitute the whole product.

Build Isolation
---------------

As a side-effect of the current repo isolation is that official builds of each
repo are implemented in a way that is also locally efficient, e.g. each repo
builds and signs its bits, publishes and runs a gamut of tests. This build
isolation produces long lags for the flow of any single change to ripple through
to a release candidate of the whole product, viz. it can take days to propagate
through each of the repos’ build, sign, publish, test cycle across all the repos
that compose the whole product. Much like repo dependency isolation, the ability
of each repo to build locally is vital when repos need to favour local speed in
the early stages of development. Yet all the isolated repo builds must come
together as a product.

We need a parallel whole product build that relies on automatic dependency flows
then batches the signing, publishing and testing of the whole product. The
output of such a build are potentially distributable artifacts that comprise the
Microsoft distribution of .NET Core. The efficiency gained by providing a whole
product build makes “moving from days-to-build to builds-per-day” a tenable
goal.

Circular Dependencies
---------------------

CLI currently has a circular dependency with the SDK and itself, this
complicates the flow of dependencies across the product. It is also a
performance hit for building the product. CLI also has a circular dependency
with ASP.NET templates, offline store, and lzma cache which complicates the
integration with ASP.NET and VS. Circular dependencies make the product build
non-deterministic.

We need to eliminate circular dependencies within the product.

Toolset Autonomy
----------------

Each repo currently is free to define its own toolset. Toolset includes
compilation toolchain components like CMAKE, CLANG, etc. and at the heart of
many repos, the CLI itself is part of the toolset. Toolsets are dependencies but
are distinct from product dependencies in that they can be more uniformly
adopted across repos. Toolset autonomy is valuable in the establishment phase of
a repo, i.e. before it is brought into the product. Once in the product, toolset
dependencies need to be coordinated, especially in the case of source build.
Toolset mismatch is often a hindrance to integration of repos into a release
candidate and hugely complicates servicing.

We need to normalize and toolset versioning and acquisition across the product.

Objective
=========

Repo isolation is appropriate and useful in the active development stages of a
product release— we continue to support isolated repo build efficiency at this
stage. As the components of the product needs to coalesce into unified release
candidates, coordination and product-wide efficiency must be favoured. The team
will implement a parallel build system that constructs the whole product based
on automatic latest-version dependency flow across the component repos, the
build will be structured and optimized release candidates multiple times per
day.

Execution
=========

Concept
-------

The source-build repo is the basis of constructing the whole product. It must
first be be made sustainable. Currently it is a vertical build that provides
source buildable tarballs, we propose to extend it to also build all supported
platforms and to produce the Microsoft distributable .NET Core componets
efficiently. For each repo to be aggregated within source-build with automatic
dependency flow and to be optimizable within the structure of source-build, each
repo that is a part of the .NET Core product will need to hew to contracts that
enable source-build to drive product construction. The exact nature of the
contract will be driven by the implementation of the coordination mechanisms in
the new source-build. Once established, will ask each repo to adhere to a set of
standardized behaviours and to adopt toolsets that enable repo coordination by
source-build . Finally, we need to implement orchestration to: 1. Provision and
manage machines, scale 2. Coordinate the vertical builds, harvest the output to
produce the distributable product 3. Finalize the build with signing, testing,
publishing) 4. This is likely some implementation of existing or future build orchestration infrastructure.

![Diagram of Product Build](media/48887747198ca0c1cf727299d020e64c.png)

Diagram of Product Build

[Visio diagram of Product
Build](https://microsoft.sharepoint.com/teams/netfx/engineering/Shared%20Documents/Planning/ProductConstructionOrchestration.vsdx)

### Requirements for each repo

1.  Builds steps must be separated and independently executable.

    1.  The basic operations of restore, build, package, sign, and test for each
        repo need to be independent. This is necessary to provide the
        flexibility to reorganize each repo’s build steps into the product build
        by enabling reordering and ‘batching’ of operations. Furthermore, the
        factoring will allow for idempotent operations to be retry-able and thus
        more resilient to transient external outages.

2.  Dependency flows can be automatic

    1.  Product dependencies are declared in a common way, viz. what the repo
        consumes as dependencies and what the repo produces for downstream
        repos.

    2.  Toolset dependencies are declared and acquired through common means.
        Certain toolsets that drive contract implementations will be made
        mandatory throughout the product and versioned centrally.

3.  Each repo build must be deterministic

    1.  This is an eventual requirement as we move the whole product to build
        deterministically. Deterministic product builds will become a necessity
        to fulfill source build requirements from Linux distros. For now, repos
        should not introduce new indeterminism in their builds.

### Requirements for source-build repo

1.  Source-build’s knowledge of product dependencies must be derivable from each
    constituent repo’s dependency declaration (what it consumes and what it
    produces).

2.  Source-build must produce a source buildable environment at will and with no
    special intervention for supported platforms.

3.  Source-build will determine the structure and ordering of repo builds that
    construct the product.

### Requirements for Engineering Services

1.  Prescribe the dependency contract

2.  Prescribe the build operations contract

3.  Provide default, extensible implementations of the contracts

4.  Provide for uniform toolset declaration and acquisition

Non-requirements
----------------

**Alternate ways to express inter-repo dependency:** The NuGet protocol is the
current way inter-repo dependencies are expressed and implemented. Although a
coordinated and orchestrated build is possible using a more centralized
composition scheme (eg. one big repo), any such scheme merely shifts the
coordination problem to some as yet unknown and unimplemented mechanism. This is
risk and churn that the product cannot afford now.

Tasks
-----

To reach the objective the main effort will be to extend source-build to aggregate constituent enable
repo builds uniformly then enable optimized product
builds. These are large tasks with significant technical hurdles to be
surmounted— we need to continue to iterate on designs for these tasks. Overall,
the approach is to be scenario-focused, eg enable automatic dependency flow for
two repos through source-build for one vertical build then broaden as we learn
from that implementation. Nevertheless, there are supporting tasks that will be
important in rationalizing dependency flows and enabling toolset acquisition and
flow. We have enough prior effort on these tasks to start implementation now.

1.  V-Team: Design and implement a product build of .NET Core

    1.  Epic: Make source-build sustainable [Epic
        \#122](https://github.com/dotnet/source-build/issues/122)

        -   Objective: Source-build can produce community and Red Hat
            source-builds with a clean checklist of known steps to ‘sync’ with
            each repo and external dependencies. Also we have the basis of
            engaging with the community credibly.

        -   Issues:

            1.  Fix CI so that it passes consistently \#100

            2.  Fix build issues across the ‘supported’ platforms (Ubuntu,
                CentOS, MacOS, RHel, Windows) (not sure of this list)

            3.  Problem: Repositories.props has old branch information

            4.  Provide usage and contribution guidance to the community

            5.  Trim /tests from source tarball

            6.  Implement a branching strategy for releases and servicing (same
                branches as CLI repo)

            7.  Manage external dependencies (eg. Application-insights,
                NuGet-Client, Newtonsoft-json, etc) - a general policy not just
                narrowly for Red Hat.

        -   Teams: Eng Services - source-build team

        -   Estimate (calendar sprints): 1

        -   Remarks: Start with 2.0 servicing, then move to master. 1.x
            servicing branches as appropriate.

    2.  Epic: Source-build can consume and aggregate constituent repo builds
        uniformly. [Epic #145](https://github.com/dotnet/source-build/issues/145)

        -   Objective: Source-build can automatically produce community and Red
            Hat source-builds with no special intervention (patches are
            eliminated for repos). Builds are still vertical for each platform
            Window, MacOS, Linux.

        -   Issues:

            1.  Design a repo API and the source-build uptake of it

            2.  Ask each repo owner to implement the API (CoreClr, CoreFx, CLI,
                Core-setup, Roselyn, buildtools, standard, sdk, cli-migrate,
                f-sharp - across 2.0 Servicing and master branches)

            3.  Get our toolset and build environment declarative and controlled

        -   Teams: Eng Services - source-build team, Repo teams

        -   Estimate (calendar sprints): 5

        -   Remarks:
            [Auto-Dependency-FlowREADME.md](https://github.com/dagood/core-eng/blob/95fca0a7e2606b1d4673793e3f26cfa0308f2df8/Documentation/Project-Docs/Product-Construction/Auto-Dependency-Flow/README.md)
            [Auto-Dependency-Flow-contracts.md](https://github.com/dagood/core-eng/blob/95fca0a7e2606b1d4673793e3f26cfa0308f2df8/Documentation/Project-Docs/Product-Construction/Auto-Dependency-Flow/contracts.md)

    3.  Epic: Make source-build capable of producing Product Build ie, the
        Microsoft .NET Core distributable components

        -   Objective: Product Build that is ‘builds per day'

        -   Issues:

            1.  Make source-build builds scalable​ - implement build
                orchestration in our Jenkins CI/CD system.

            2.  Combine vertical builds into the product distribution​

            3.  Signing - find a solution to the problems of batch signing the
                product.

            4.  Symbols production

            5.  Testing​ - understand how to optimize product build testing and
                implement.

            6.  Engineering telemetry, build dashboard.

            7.  Publishing

        -   Teams: Mostly Engineering Services - source-build, Jenkins CI,
            Helix, MC​

        -   Estimate (calendar sprints): 5

        -   Remarks:

    4.  Epic: Make source-build produce Linux reproducible builds

        -   Objective: a .NET Core that anyone can source-build into their
            platform using idioms like make install

        -   Issues:

            1.  Implement something like: ./configure, make, make install​

            2.  Make the build system completely deterministic

        -   Teams: source-build team

        -   Estimate (calendar sprints): 3

        -   Remarks:

2.  CLI team: rationalize the SDK/CLI dependency flow

    1.  The CLI subsumes the functionality of present in the SDK repo

    2.  Move the compositional aspects of the ‘SDK’ component to a separate repo

3.  CoreFx team: Toolset - move to using more from the shipping SDK

    1.  Use CoreFx repo as a pilot for other repos

    2.  Implement use of shipping version of SDK as toolset.

    3.  If needed, implement use LKG of SDK as an option in lieu of the shipping
        version

    4.  Buildtools support for shipping SDK (with help from Engineering Services
        team).

4.  Core-eng team: enable controlled acquisition of toolsets – makes builds
    across repos more deterministic

    1.  Implement the push model of bootstrap scripts and tool acquisition
        scripts

    2.  Implement the toolset acquisition scripts and tools-version metadata

    3.  Move buildtools to use shipping SDK

Coordinating Instructions – liaison and connections with other ongoing efforts
------------------------------------------------------------------------------

### Transport/Blob Feed

Blob feed is a prerequisit endpoint for automatic-dependency flow.

Build First Principles
======================

These are the core principles which should guide how we implement our core
engineering goals.

Builds are non-mutating
-----------------------

This is simply a standard principle of build that we must respect. The source
tree of a repository should not mutate as a part of building for local or
orchestrated builds. This is fundamental to ensuring builds are reproducable,
incremental and correct.

Mutating builds would also be particularly negative for us as it would require
us to essentially publish a new SHA per build. Having an accessible SHA on our
repository is necessary to support servicing and dump debugging. These must line
up with the actual source changes that were built.

The only way to do that would be to commit every mutation and publish it to a
tag. That would produce an undesirable number of "build" commits to our
repository that each represent a new servicing point. Very undesirable.

Toolsets are configurable
-------------------------

The orchestrator must be able to configure the set of tools used to build a
repository. A repository will naturally have a default method for finding
MSBuild, the C\# compiler, NuGet, etc ... In every case though there must be a
way to explicitly provide a path to the tool to be used.

This is is key for us to support our source build requirements. There our entire
tool chain must be bootstrapped and subsequently used to rebuild our entire
source tree. This also helps teams validation toolset changes by allowing broad
automated testing of new toolsets.

Repositories must conform to an API
-----------------------------------

The number of repositories that comprise our build is ever expanding. In order
to efficiently, or just sanely, manage them they need to have a common surface
area that can be tooled. The goal is to effectively have an API, or more simply
a set of verbs, that every repository provides. For example: restore, build,
produce, etc ...

Repositories should have great flexibility in how core verbs are implemented.
Yet they must conform to a standard presentation. This is key for both being
successful in our orchestration pursuits and in providing a consistent story for
developers.

A developer shouldn't need to digest a lengthy README before doing the most
basic of operations in a repository as is the case today. Instead we need to
present a consistent set of operations that work across all our repository. This
makes it easy for developers, internal and external, to move between our
repositories and contribute fixes.

Repositories can build in isolation
-----------------------------------

The primary use case of our repositories will continue to be developers working
on a repository in isolation. This is our inner loop for bug fixes, feature
development and general day to day activities. This will true even when the
change occurs in a branch which is a part of our orchestrated build scenario.

Repositories will not depend on machine state
---------------------------------------------

Repositories cannot require machine state as a part of implementing our core API
requirements. Doing so would negate the benefits of having a repository API
because the orchestration would need images and tools specific to each
repository. It would also continually impair our ability to build all
repositories on a single machine (without first going through significant setup
headaches).

Repositories can opportunistically leverage machine state when possible. For
instance there is no need to download the CLI when an appropriate version is
already installed. However, repositories cannot require it be installed to
build.

No circular dependencies across repositories
--------------------------------------------

Circular dependencies block deterministic builds and can prevent incremental
builds. Circularity reduces overall throughput.

The .NET Core product can be built from source
----------------------------------------------

In addition to having source-buildable .NET Core components to support Linux
eco-system, the Microsoft distributable components should be buildable from the
same base so that we have a single point that defines both the community release
and the Microsoft distributable components.

Product Construction Concepts and Terms
=======================================

Product Build
-------------

A build of the Microsoft distribution of the .NET Core product. It encompasses
all supported platforms. It is the composition of all constituent independent
repo builds (e.g. CoreClr, CoreFx, CLI, etc).

### Alternative terms:

Whole-product Build, Full-stack Build, Coordinated Build

Product Build Orchestration
---------------------------

The mechanism by which we apply scaleable compute, coordinate services to build,
sign, package, and distribute the product.

### Alternative terms:

Orchestration

Source Build
------------

The build that results in source code tarballs that can in turn be built to
provide a community build of .NET Core (strictly speaking this term should be
understood as meaning “source-build build”).The community build can be in turn
distributed and supported by Distribution Maintainers (e.g. Red Hat). Source
Build is a vertical build targeting a specific platform whereas the Product
Build produces components for targeting all supported platforms that is the
Microsoft distributable components of .NET Core. This term should not be
confused with the repo source-build since the source-build repo is the basis for
building both the community build (built from source) and the Microsoft
distributable (Product Build).

### Alternative terms:

Build from Source.

Reproducible Build
------------------

The ability for isolated repo builds, and in turn the orchestrated and source
builds, to produce byte for byte equivalent output between builds. From
[reproducible-builds.org](https://reproducible-builds.org/):

> Reproducible builds are a set of software development practices that create a verifiable path
>from human readable source code to the binary code used by computers.

### Alternative terms:

Deterministic Build

Repo
----

The source repository for the constituent components that make up the .NET Core
product.

### Alternative terms:

Component Repo

Isolated Repo Build
-------------------

The build of the Repo independent of the assemblage of the product, ie. The repo
is built with its own declaration of dependencies. The Isolated Repo Build may
leverage pinning of dependency versions to isolate itself from upstream changes.

Toolset
-------

External or internal tools (such as CMAKE, CLANG, MSBuild, .NET Core SDK,
BuildTools) used in Isolated Repo Builds or Product Builds. Though toolsets are
a dependency, the references to any toolset is from build scripts or targets.
They are not directly included in the product distribution.

Product Dependency
------------------

In the context of either Product Build or Isolated Repo Builds, these are the
artifacts (usually NuGet packages, though not exclusively so), produced by a
repo build that are used by downstream repos, ie they are referenced in
downstream repo source code.

External Dependency
-------------------

They are the binaries outside of the .NET Core product that are referenced by
repo source code.

Dependency Flow
---------------

In an aggregated build of multiple repos, as each Repo Build is performed in
sequence, dependent artifacts are produced then consumed by downstream repos in
their builds. It is the reification of references declared in repo source code.

Isolated Dependency Flow
------------------------

In an isolated repo build, it is the dependency flow wherein each repo
explicitly references specific versions of the artifacts it consumes (pinned),
e.g. auto-upgrade PRs that can be merged to pin a new version of upstream
dependencies.

Automatic Dependency Flow
-------------------------

In a Product Build, it is the dependency flow wherein each repo references
versions of upstream repo build artifacts that reflect the implicit up-to-date
state of the product. Usually the versions are implicitly the latest produced in
upstream repo builds but there maybe exceptions, such as samples, that reference
pinned prior versions as a product prerogative.

### Alternative terms:

auto-dependency flow
