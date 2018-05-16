+++
title = "An Analysis of vgo"
keywords = ["golang", "go", "clapback", "dep", "dependency management", "package management"]
+++

When Russ started releasing his [series of blog posts introducing vgo](https://research.swtch.com/vgo) in late February, I also [put together some words](https://sdboyer.io/blog/vgo-and-dep). In that post, I indicated that I would be working on an assessment of vgo that I would make public as soon as I could. This, finally, is that assessment, although over the past couple months it has transformed into something more.

Over the course of reviewing vgo, I have vacillated between “this is clearly infeasible” and “this is how all software should be built" - a reflection of the sheer number of interwoven considerations in this problem space. Now, having largely finished my assessment, I think that both of these ostensibly contradictory positions are basically accurate. 

While vgo makes a powerful normative statement about the happy path for software development, it also ignores a number of crucial system properties, particularly those having to do with creating a workable, humane system for the people authoring code within it. These deficiencies lead me to believe that vgo - specifically, the minimal version selection (MVS) set of algorithms - is not fit for purpose, and we should pursue a different model.

Making this case is not a simple matter. It's easy to debate the appropriate scope for a tool like this. What the tool does is intrinsically bound up with difficult or unknowable questions about what is reasonable to expect developers to do. And we have to dive deep into algorithmic properties in order to extract inferences about the social conditions that will result from them.

To cover all that, I've broken this up into a six-post series. This post outlines my concerns and plans in broad strokes, while subsequent posts look in greater detail at specific topics. This is not my day job, though, so I won't be publishing these rapid-fire, one day after another. I will release them as I finish them; as I write this, two more are nearly done.

Throughout the series, I'll be referring to three core algorithms, and to their corresponding tools:

- **MVS**, the engine behind vgo, as it is proposed across the blog posts and formal proposal.
- **[gps](https://github.com/golang/dep/tree/master/gps)**, the engine behind dep, which uses a SAT-based approach.
- **gps2** (not the final name!), a hypothetical SAT-based approach, replacing the Go toolchain like vgo, but driven by an improved version of dep's core algorithm that corrects shortcomings in both MVS and gps.

Let's be clear: I would vastly rather be in a situation where I could reveal gps2 alongside these blog posts, as Russ did with vgo. I find the prospect of writing about hypothetical software nauseating. But time is short, vgo is on the march, and it's impossible to make this evaluation of MVS constructive and useful without a coherent alternative against which to compare. gps/dep can serve as a basis for comparison at times, but the simple fact that they were designed within the constraints that a third-party tool necessarily had to operate within often makes them a poor comparison.

That leaves me no other choice than to write these posts with references to an as-yet-unwritten algorithm. On the plus side, I've written such an algorithm before, and gps2 wouldn't be a huge departure from dep or vgo - more of a happy union. The final post in this series will describe gps2, and a toolchain built around it. 

With that context out of the way, though, the right place to get this started is with the good parts of vgo.

## What Works in vgo

There's a lot about vgo that's great. Outside of the core algorithm, Russ and I agree on almost everything. Some of what vgo proposes to introduce are things that the Go community has been craving for years, and it's wonderful to see them finally happening. So, with the remainder of this series being so critical of MVS, it makes it all the more important that we start with what works. I don't want to give the erroneous impression that some of vgo's other, non-MVS properties are also on my chopping block; I suspect that those are the ones that Go folks might be primarily excited about.

Many of these are also improvements that we had contemplated with dep. But, because [the package management committee](https://docs.google.com/document/d/18tNd8r5DV0yluCR7tPvkMTsWD_lYcRO7NhpNSDymRr8/edit) chose to intentionally limit dep's scope in order to minimize duplication with, and ease later integration into, the Go toolchain, we never pursued them. Not only would a gps2-based toolchain preserve these, but the utility of some would be increased when paired with a more domain-appropriate algorithm.

### Module self-identification

One of the single most powerful properties in vgo is the first line of `go.mod`: 

```go
module root/import/path/for/module
```

This declaration makes it a compiler-enforced requirement that all modules define the root import path at which anything in their tree must be imported. That is, any given package's position in the global space of import paths is determined by how it was imported, not by the package itself. The `module` line, however, means that modules give themselves a canonical import position (aka, name), and that canonicality fluidly extends to all packages.

The ambiguity of importer-defined identity caused a lot of pain for dep. For example, the Go toolchain has disallowed import paths that differ only by casing for years - e.g., `github.com/Sirupsen/logrus` vs. `github.com/sirupsen/logrus`. This was to try to preclude anyone creating a valid Go program that allowed the possibility of clashes between case-sensitive and case-insensitive filesystems. However, the compiler does not and could not determine which of the casing variants is canonical. Either is fine, but there can only be one. This put dep in an awkward position for the unfortunately-common case of `sirupsen/logrus` - so many people were running into it that we had to [add a check to the solver](https://github.com/golang/dep/pull/1079) that enforces a casing invariant.

We considered adding a self-identification property like this in dep. But Glide has just such a property ([`package` in `glide.yaml`](http://glide.readthedocs.io/en/latest/glide.yaml/)). My experience there led me to believe that, if the compiler isn't enforcing a self-identification property, then having such a property just adds more ambiguity, resulting in more harm than help.

Fortunately, vgo fully embraces the idea of self-identifying code, which not only solves these problems but paves the way to what is probably the single most exciting property for Go developers: the elimination of GOPATH.

### GOPATH-less development and versioned source storage

Getting rid of GOPATH has been a holy grail for almost as long as there has been a GOPATH. Finally, here it is! No longer will Go code be awkwardly sequestered away from the rest of our code, and weaving Go code into larger monorepos will become trivial overnight. Of course, code that was previously on GOPATH has to go somewhere, and that space needs to store versioned code. Enter, `$GOPATH/src/v/`.

The only necessary condition for this is the compiler be able to receive an explicit list of module versions for a given build. It doesn't matter whether that list is produced by MVS, some other algorithm, or read from a file. In fact, I [sketched out a proto-proposal](https://gist.github.com/sdboyer/def9b138af448f39300cb078b0e94cc3) in early 2017 that was dep-centric, but with the same goals. The first half of that proposal is very nearly the same as what vgo now does; the similarities are such that we could replace MVS with a more appropriate algorithm, and get all of this very nearly for free.

I have only one qualm with vgo's current design of `$GOPATH/src/v`. By placing human-readable versions directly in the path structure, it means the filesystem tree is not strictly immutable, as a `git push --force` to a tag can require a restructuring of the paths therein. As a result, it's technically unsafe to simultaneously run multiple `vgo get` operations against the same `$GOPATH/src/v`. 

`go.modverify` can make that safe, but has other problems that we'll explore in a later post. Fortunately, well-executed registries can also probably obviate the issues, and more completely than `go.modverify`.

### Registries

The topic of having a registry - something comparable to a CPAN, crates.io, or rubygems.org - has long been a controversial topic in the Go community. Generally speaking, such discussions have centered around the possibility of a hosted registry: a service to which packages are published canonically, in lieu of direct interaction with source control. "Canonically," as in, their module path points to the registry, not, say, GitHub.

Instead of a hosted registry, the vgo blog posts outline a proxy registry - a service with which the toolchain communicates over HTTP, that in turn reaches out to VCS hosting services - as a way of simplifying the tool itself. Proxy registries could also rely on a sub-service that provides a globally-replicated mostly-sane `go.modverify`, obviating many of the issues described above. Folks at Microsoft are working on the [Athens project](https://github.com/gomods/athens) now, which hopes to cover a vgo-style registry, and more.

A proxy registry will be tremendously helpful. Managing interactions with VCS systems has been a consistent thorn in dep's side - so much so that we began work towards registries at the Gophercon 2017 hack day. But our work was twofold - we were looking at both proxy and hosted registries simultaneously, as it seemed they could easily share an HTTP API. We produced provisional specifications, both [behavioral](https://docs.google.com/document/d/1pKPhSoMJd4KAj7twz0X2Elop7GqClk2D84UHLZhsg-I/view) and [HTTP](https://docs.google.com/document/d/1RUdO08qgFg47S7sg5zoXvwuxde3HiKQVazsSMiQER1M/edit?usp=sharing), and there were a couple (unmerged) pull requests that put provisional support for these registries into dep (hat tip to JFrog folks for their help on that!).

Hosted registries have their pros and cons, but probably the single most valuable thing they can do is act as a gatekeeper. The most obvious example of that is semver enforcement: a hosted registry could reject attempts at releases that do not follow [API-level backwards compatibility rules](https://golang.org/doc/go1compat#expectations). ([Elm does this.](https://github.com/elm-lang/elm-package#publishing-updates)) Proxy registries, being proxies, can't block the creation of the release/git tag, so the only way they can truly enforce is to filter out noncompliant versions. That gets untenable quickly.

### Nested modules & monorepos

Nesting multiple modules within a single repository will not be useful for everyone, but it will be life-changing for those who do need it. Decoupling the unit of dependency (modules) from the unit of storage (source control) makes it far easier to manage complex projects over time, as it is often desirable to colocate related but distinct modules that we're working on simultaneously. This is especially true given the way that services (e.g. GitHub, and GitHub-centric supporting services) organize their tooling around repositories. Splitting out related projects into separate repositories often presents prohibitively difficult workflow challenges.

With dep, we made the choice early on that we wouldn't support nested projects - that we'd equate the unit of storage with the unit of dependency. It was a difficult tradeoff. We were trying to preclude scenarios where individual packages from the same repository might be brought in at different versions. They, in turn, might bring in packages from other repositories at differing versions, and it turns into a fractal pretty quickly. If the packages were unrelated but colocated in a monorepo, these outcomes were mostly fine; however, it was prohibitively difficult to establish that intent on the part of the package producer.

Vgo does two things that make module nesting feasible:

* While vgo can convert existing Go projects on the fly for a number of the existing tools, it only does so for metadata files at the repo root. vgo only allows an import path that points to a subdirectory of the repo root to act as a module if there is an explicit `go.mod` file in it.
* The separate source storage area. The toolchain's semantics for `vendor` - that it covers all sibling directories and their children - meant that nested projects within a monorepo were nonsensical for dep, because you couldn't build them correctly with `vendor`. That is, if your monorepo had a `monorepo/cmd/foo/vendor`, it wouldn't cover `monorepo/lib/bar`, which you might want to use from your `foo` command. Having a separate storage area, so that import path lookups are no longer current-package-position-independent, solves this. This is also why vgo [will only support `vendor` at the root of a nested module tree](https://github.com/golang/go/issues/25073).

Both of these are orthogonal to MVS. gps2 would follow the exact same pattern.

### High-Fidelity Builds

vgo's concept of ["high-fidelity builds"](https://research.swtch.com/vgo-mvs) is an essential aspect of the system design and a natural outgrowth of MVS. It guarantees, among other things, that when first incorporating some module `A@v1.0.0`, which depends on module `B@v1.5.0`, then `B@v1.5.0` will be what ends up in your depgraph, even if there are dozens of newer versions of `B`. The only possible deviation from this is if you already have `B` in your module's depgraph, in which case the newer of the two versions will be used. (Maven uses [an algorithm with a similar property](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Transitive_Dependencies), albeit while operating on version sets that have no notion of a compatibility ordering.)

There's been a fair bit of speculation about whether this would induce people to update not enough, or perhaps too much; Russ indicates on the vgo blogs that he believes it will be the Goldilocks amount - "just the right speed." The confusion here is indicative of something significant, for reasons we'll get into in the failures post. But of all the strategies one could use for version selection, something like this is certainly the single most likely one to produce a working build.

In fact, I like this class of strategy so much that [i wrote about it two years ago](https://medium.com/@sdboyer/so-you-want-to-write-a-package-manager-4ae9c17d9527/#b99d), and designed gps with this capability under the moniker "preferred versions" Today, we could turn preferred versions on in dep more or less like flipping a switch. We haven't yet because I didn't think it was reliable enough yet, and we hadn't worked out a clear CLI interface. (More context in [this issue](https://github.com/golang/dep/issues/622#issuecomment-303259017).)

However, vgo has given me some ideas about how to reduce the scope of what preferred versions do without hampering their effectiveness. I'm now reasonably confident that I have a way of dealing with the problems with preferred versions in general. I'm certain, however, that if we restrict it to _just_ newly-added dependencies - the example above, and Russ' [trophy example](https://research.swtch.com/cargo-newest.html) - it would be straightforward, have zero reliability issues, and might actually solve 95% of the problem in practice. I'm working on a PR, in between writing these posts.

If you've read any of the vgo materials, it might be surprising that dep has something like this at all. Everywhere that the high-fidelity property is discussed, Russ contrasts it against dep/cargo/pub etc. in a manner that makes it easy to erroneously infer that those systems could never have a high fidelity property - despite my explicit request that at least some reference to preferred versions be made. Perhaps Russ chose to omit it because dep's preferred versions implementation isn't live. In any case, it's unhelpful to the discussion for folks to be under the erroneous impression that losing MVS necessarily entails losing the high-fidelity property.

### Pseudoversions (mostly)

Pseudoversions are a way of defining a [total order](https://en.wikipedia.org/wiki/Total_order) across both explicitly released semver-style versions, and arbitrary revisions. They combine the semantic ordering of releases with a chronological ordering of any non-release revision.

Simply imposing an ordering is not mechanically difficult - gps has [functions](https://godoc.org/github.com/golang/dep/gps#SortForUpgrade) that [do exactly that](https://github.com/golang/dep/blob/4c74ecd27d7c268e1e9ce6a043c10cf7c46db40f/gps/version.go#L800). But gps' approach is an arbitrary way of relating the [different version types](https://golang.github.io/dep/docs/Gopkg.toml.html#version-rules) in its model, chosen primarily as a way of servicing a desired outcome in the solver; there is nothing intrinsically meaningful about ["semver before branches"](https://github.com/golang/dep/blob/4c74ecd27d7c268e1e9ce6a043c10cf7c46db40f/gps/version.go#L664).

Pseudoversions also discard certain classes of information in an interesting and useful way. I have seen people create all manner of convoluted histories in git repositories, then turn to their build systems to somehow work well on top of these flows. This has irked me for years, but without a clear rule that I could use to sort these workflows into "saner" and "less sane," it was difficult to imagine how I might eliminate them. The way vgo uses pseudoversions strikes me as a good candidate for such a rule:

- Any revision in a git (or whatever) repository can be referenced, but
- Automated updates will only really help with chasing the tip of a branch

These two, respectively, cover the two cases I see most often, and therefore strike me as "saner":

- When sending a PR with a fix to some upstream repository, you want to point to the latest commit in the PR while you wait for it to be accepted
- Companies that have just one mainline of development, and everything just chases tip

Pseudoversions aren't an unqualified win, though. Clustering them all on `v0.0.0` leads to MVS making some absurd choices under certain circumstances, like when mixing branch-chasing with tagged releases. This would be particularly acute during the period of migration to an MVS-based toolchain, as projects that have been tagging for years and have entered the `v2.x.x` range and above will only be able to access those tags as pseudoversions. This is another one of those problems that I believe that gps2 would address.

### Semantic Import Versioning (mostly)

The basic premise of [semantic import versioning (SIV)](https://research.swtch.com/vgo-import) rings true: a change in behavior should be accompanied by a change in name. That Go has such a (relatively) easy way of renaming - versioned import paths - is a happy coincidence. It's even better because versioning an import path doesn't require any compiler magic, as it's effectively just renaming a tree of packages. Its utility extends beyond Go, though, as evidenced by the fact that it's at least partially inspired by ideas from Clojure. Personally, I'd go so far as to say that language designers should consider "easy package renaming" to be an important property for new languages.

SIV is really just about defining a correspondence between the name used for a given module, and the set of versions that can apply to that module. All of that precedes what algorithms (MVS/gps, etc.) might be exploring those sets of versions - meaning that SIV is not dependent on, or associated with, any particular algorithm. It could certainly be helpful in dep, and would likely be a cornerstone of . MVS, though, absolutely cannot exist without SIV, as it cannot make sound decisions without the compatibility invariants SIV is supposed to provide.

Unfortunately, SIV is not a zero-cost abstraction. In some cases, even "high-cost" would be charitable. The costs of safely allowing multiple major versions of a module in a build are all of the same form: increasingly complex logical abstractions. My concern is that enforcing it _universally_ might be prohibitively costly - especially for an ecosystem and community that has been operating without it for most of a decade.

If gps2 were to make SIV optional (it could work either way), it would greatly reduce the risk of mismanaging multi-version complexity. Modules that follow it would be a self-selecting group, and therefore more likely to be cognizant of pitfalls. On balance, it's likely preferable to retain SIV as a requirement, and instead relieve pressure by doing everything we can to help people stay in the v0 experimental range until they're really ready to commit to a v1 promise. That will minimize the need to have a v2 in the first place. 

The fourth post in this series is focused exclusively on SIV, and will explore these issues in detail.

### Implicit Compatibility

Almost all other dependency management-type tools rely on the user to specify compatibility ranges, often relying on unary operators like `^` as a shorthand for "anything at or above the given version, up to the next major version." vgo, however, internalizes the idea of compatibility ranges, so that people needn't declare it.

This is hugely powerful, for the simple reason that asking humans to predict the future is generally a bad idea.

Specifying any kind of semantic version compatibility range for a dependency is necessarily a speculative act. Any range statement will include at least some numbers that do correspond to actual versions:

- `>=1.2.0, <2.0.0`: does `1.3.10` exist? `1.3.11`? `1.99999.99999`?
- `>=1.2.0, <1.2.0`: does `1.2.10` exist?
- `1.1.0 - 1.1.1`: does `1.1.1-alpha1` exist? `1.1.1-rc42`?

_Note: many languages exclude prerelease versions from ranges, which would eliminate the third case. That's fine - it doesn't change the point._

When tools provide this kind of expressiveness to package authors, they're asking people to predict what future (in)compatibilities will exist. Humans are notoriously bad at predicting the future, and this case is no exception.

Instead, MVS hardcodes a mostly-semver assumption of backwards compatibility directly into the algorithm. That's far from perfect, of course, but being that we'd all at least like compatibility to be the norm, it's preferable to assume it. Then, within that implicit context of compatibility, the other statements that the user can make - `require`, `replace`, `exclude` - all target _individual_ versions. Consequently, we're no longer asking the user to make a prediction about the future, but a statement about the observable present.

Now, my central disagreement with Russ is over whether it should be possible for a module to specify an incompatibility with one of its dependencies, _and_ have that rule be globally respected by the tool (unlike `exclude`). But whereas gps allowed all manner of arbitrary constraints for making such declarations, gps2 incompatibility declarations would target only a single version. That means we're no longer asking people to predict the future; it also has some useful mathematical properties, though we'll explore those in the final post.

(Note: in our individual discussions, Russ has agreed to the idea of a service which would be a record of such single-version incompatibility statements. However, trying to abide by them during version selection would break MVS' invariants; he plans to treat them as warnings instead.)

## MVS: A Category Error

The above features are mostly excellent, and I'm sure Go developers are excited to get at them. Unfortunately, those features are all wrapped around MVS, which is an algorithm for solving a math problem, not a community problem.

But let's start at the basics. If there are two algorithms that satisfy the same requirements, and only one is NP-complete, you pick the other one. That's axiomatic. Moreover, if you have only an NP-complete algorithm for a particular problem, finding a less complex alternative that does the same job is an electrifying discovery. When such an alternative algorithm is proposed, however, the inevitable question to be answered is whether it _actually does_ meet the original requirements.

One way of thinking about this is through the framework of incidental vs. essential complexity, introduced by Fred Brooks in his famous paper, [No Silver Bullet](https://en.wikipedia.org/wiki/No_Silver_Bullet). He makes the distinction between complexity that has crept into software _incidentally_ and may be safely eliminated, versus complexity that is _essential_ to the problem at hand. 

Reading the vgo materials suggests that Russ believes the SAT-entailing aspects of current language dependency managers fall into the "incidental complexity" category. That is, they're problems we've essentially created for ourselves, and if we'd just trim the fat - a la MVS - then everyone would be unequivocally better off.

But, in avoiding SAT, MVS also cuts out some of the complexities that I believe are essential to the domain. Being essential, the problems don't go away when MVS ignores them. Instead, they're redistributed into other, often less obvious places. If reading the vgo blog posts gave you a general sense of unease that you couldn't put your finger on, that might've been you intuitively sensing some of these redistributions.

Now, having pushed out MVS's rules to their logical conclusions, I've seen where much of the displaced complexity lands - and I believe that the cure is worse than the disease. That is, MVS will cause more harm than arises from the NP-complete problems Russ designed it to circumvent.

There are six essential issues that lead me to this conclusion. We'll explore each of them, and more, over the course of this series:

- MVS has almost all the failure modes of a dep-style system, and some additional ones. Failures manifest as false positives. The only failure mode MVS lacks is pathological SAT solving; a later post will cover approaches to mitigating realistic SAT risks.
- When incompatibilities with new versions of your dependencies arise, MVS affords you only extreme options:
  - Refactor - optimal if you can do it, but may be prohibitively difficult, at least in the short term.
  - Lobby for change - maybe it works, maybe it doesn't; if it does, it's usually because the "incompatibility" was actually a bug.
  - Fork - this is a nuclear option, and always will be; it carries maintenance burdens for you and creates difficult-to-trace duplication within the module ecosystem, along with other, less obvious costs.
  - Ignore it - works fine for you, but creates a time bomb for others. By MVS' rules, it's antisocial community behavior.
- Compatibility is a hopelessly messy concept. API-level compatibility - aka, the [Go 1 compatibility promise](https://golang.org/doc/go1compat) - is useful, but covers only the API, not behavior. Below the API, the only possible general definition of compatibility is "change nothing," which is unhelpful. Absent clear, general rules we can mutually agree upon, "compatibility" devolves into a question of "who has the power." That, combined with a tool-established obligation to follow updates, is a recipe for toxic community interactions.
- SIV, by virtue of allowing pseudo-duplication of packages, will make more prominent a currently-nascent class of global state-driven complex runtime failures. These problems are far afield from Gophers' present-day thinking. Mandating SIV, as MVS entails, could elevate these effects from mild and infrequent to significant and pervasive.
- By blindly assuming compatibility, even in the experimental v0 range, MVS creates a hostile environment for experimentation. We should expect that will lead to v1 releases being rolled before authors are truly comfortable with their promises, which will not only undermine the commonly-held meaning of v1, but exacerbate the aforementioned problems with mandated SIV.
- The semantics of `require` compact minimum version with current version together, almost necessarily resulting in the loss of crucial information about what the "true minimum" for a given dependency may be. The loss of this information ultimately renders vgo unsuitable as an intermediate layer on which community tooling might improve.

There are other problems, but these are the foundational issues that cannot be fixed under MVS. Certainly, this is worlds away from "mostly don't pay attention to versioning," as [the first vgo blog post suggested](https://research.swtch.com/vgo-intro).

Still, some of these are pretty low-level concerns, and even if they don't sound particularly great, I don't imagine it's immediately obvious how they lead to a conclusion that a wholesale rejection of MVS is necessary. To help establish that context, and frame the detailed discussions in this series' later posts, let's step back from the trees for a look at the forest.

### The Forest of Risk

i've indicated that MVS is not fit for purpose, but have not been explicit about what, exactly, the purpose _is_ for tools in this domain. It's a tricky question, as there are a lot of overlapping, often competing goals that aren't readily separable. 

I spent a fair bit of time defining the goal in my package management essay from 2016. Early on, I gave a more mechanically-oriented description by [breaking the purpose down into steps](https://medium.com/@sdboyer/so-you-want-to-write-a-package-manager-4ae9c17d9527#8124):

> 1. divine, from the myriad possible shapes of and disorder around real software in development, the set of immediate dependencies the developer **intends** to rely on, then
> 2. transform that intention into a precise, recursively-explored list of source code dependencies, such that anyone — the developer, a different developer, a build system, a user — can
> 3. create/**reproduce** the dependency source tree from that list, thereby
> 4. creating an isolated, self-contained artifact of project + dependencies that can be input to a compiler/interpreter.

One could quibble a bit about whether MVS meets some of these, but I'd say it essentially passes this bar. So the problems with MVS aren't evident at the level of basic automation and structure. You have to dig further, until you reach the more foundational idea of [risk management](https://medium.com/@sdboyer/so-you-want-to-write-a-package-manager-4ae9c17d9527#4e66):

> The themes here are time, risk, and uncertainty. When developing software, there are unknowns in every direction; time constraints dictate that you can’t explore everything, and exploring the *wrong* thing can hurt, or even sink, your project. Some uncertainties may be heightened or lessened on some projects, but we *cannot* make them disappear. Ever. They are natural constraints.

The deeper purpose of tools in this domain is to mitigate the various risks of relying on other peoples' code. MVS focuses exclusively on eliminating one class of risk within that set - pathological SAT solving arising from arbitrary constraints - because, as far as I can tell, it's the one risk that's well-defined enough to be in a known complexity class. But it does so by increasing other risks, sometimes drastically - and without a critical examination of what factors lead to unmanageable SAT issues manifesting in practice.

To illustrate what's being missed, let's look at "risk management as a design goal for dependency management" through two different lenses: distributed systems and economics.

#### Dependency Management as a Distributed System

The asceticism of vgo's design will be familiar to any moderately experienced Go developer. MVS combines strategically-applied brittleness (e.g., the compiler barfs on unused imports), with leaving complex problems to humans (e.g., generics). When I see people reacting to the vgo proposal by saying that it “feels very Go-ish,” I think it's these underlying patterns they’re recognizing.

But general principles are not necessarily applicable in every situation. Both “brittleness is instructive” and “complexity is for humans” need very tight feedback loops to work well, which largely limits their applicability to problems that are solved within a single mind. When those feedback loops are stretched out over time and multiple people, they become drastically less effective. Dependency management is spread across both.

If we think of dependency management as a problem spread across multiple people, then it’s natural to wonder, “might this be a form of a distributed systems problem?” I believe it is, and that it’s useful to adapt the [Fallacies of Distributed Computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing) to dependency management in order to relate some of the harmful oversimplifications in MVS to a problem space that folks are already familiar with.

>  The network is reliable → **Human communication is reliable**

MVS hardcodes the assumption of backwards compatibility. As I'll detail in a later post, compatibility is an empty idea unless the author describes the intended behavior of their code - that is, some kind of specification.

In Go, "specification" really just means godoc comments, and to some extent, corresponding tests. Trying to glean useful information from these can be a crapshoot; even well-written documentation often leaves important information out. Moreover, such specifications generally only deal with current behavior - not possible future changes.

These are lossy mediums. Murkiness, confusion, and disagreement are the norm. That's an unstable foundation, and brittle systems like MVS need stability to work well.

> Latency is zero → **Incompatibility remediation is obvious and trivial**

When apparent incompatibilities arise, it can take considerable time to even sort out what's happened - where is the problem? is it a bug? an intentional change? Will the upstream maintainer fix/revert the change, or keep it?

Open source software communities run on async. Everyone has different schedules, priorities, and motivations. As such, there's no guarantee how, when or even if issues will resolve. But MVS introduces new synchronous blocking problems: if we have invested the time in figuring out a problem with a dependency, our ability to create a release cannot be blocked on some potentially costly refactor, whether of our own code or in one of our other dependencies. (The post on failure modes will illustrate how this can occur.)

> The network is homogeneous → **Module names/import paths have consistent meanings over time**

Different people will have different perceptions of what the intended behavior is for a particular module's code. Usage will diverge as a result. This is the obverse of Hyrum's Law, and it stands in direct contrast to vgo's platonic ideal of an ecosystem where import names having consistent meanings across releases.

As we will see in the compatibility post, the only truly consistent aspect of a name's meaning over time is: "I am the maintainer, I have the power, so it means what I think it means."

> Transport cost is zero → **Open source labor is free and plentiful**

MVS' solution to essentially every possible incompatibility problem is, "someone needs to write more/better code." But "change the code" has always been an option - the generally preferable one in existing language package managers, and the only one available before modern language package management existed. Which is to say, MVS' solution isn't new, or really even a solution - just an assertion that, given compatibility rules and a coherent renaming strategy like SIV, we should discard the entire class of software that is language package management. (Russ has pointed out to me that the vgo blogs refer only to versioning, not to a "manager.")

Compatibility rules and SIV certainly help. But they miss an important part of the picture. While asking maintainers to [explicitly define compatibility ranges](#implicit-compatibility) has significant problems, it does allow maintainers to set boundaries on the work they are willing, or able, to do.

Before language package managers and semantic versioning began automating the dependency management process, the very awkwardness of working with dependencies insulated maintainers from undue pressure. Now, however, as dependency management tools streamline and automate updating, human labor is increasingly the bottleneck, and allowing maintainers to set such boundaries becomes a matter of self-care: "Our project depends on `X@v1.5.0` right now, but it doesn't work with `X@v1.7.0` or newer. We want to be good citizens and adapt, but we just don't have the bandwidth right now."

MVS, however, throws the baby out with the bathwater by stripping this control from authors. As a result, even if compatibility rules and SIV are usually sufficient to deliver good results (at best, a tenuous proposition), for those times when it's not, the rules of the system establish norms that expect maintainers to put the state of the ecosystem above their own/their organization's priorities and needs.

Now, FLOSS licenses mean that maintainers aren't actually _obligated_ to do anything. But this is about the norms and expectations communities establish - not law. And it's a red flag when those contravene legal protections.

There's already a tendency in open source to dogpile maintainers when compatibility promises are broken. But MVS enshrines this natural tendency as a norm. That's coercive, verging on exploitative, and antithetical to the very notion of a voluntary community.

#### Dependency Management as an Economic System

The vgo proposal [gives two examples](https://github.com/golang/proposal/blob/master/design/24301-versioned-go.md#build-control) of overactive constraint problems: one artificial example involving primes and evens, and another from Kubernetes, where Kubernetes' use of Godep [prevented one user of an unnamed tool](https://github.com/kubernetes/client-go/issues/325) from using a newer version of a YAML library. (Yes, dep is absent from this - the example is a strawman.) There is, of course, no example of a constraint being helpful by eliminating versions that truly do _not_ work.

There's an unspoken premise behind these choices of examples: if the toolchain provides us with a sharp instrument - in this case, the ability to declare "`A@x` doesn't work with `B@y`" - then we will necessarily stab each other with it. Therefore, vgo should take that ability away from us; the inevitability of negative outcomes outweighs the potential upsides of automation and shared knowledge coming from helpful constraints.

But there's an unexamined premise here: _why_ are the negative outcomes inevitable?

Now, I understand the value of cutting out unnecessary degrees of freedom. I get why defensive coding is important. And I've seen people wedge themselves into some truly absurd spots with software I've written. But it's still lazy thinking to simply assume that users will necessarily fill up every nook and cranny of what a tool allows. We have to temper that tendency by thinking about _why_ a user might take actions with these kinds of negative consequences. In almost every case like this I've seen, it's because the user is trying to further one of their own goals, but that has unintended consequences for others.

There's a standard term in economics for idea: [externalities](https://en.wikipedia.org/wiki/Externality). An externality is a cost or benefit experienced by people who did not choose to incur those costs or benefits. A classic example of a negative externality would be secondhand smoke: when someone else makes the choice to smoke for their own purposes, it changes the immediate environment in a way that can harm me. It was not the smoker's intent to harm me - they only sought to meet their own needs - but it occurred nonetheless.

There are two basic approaches we can take to address externalities:

- Construct the system so that the person taking an action with potentially negative externalities (like declaring an incompatibility) has to pay some cost. This is known as "internalizing the cost."
- Isolate actions that the user takes for personal purposes from actions with potentially negative externalities. (Economists work on natural systems and don't usually have the luxury of constraining choice; we at least theoretically do, to the extent that the set of possible choices a user can make are determined by tool design.)

We'll look at these in greater detail in the final article of the series. But consider the latter technique in the context of the Kubernetes YAML example: the library is pinned to an old version because Kubernetes needs a reproducible build, and pinning is the only action Godep allows. But if Kubernetes were using a different system where they could have their reproducibility, without the negative externalities of pinning, they certainly would - _especially_ if the tool's design gave them that for free.

Carefully separating the levers has been a design ethos in the dependency management space for some time. We want the actions users take in pursuit of their own goals (e.g., updating a dependency) to have either no externalities, or overwhelmingly neutral-to-positive ones. For actions with a broader range of potential externalities (e.g., declaring an incompatibility), we want them to be performed solely _for_ those effects - they have no conflicting personal incentives.

This separation doesn't guarantee no one's utility will ever decrease. Nothing can. But, if individual goals can be decoupled from negative externalities (e.g., Kubernetes can have reproducibility without causing pinning), then it's feasible for a community to converge on best practices that will minimize cost and maximize benefit. For example, these might end up being reasonable guidelines on declaring incompatibilities:

- Declare incompatibility on a dependency when the same inputs return different outputs in a way that significantly impacts program behavior.
- Don't declare an incompatibility for performance regressions.

While gps allows a number of declarations that attach potentially harmful externalities to individual goals, gps2 could achieve much of the aforementioned separation. MVS, on the other hand, has just one bedrock directive: `require`. As a result, individuals' goals (e.g., updating a dependency) are unavoidably shot through with externalities. The next post, on failure modes, will illustrate exactly how that works.
