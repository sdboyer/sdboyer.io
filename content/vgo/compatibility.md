+++
title = "Compatibility"
linkTitle = "Compatibility"
date = 2018-07-12T09:12:18-04:00
keywords = ["golang", "go", "clapback", "dep", "dependency management", "package management", "compatibility", "semver"]
draft = true
+++

# Compatibility

Definitions of compatibility are absolutely essential to vgo, in ways both obvious and subtle. Almost everything else flows from these definitions, so they're a good place to start, if only to establish a good basis for discussion. Before we get to compatibility proper, though, we need to set some context by talking about the context in which compatibility is defined.




## Version Universes

A version universe is the set of versions that correspond to a given name - specifically, with vgo, a given module. To be absolutely clear about what that means, let's explore how these universes are derived from git.

We can imagine a git repository as having a number of different types of addressable identifiers:

![git-baseline-contents](https://www.dropbox.com/s/0qggz5kgw2fuvxa/git-baseline-contents.png?raw=1)

(Any real git repository is likely to have far more commits than just the two explicitly shown here - just assume they're there, but not shown.)

There is a relation between these lists: every tag or branch corresponds to exactly one revision, and revisions may have 0..N corresponding branches or tags at any given time.

The first thing that vgo does is convert revisions - in git's case, a 40 char hex-encoded SHA1 hash digest - into "pseudoversions", which are semver-shaped - a `v0.0.0` base number, with a semver-ish suffix that includes both a datestamp of the commit object, and a portion of the revision itself. This allows them to be sorted both with respect to each other (chronologically, by the datestamp) and with respect to actual semver tags (coming after all of them):

![rev-to-pseudo](https://www.dropbox.com/s/kk8gegrwjmqxq47/rev-to-pseudo.png?raw=1)

(Pseudoversion strings are long, and don't fit nicely in the diagram, so i've elided their inner bits. But let's imagine here that the `COMMIT_DATE` on `abcd123` was later than on `ef45678`, therefore it shows up higher in the list.)

Branches and non-semver tags are not directly supported by vgo, but they can be implicitly addressed through the pseudoversion corresponding to their underlying revision. Let's say that, right now, branch `master` points to commit `ef45678`, and tag `foo` points to `abcd123`:

![other-to-rev-to-pseudo](https://www.dropbox.com/s/l2mzulvsfxwjut9/other-to-rev-to-pseudo.png?raw=1)

This is all implicit, though. The only actual versions that MVS operates on are semver-shaped:

![semver-set](https://www.dropbox.com/s/29dp4hrqf746vav/semver-set.png?raw=1)

This is crucial for MVS, because it is predicated on the idea that we can arrange versions into a total order of compatibility - that is, that we can define a relation across the set of versions where each larger version has the property that it is compatible with the smaller version. Or, put simply: that newer versions are **backwards compatible** with older versions.

vgo adds a twist that's in keeping with the ideas of semver. Sort of. Instead of simply having the assorted major versions - across which it is acceptable to break compatibility - all exist in the same module's version universe, it partitions them into separate universes for each family:

![sep-univ](https://www.dropbox.com/s/fohph67ahigzxh4/sep-univ.png?raw=1)

Next, vgo descends from source metadata into the code itself, in search of a `go.mod` file. If none exists, it assumes the v0/v1 family; otherwise, it looks for a version family suffix in the final path position of the `module` declaration.

Inspecting the `go.mod` in this way tells vgo the version family to which a pseudoversion should be assigned:

![sep-univ-pseudo](https://www.dropbox.com/s/2b40lfpawxahxp0/sep-univ-pseudo.png?raw=1)

However, there's always the possibility that what `go.mod` says (if it exists at all) doesn't line up with the tag. Unfortunately, good citizens of the Go community that have been tagging their projects for a while now will bear the brunt of this pain: let's say that `v2.0.0` was released without a `go.mod` in it. vgo will not allow that version to be addressed directly:

![no-v2](https://www.dropbox.com/s/owaixyisx0qs0yn/no-v2.png?raw=1)

Although `v2.0.0` can still be addressed indirectly, via its corresponding pseudoversion:

![v2-pseudo](/Users/sdboyer/ws/sdboyer.io/static/vgo/v2-pseudo.png)

And this, finally, represents the version universes derived from a git repository. Within each module universe, vgo assumes backwards compatibility:

![backcompat](https://www.dropbox.com/s/z6o6xdta493gsqv/backcompat.png?raw=1)



_Note: vgo also allows multiple modules that are not at the root of the repository. That property is orthogonal to compatibility and major version families, however, so it's not shown here._

ADD ITEM DIFFERENTIATING A BUG FROM AN INCOMPAT



Now that we've got a clear picture of version universes and how they shape a compatibility space, we can properly dig in on compatibility itself.

Compatibility is the grand, unspoken assumption of vgo, and MVS. It more or less takes for granted that it's feasible for a huge, mostly-disconnected group of people to form a software ecosystem that is compatible.

MVS is driven entirely by the assumption that it is operating on backwards-compatible version universes, and it precludes making an incompatibility declaration (e.g., "`A@v1.5.0` doesn't work with my module") that has force beyond the module where it is declared. In other words, if backwards compatibility assumptions don't hold within a version universe, then MVS will produce a broken depgraph sooner or later, and the user has to remediate.

This can be directly rephrased in more intuitive, social terms: if you determine that your module doesn't work with some version of one of your dependencies, you can record that fact privately, but you are unable to inform others about it in a productive way.

Blindly assuming compatibility is obviously problematic. MVS aims to claw its way back into the realm of feasibility on the basis of two arguments:

* When compatibility invariants do not hold, changing code and releasing an update is both reasonable and feasible.
* The relative simplicity of the algorithm will make it easy for users to remediate problems, and provide them more confidence in the system overall.



"Assume that software is generally compatible, and that the cases where it's not are trivial and uninteresting."



We'll explore the costs of breakage later. For now, we'll look in greater detail at the idea of compatibility itself, and the circumstances under which we should expect it to be insufficient - that is, when we should expect breakages to occur. It matters a great deal to establish these insufficiencies, as the vgo blog posts are at best taciturn on this front, and we cannot truly  consider the costs of vgo without a proper accounting of the ways in which its assumptions can break.

It's much easier for this to seem plausible when handwaving over the nature of incompatibility.



Everyone says, "semver is a social concept," but i think they mostly just use that to mean, "it's not precise. ¯\\\_(ツ)_/¯". But it IS precise - it tells us whose fault it is.

## API Compatibility

As the vgo blog posts outline, the idea of compatibility here should be a familiar one to Go developers: they are, in essence, [the Go 1 compatibility promise](https://golang.org/doc/go1compat#expectations) (G1P). These rules can generally be characterized as "accretion-only": add as many new exported identifiers as you want, but never remove, rename, or change the type of an exported identifier (package-level variable, function, method, or type) in any non-`internal` package.

Despite these being familiar rules in the Go world, they have their complexities. G1P gets the hardest part out of the way up front: it explicitly does _not_ guarantee that all valid Go programs will compile against newer versions of the standard library. For example, adding a new method, `Foo()`, to a non-interface type in stdlib is considered a compatible, accretion-style change, but it can cause a build breakage downstream if that type was embedded into a struct alongside another type that already had a `Foo()` method.

Axel Wagner wrote [a great article](https://blog.merovius.de/2015/07/29/backwards-compatibility-in-go.html) about API compatibility in Go in which he came to the conclusion that, if the compatibility standard is "changes where it is _impossible_ to break someone else's build," then there are almost no truly backwards compatible API changes. i think he's right. But such a definition - let's call it "absolute build safety" (ABS) - is so strict that it would prevent us from changing our software in a great many useful ways. That's a non-starter, so we are forced to accept some lower threshold when it comes to compatibility - that is, G1P.

Go often sacrifices purity for practicality, and i'd say this particular compromise has worked out pretty well, at least in the standard library. But it also leaves behind a gaping teleological hole: while the utility of a version universe that observes ABS is clear (compilation cannot break), the utility of a G1P-observant universe is not. G1P doesn't guarantee compilation, let alone correctness. Mechanically, its only definition is tautological: "a version universe that follows the backwards compatibility rules is backwards compatible." Such closed-loop tautologies are not usually terribly useful, and yet, i doubt anyone could credibly argue that G1P has not been a profoundly useful concept in practice.

This disconnect arises, i believe, from the assumption that compatibility's utility is mechanical. It's an easy mistake to make, as compatibility rules are at their best when they are sufficiently precisely specified that a machine can enforce them. The true utility of compatibility rules, however, is not mechanical, but social: "if changing versions doesn't work, **compatibility rules tell us who's at fault.**" This is enormously practical: when something goes wrong, it suggests _what to do next_: should we make the fix on our side, or should we go and tell someone else about it so they can make a fix?

Once we reconceive of compatibility as a social system centered around assigning blame, it's clear that "backwards compatibility" can't be the only compatibility game in town, as it is concerned only with the responsibilities of one party - the code producer. We also need to talk about the code _consumer_.

This separation of producer and consumer is also entailed by lowering the threshold of compatibility from ABS to G1P: whereas ABS describes a one-party responsibility, G1P spreads that responsibility across producer and consumer. That is, if module _producers_ are empowered to make changes that are admissible by G1P, but not ABS, then it entails at least some responsibility on the part of module _consumers_ to "behave well" - that is, not write certain programs that, despite being valid Go code, may not tolerate updates to their dependencies.

From here on out, i'm going to refer to producer-side responsibilities as "backwards compatibility", and the consumer-side responsibilities as "cross-compatibility." We can represent them like this:

![cross-compat](https://www.dropbox.com/s/26io9w1a3yz322c/cross-compat.png?raw=1)

This is really just an illustration of a vgo `require` statement's semantics:

```go
module mod2

require "mod1" v1.1.0
```

Or, equivalently, a `version` constraint in a `Gopkg.toml` :

```toml
[[constraint]]
  name = "mod1"
  version = "1.1.0"
```

_(This kind of constraint is [the strongly preferred path in dep](https://golang.github.io/dep/docs/Gopkg.toml.html#version-rules) because it helps avoid conflicts - the same motivating goal that vgo takes to an extreme by **only** permitting minimum version declarations.)_

Both of these declarations mean the same thing: `mod2@v1.0.0` is compatible with `mod1@v1.1.0`, and all later versions, both existing and potential, in `mod1`'s `v1` family. Or, in human terms: if `mod2@v1.0.0` turns out not to work with any successor version of `mod1@v1.0.0`, and that later version followed backwards compatibility rules, then the author of `mod2`'s is to blame.

i hope that phrasing raises a yellow flag, if not a red one. i could have been more diplomatic by describing it as one or the other of the authors' "responsibility," or by just referring to code, pointing at one or the other module as "where the fix should be made." But i think that risks erasing what the experience of being a human working a software ecosystem governed by MVS will be.

It's not that we shouldn't have frank discussions in which we identify problems and mistakes. That's always  going to happen. Rather, it's because MVS must strip authors of the tool they need (globally-respected excludes) to prevent contagion failure. Without those controls, if a module author cares about not retransmitting a problem to their dependers, it  _requires_ that blame be assigned, and pressure applied until a new release is made. Software communities are plenty thunderdomish without algorithmic encouragement.

Of course, there's another complication camped out in here, as well: API-level compatibility problems are trivially easy to identify. Behavioral compatibility, however, is often much less clear. Let's look at that next.









It's important that we separate cross-compatibility from backwards compatibility when evaluating vgo. Backwards compatibility is a well-established concept in the Go world, thanks largely to G1P, and i think it's fair to say that the Go community has been cognizant of it in the ecosystem for years. Cross-compatibility, on the other hand, doesn't exist as a concept independent of backwards compatibility in the current discourse. It needs to, though, because 




In some cases, cross-compatibility can be fulfilled through choices that are more or less tradeoff-free: e.g., "when populating fields in a struct defined in another module, always use explicit field names." Other times, as in the case of the double-type embedding example above, there may not be an equivalent alternative approach. So, if the consumer needs to embed two types to fulfill their design, then they have no choice but to do so, and in so doing break a cross-compatibility rule that introduces the possibility for future breakages.



The first thing we need to dispense with is any kind of notion that, once two people are in the room, there will be an objective answer to what "correct" code is. At most, what we're actually talking about is a power dynamic: there is one person (or group), the authors, who are empowered to decide what "correct" means in the context of their namespace. The other person or group(s) might disagree, and they can argue their case, but at the end of the day, it doesn't really matter, because they don't hold the keys.

Please note that i'm not necessarily arguing that that's the way that things _should_ be. But it's a surprisingly useful affordance to make, because when we talk about "compatibility", what we're really talking about is a very handwavy form of software correctness. And being that correctness is really just a measure of software's adherence to some specification, it's already a bit handwavy, because conversations about compatibility 



CORRECTNESS IS A MEASURE OF ONE PROGRAM; COMPATIBILITY IS THE NATURE OF CHANGES MADE OVER TIME TO A SPECIFICATION, AND ITS IMPLEMENTATION?

it's not a coincidence that Hickey gave his talk in the context of discussing spec, the Clojure specification meta-system. 

## Behavioral Compatibility

If we're going to talk about behavioral compatibility, then we have to start with an impossibility assertion: there is not, and can never be, a general definition of behavioral compatibility.

It's possible to talk about behavioral _equivalence_. We could see that as the essential concern of formal methods: create a precise specification, and then prove that programs conform to it. Proving that two different programs conform to one specification is the same as proving behavioral equivalence between the programs - or at least, equivalence within the scope of that spec.

But we can't conflate equivalence and compatibility. In the same way that ABS is a stricter superset of G1P, behavioral equivalence is a stricter superset of behavioral compatibility. And if ABS is inflexible, then behavioral equivalence is downright absurdly brittle; completely new behavioral paths may be created, but for any existing logic path, behavioral equivalence would forbid even the most trivial changes. So what, then, is the behavioral-level equivalent of G1P?

That's the thing that can't exist. It's easy enough to say why in theoretical terms:

* It's _impossible_ because Go is Turing complete, so the behavior of Go programs is undecidable; [Go's type system](https://golang.org/ref/spec#Types), by contrast, is decidable.
* Even if we ignore decidability, it's still _intractable_. In the type system, the set of primitives - e.g., structs, interfaces - over which compatibility definitions must to be made is both finite and small, and therefore reasonable for developers to hold in our heads.
* because there is no separable set of "base" behaviors that is enumerable 

Contrast this to behaviorGo is Turing complete, so its behavior is, in the general case, undecidable



In the type system:

* "Accretion" is essentially "adding elements to sets." If we think of Go's relevant major constructs in set-theoretic terms, that means:

  * **Modules** are named sets of packages. New packages may be added. Existing packages may be modified in accordance with their accretion rules.
  * **Packages** are sets of exported identifiers. New identifiers may be added. None of [the four package-level object kind]() (variables, constants, functions, and types) may be changed.

  * **Structs** are a named set of fields. New fields may be added.
  * **Method sets** are the set of methods attached to a type. New methods may be added.
  * (Prominently absent: interface method sets and function signature sets)


These definitions are nice, intuitive, and - fun fact - utterly impossible to formulate for behavior. The behavioral level has no analogous universal system of sets against which to define accretion, because there are no universal classes of behavior.

There are universal _operations_ - e.g. addition, subtraction, reduction, mapping. Fun exercise: try to compose semantics from operations, see how many steps it takes before you've created a Lisp, then [realize you still have a semantics problem](https://www.codeproject.com/Articles/1186940/Lisps-Mysterious-Tuple-Problem).

There are universal _representations_ - e.g. state machines, automata. The TLA+ proof system, for example, is essentially a language for exhaustively defining state machines.

But there is no universal _semantics_ of programs. If there were, then it could be used as a specification against which all programs could be proven correct. 

i realize this all is terribly abstract. But it's important, because in the vgo posts, Russ indicates that it was an original design goal of Go that "import path names mean the same thing over time." It's a seductively powerful idea, and certainly a world that we would love to live in. But it cannot have a precise mechanical definition, because we cannot create a general definition of behavioral compatibility analogous to the API-level G1P.

Without a precise mechanical definition, we can't reasonably categorize changes in a way that makes breakages the fault of either producer or consumer. In the absence of clear rules, power ultimately prevails: whatever choice the producer made is right. If i'm on the consuming end of such a change, and i want to be a good community citizen, then i must either submit to the producer's authority and do the work to compensate for the change, or fork and take on the responsibility for maintaining code that i likely do not fully understand. It's not NP-complete, but that still sounds plenty hellish to me.







The map iteration order problem is a great example of this. They can say, "don't rely on unspecified behavior," but they can't say anything about whether or not that specification will change. We all just implicitly agree, on some unexpressed level, that it would insanely harmful to change the specification, and they take the extra step of codifying it at a behavioral level so that the object people are actually interacting with and observing is, in the context of a developer's normal day-to-day environment, consistent with the spec. 













In fact, behavioral compatibility-as-equivalence is an explicit _anti-_goal of software design. We create interfaces in the first place to hide away the details of implementation.





The G1P dominates the dialogue around compatibility in the Go community, to the point where i think it's begun to obscure this basic reality of software. We are collectively reduced to describing sub-API problems as "subtle," when in fact the entire We end up assuming that just because two versions of a package are API-compatible, they're somehow also behaviorally compatible. In practice, it's usually more or less true, but because there is no known way of defining behavioral compatibility, we can't actually have this as a guarantee.

In much the same way that ABS is able to describe the subset of API changes that _could not possibly result in breakage_, it's possible to decide exact behavioral _equivalence_. If you write a software specification in TLA+, check two programs against that specification, then they are behaviorally equivalent. But, as with ABS, this is a brittle, unuseful guarantee for "safely" evolving the software. Almost any change to the verified behavior will cause the checker to fail - and that means the specification will have to be updated. And therein lies the rub: while we can check that a program conforms to a spec, there's no known general way of guaranteeing that one _specification_ is compatible with another.



But that doesn't mean we can't say anything useful about behavioral compatibility. Because we already know that we're looking for a socially-applicable ruleset, not a mechanically-applicable one, what we really need is a way of _classifying_ changes, so that responsibility for breakages arising from them can reasonably be assigned. In a loose sense, 





We'll get to some examples from real Go projects in a bit, but to make these concepts as useful as possible, let's use an artificial example. Imagine we have this:

```go
// Sq returns the input multiplied by itself.
func Sq(x uint) uint {
  return x * x * x
}
```

The type system guarantees that this function will never return a `string`. That's very nice to have, but it tells us nothing about semantics that are not directly represented. Such is the nature of type systems; to use [Chris Smith 's formulation](https://cdsmith.wordpress.com/2011/01/09/an-old-article-i-wrote/):

> Proof establishes lower bounds on correctness.

"Lower bound" means that the type system provides a guarantee that certain classes of errors cannot exist (no `string` return here) - but nothing more. And indeed, the function's behavior is to cube the input, even though the docs and name indicate it should be squared. Also, there's a weird, unspecified thing - integer overflow behavior.

* **Impossible**: return anything other than a `uint`
* **Specified**: return the square of the input
* **Unspecified**: integer overflow behavior






It's normal for "can't have undesirable properties" and "has desirable properties" to sound the same - it's the [fallacy of the excluded middle](https://en.wikipedia.org/w/index.php?title=Fallacy_of_the_excluded_middle). In this case, though, there's a third option: unspecified properties. The only thing we have here that could really count as a specification is the function godoc and the name of the func itself - pretty realistic - so let's consider those the spec. We could classify our example this way:

* **Undesirable:** returning a `string`
* **Unspecified:** behavior on integer overflow
* **Desirable:** returning the square of the input

[The Go spec defines integer overflow behavior](https://golang.org/ref/spec#Integer_overflow), but that's not in the specification _here_. Now, it's probably reasonable to infer that, this being a Go function, it will inherit Go's overflow semantics - certainly that's the case in the example function body right now. But "behavior right now" and "specified behavior" are not the same; they're not even in the same category, even though they're clearly related.

Let's visualize these interrelated ideas, step-by-step, beginning with the set of all conceivable behaviors for `Sq()`:

![behave-all](https://www.dropbox.com/s/rra4yfwlhg33lxm/behave-all.png?raw=1)



Next, let's incorporate the type system. It divides the set of conceivable `Sq()` behaviors into those that are possible (an `int` is received, and an `int` is returned), and those that are impossible (anything else):

![behave-poss-imposs](https://www.dropbox.com/s/08ksch21vpob4z2/behave-poss-imposs.png?raw=1)

Because `Sq()` has a simple `uint` input, we can reasonably enumerate the size of the possible space: on a 64-bit machine, there are `1<<64` possible inputs - all the possible values of a 64-bit `uint`.

Next, we want to represent the role of the specification - the function doc. This subdivides the set of possible behaviors into specified and unspecified behaviors:

![behave-spec-unspec](https://www.dropbox.com/s/hkv6knoi4m62l6i/behave-spec-unspec.png?raw=1)

The specified area corresponds to the input values that do not result in integer overflow, and the unspecified area to values that do.

The relative sizes of these circles is not meaningful. If it were, then the "specified" area would be invisibly tiny: on a 64-bit machine, it encompasses only `0` to `1<<32-1`, whereas the unspecified area encompasses all values from `1<<32` to `1<<64-1`. This is a deceptively important point, as it is not uncommon for a large swathe of possible function behaviors to be either poorly or entirely unspecified. This example illustrates it nicely; most people don't intuitively think about large numbers, so even if they know that overflow behavior here is undefined, they might be inclined to dismiss it as an edge case. In fact, 99.999999977% of type-valid inputs to `Sq()` have undefined results.

Another thing to note: "specified" is orthogonal to "error." It's legitimate, even preferred, to specify that certain inputs result in an error; it's also perfectly plausible that some non-error inputs might be unspecified. Specification is exclusively a question of whether there exists a normative statement about how a program _should_ behave for some particular input.

Now, let's tease this apart just a tiny bit more by separating the idea of specification into the two ways in which we actually tend to do this in Go: via documentation, and via tests. (Yep, tests are a form of specification!)

![behave-correct](https://www.dropbox.com/s/b4lh5kjqtrzdl7d/behave-correct.png?raw=1)

This is playing a little loose with the term "correct" here - if tests are a mechanically executable form of specification, and the tests are bug-free, then the entire area covered by tests could be considered correct. Reasonable people could disagree on this classification, but we depict them separately here for reasons that we'll return to later in this section.

To better understand what we're actually describing here, consider this formulation from Chris Smith:

> Proof establishes lower bounds on correctness.
>
> Testing establishes upper bounds on correctness.

These diagrams illustrate both the upper and lower bounds. The lower bound, established by proof - that is, Go's type system - is the set of possible type-valid `Sq()` behaviors.

The upper bound established by testing is almost necessarily smaller than the possible space established by the lower bound. To truly cover everything in `Sq()` would require this test:

```go
func TestSq(t *testing.T) {
	// Assume 64 bit uints
	var max uint = 1 << 64 - 1
	for i := uint(0); i <= max; i++ {
		if i * i != Sq(i) {
			t.Fatal("Expected %v for input %v, got %v", i * i, i, Sq(i))
		}
	}
}
```

Which is unlikely to finish in a human lifespan. And there's no shortcuts here - if it's not mechanically checked, it's not correct. That's horribly pedantic in this situation, but that's only because `Sq()` is an extremely simplified example. For example, [fuzzers](https://en.wikipedia.org/wiki/Fuzzing) exist largely because it's infeasible to explore the essentially unbounded possibility space of a `string`.

### Enter, Humans

These diagrams describe a mostly-objective reality of software (mostly, because documentation can be subject to interpretation). But our present problem is compatibility, and we know that's a human issue - a question of where to place the blame. Whatever the underlying reality of the software might be, what actually matters is what the producers and consumers of software _think_ about it.

It's reasonable to assume, for example, that people producing software have a good understanding of it. But that understanding isn't perfect. For any nontrivial software, there's always going to be some portion of software's behavior that its own authors don't fully comprehend - and that area grows considerably if there are multiple authors.

These blind spots tend to be in the "unspecified" area, but that's not a guarantee. It's normal for folks to not model a problem in quite the right way, particularly when initially creating an API.



When these issues come to the author's attention, they'll make a change (eventually) to correct them. In a simplistic sense, if the fix requires changing a specification, then it's backwards-incompatible, and the resulting breakages are the producer's fault. If it's a change to unspecified behavior - perhaps moving some behavior from unspecified into specified, as was the case in [this recent stdlib issue]() - then any resulting breakages were the fault of the consumer: they shouldn't have been relying on unspecified behavior.

But here's the rub. First, 

That'd be nice, except 



Now, we could go through and categorize all the individual 



 otherwise, it should be a No matter which area the uncertainty falls into, though, 



Especially when the 



This is pretty similar to the 



There's an entire movement out there that believes it's _wrong_ to have comments on your code. "It should be self-evident!"



The perspective that a consumer of code has on all this is quite different. Especially when first incorporating a dependency, consumers generally have a pretty limited understanding of how it works. By and large, people go hunting around for libraries that seem like they might solve their problem

Most importantly, what folks base their work on 

Most importantly, the thing that consumers of code can't see is how the producer _thinks_ about their software. 

"Want to respect" vs. "forced to obey"



As a general rule, package management is a class of software with a very _low_ threshold for inspiring people to change behavior. 

As a consumer of code, _all_ of this is opaque. There's a bit of the specification that's visible, but 





  are all well and good when we discuss it as a property of programs, but humans are what matter in the 





### Example: gRPC FailFast

gRPC has a ["FailFast" option](https://godoc.org/google.golang.org/grpc#FailFast) that determines the behavior of an RPC call initiated while the underlying gRPC-managed connection is in a failure state. When fail-fast is enabled, such a call will immediately return an error; when FailFast is disabled, the call will block.

The gRPC specification [indicates](https://github.com/grpc/grpc/blob/master/doc/wait-for-ready.md) that fail-fast (though preferred term is now "wait-for-ready," which is the negation of fail-fast) should be enabled by default. However, the initial grpc-go implementation did it the opposite way - fail-fast was disabled by default. [This pull request](https://github.com/grpc/grpc-go/pull/738) inverted introduced the explicit option to control this behavior, but it also inverted the default to be in line with the upstream specification, resulting in a backwards-incompatible semantics change.

Behavior like this is subtle, and there's no good reason to believe that it would be caught by a grpc-go depender's tests. The conditions necessary for it to occur may very well may only manifest in production at higher traffic volumes. And, because the change involves two of the more insidious problems in software - error handling and concurrency control -  it's not hard to imagine such a change causing an outage.

It's trivially easy to return to the original behavior - a single extra function call - but that's only easy _if_ you know the change has occurred. If your library is interacting with gRPC directly, then it's reasonable to believe that you might 



 trivially easy to deal with: if you actually want wait-for-ready instead of fail-fast, it's as simple as adding a parameter to the call initiating the gRPC connection. Just because making the change is easy, however, doesn't mean that discovering the problem is. 



 a failure mode and changes the threading profile of a program in outages where  where, because it involves adding a new failure mode, could result in a project 



 if If a consumer of grpc-go wasn't following the project closely, then 



 governs the behavior of RPC calls that  governs the behavior of RPC calls that are made 

It's not specific to Go - bu gRPC specification

At the same time, how should this be classified? It was inconsistent with the upstream spec, 





These diagrams also help illustrate one of the main challenges with proof and formal verification in general: a spec that is insufficiently precise and comprehensive can miss certain aspects of a program - so, things in "unspecified" - that recontextualizes the "correct" code, in such a way that it no longer has the correctness properties that the spec's author actually intended. This is not an abstruse academic problem: [it is exactly what brought us the KRACK vulnerability in WPA2 last year](https://galois.com/blog/2017/10/formal-methods-krack-vulnerability/).

OK, that's the picture on the proof side. Now, we need to pull a second tidbit from Smith's article to complement the earlier one:

> Testing establishes upper bounds on correctness.





This discussion of specifications, proof, and correctness - all things pretty far from most Go programmers' minds, i think - matter is because they provide the clearest framework for discussing the notion of _behavioral_ compatibility over time, rather than simply API compatibility (the G1P). And a framework for this is important; in the Go community, i think we've gotten so accustomed to treating the G1P as a sufficient guarantee of compatibility that it's easy to forget that API-level considerations are a comically small sideshow to the main, complex event.





The longer the debate goes on, the more dependencies will adapt to the new version, creating a split-brain in the world - you can't go back without forcing them to go back, too. And, because they are all theoretically incentivized to update quickly in order to make a nicer universe for everyone else, we should 









## Being a Human Under MVS



As someone who spent most of two years in the dark as to what changes that Russ might be willing to accept in the toolchain with regards to package management, and therefore trying to walk carefully around changes to avoid a fork, i feel uncommonly qualified to talk about the inhumanity of that position.



There's a reason that my 2016 article started out by centering the discussion in the considerations an individual person has to juggle when thinking about interacting with dependencies, rather than immediately jumping to the high-level picture. Laying out the diagrams and algorithms gives a sense of omniscience over the whole domain. But that is a false feeling. The reality is, when interacting with dependencies, or honestly even our own software, there are almost invariably huge gaps in our knowledge of what the software's behavior actually is, let alone what it should be, or why. But it's crucial to keep this centered in our mind, because 

When looking over these examples, you have to constantly remind yourself that we are reducing potentially enormous social complexity into cute little diagrams. At this level of abstraction, it seems easy to say, "well, just upgrade," but reality often imposes. Do you have changes in flight? Is there a pending critical security issue in the currently released version that your team is planning on releasing next week?  It most behooves the design of the system to assume that, despite the facility with which we can represent these relationships visually, each little module box is a wholly discrete entity, managed by people who do not know _any_ of the people in the other boxes, live halfway around the world from each other, the only human language they have in common is a broken non-native one, they're single parents, and their bosses could not possibly care less about the nuances of software dependencies.



Humans are not a hive mind, and any design that assumes we are is fundamentally untenable. Not even _machines_ are a hive mind, as distributed systems teach us - decisions made on the basis of locally available information may have significant deleterious effects globally, but it is costly, often prohibitively so, to expect  everyone to maintain a global view on the system at all times. Even worse is when such systems expect that  people can not only see globally, but _act_ globally.

But, like all systems that assume perfect information, vgo _sounds_ really nice on paper. 



If you want to understand what Russ thinks about the value of the community's labor, you need look no further than this very process: he has tried to sweep aside years of work, careful evaluation of use cases, consensus building, and actual working code. All because he has the power to do so, and an insufficiently-considered idea he wanted to try. Under MVS, that's the new normal.



===



Each iteration of that loop expands the "tested" circle a bit more, creeping ever closer to 

Here, the lower bound established by proof - that is, Go's type system - divides "impossible" from "possible". As we noted earlier, the type system provides clear, absolute guarantees, but they are negative ones: certain undesirable behavior _cannot_ happen. Formal methods to prove a program correct essentially do so by proving all undesirable 







This three-way classification of the program, then, is all about the overlap:

![behave-duuu](/Users/sdboyer/ws/sdboyer.io/content/behave-duuu.png)

The green section is the desirable behavior that is both described by the spec, and implemented by the program (let's assume the bug is fixed) - we refer to it as correct with respect to its specification. It's meaningless to say that a program is somehow correct in general - correctness is only something that can be evaluted with respect to a specification.

The unspecified section corresponds to the integer overflow case. The impossible section is a negative space - conceivable program behaviors that we do not want, and that an automated prover (for Go, the type checker) guarantees cannot exist in the program. We take this kind of thing for granted, so it's a little weird to think about, but the disallowed programs in the black space are all the ones that become possible if we switch to `interface{}`:

```go
func Sq(x interface{}) interface{} {
	return x.(int) * x.(int)
}
```



There's some other possible spaces in here. For example, the specification might describe some properties that are not fulfilled by the program:

![behave-incomplete](/Users/sdboyer/ws/sdboyer.io/content/behave-incomplete.png)

i'm fudging a bit here, Formally, we can't distinguish between "incorrect" and "incomplete." It might be more accurate to say that  Again, though, it's useful for explanatory purposes, and making the artificial distinction doesn't hurt anything in our case.

There's one last division we need to make: in addition to being simply unspecified, some behaviors are also _unknown_: that is, the behaviors are observable, but no maintainer of the code is aware that they exist:

![behave-unknown](/Users/sdboyer/ws/sdboyer.io/content/behave-unknown.png)

"Unknown" and "unspecified" are functionally identical, but for our purposes, the distinction is still useful. It's the first of these properties that brings us away from the machine, and to theyeah mind. That line is of paramount importance, as we'll see.






<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM2MTU4MDI0LC0xNjA4MDE3MTkyXX0=
-->
