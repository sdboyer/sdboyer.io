+++
title = "Failure Modes"
date = 2018-05-25T09:12:18-04:00
keywords = ["golang", "go", "clapback", "dep", "dependency management", "package management"]
+++

Twenty years ago, Alistair Cockburn [wrote a piece](http://alistair.cockburn.us/Characterizing+people+as+non-linear,+first-order+components+in+software+development) about the role of humans in software development. He started with a lovely thought:

> We have been designing complex systems whose active components are variable and highly non-linear components called people, without characterizing these components or their effect on the system being designed. Upon reflection, this seems absurd, but remarkably few people in our field have devoted serious energy to understanding how these things called people affect software development.

This post is focused on the failure modes of vgo, but in the spirit of Cockburn's sentiment here, it is focused on both mechanical and human failures. I believe that's the only way one can approach a problem like dependency management.

I went back and forth over the order in which to release this post and the compatibility post. I settled on putting this one first because I suspected that people would be (understandably!) skeptical about some of the strong statements in the first piece. To that end, this post provides evidence for four of the six [main issues](/vgo/intro#mvs-a-category-error). Paraphrasing the originals:

- MVS has all the same failure modes, plus more, minus pathological SAT.
- MVS only allows extreme remediation strategies on conflicts.
- MVS is hostile to experimentation.
- MVS loses crucial information, which makes it an unsuitable intermediate layer, and causes problems of its own.

The drawback to having failure modes precede compatibility is that the latter deals with _why_ we should expect incompatibilities to occur, even after v1, even with semantic import versioning, and even with cooperative participants. However, because I suspect most people accept that as a no-brainer, it was preferable to defer addressing that issue for this one.

This post relies heavily on basic concepts from game theory. Earlier drafts did not, because I was concerned it might be too off-putting for people, plus my own misgivings about game theory as a methodology. But, without a framework for more precisely expressing the problems induced by MVS, the concerns came off as fuzzy - unsurprising, given the complexity of the problem at hand. And the truth is, dependency management is almost a dream problem for game theory, particularly [evolutionary game theory](https://en.wikipedia.org/wiki/Evolutionary_game_theory): most relevant actions are part of an immutable public record, decision points can be readily identified, causality can be traced, identity is stable, and costs are easily identified.

That said, this post is not a formal game theoretic argument. (If there are any academics out there who'd like to collaborate on that, please [reach out](https://twitter.com/sdboyer)!). For example, I've omitted reference to payoffs - a necessary part of a proper model, but tricky in part because the motivation for participation in FLOSS is, at the very least, the subject of some debate.

I've approached it this way because I'm not trying to construct a formal proof, so much as help to tease out what has been a rather handwavy discussion about the social implications of vgo's technical failure modes. To that end, I've focused on identifying key decision points that people can make when interacting with the ecosystem of dependencies, primarily under MVS, but also under dep/gps, and the hypothetical gps2 when relevant. At each of these points, I identify a set of choices - "strategies" - that the people have available to them. By distilling these into clear, separate choices, faced by individual people at specific times, it helps us resist the temptation to handwave over complex, multi-step human processes.

This is essential, because it gives us the tools to systematically approach and analyze the wishful thinking that pervades the _social_ mechanism design - or really, lack thereof - in MVS. While the desired [technical outcome is clearly articulated in the official proposal](https://github.com/golang/proposal/blob/master/design/24301-versioned-go.md#semantic-import-versions):

> > An explicit goal for Go from the beginning was to be able to build Go code using only the information found in the source itself, not needing to write a makefile or one of the many modern replacements for makefiles. If Go needed a configuration file to explain how to build your program, then Go would have failed.
>
> It is an explicit goal of this proposal's design to preserve this property, to avoid making the general semantics of a Go source file change depending on the contents of `go.mod`.

The meaning of an import path should remain the same. And the tool and algorithmic design certainly enforces this. But the social mechanism design is essentially absent; it is basically just, "break on every state that is not the desirable final state, and that pain will drive humans to fix it." 

The problem is exemplified by [this statement (under Upgrade Speed)](https://research.swtch.com/vgo-mvs#upgrade-speed):

> But I think in practice dependencies will move forward at just the right speed, which ends up being just the right amount slower than Cargo and friends.

I can't tell if this is an intentional or accidental invocation of the [Goldilocks principle](https://en.wikipedia.org/wiki/Goldilocks_principle). Either way, I have no idea what "just right" actually means. All I can extract from it is a tautology: "I think MVS is right, therefore the upgrade speed under MVS is the right upgrade speed." Now, maybe there is a Goldilocks zone for dependency management, and maybe MVS would hit it. But because it's humans collaborating to do the work, if it does, it'll be because it creates a stable, habitable ecosystem for those humans. Through an exploration of both the technical and social failure modes, this piece aims to show that MVS, as a result of its intentional design choices, will run afoul of [this basic, obvious truth](https://theconversation.com/new-take-on-game-theory-offers-clues-on-why-we-cooperate-38130) from evolutionary game theory:

> The collapse of cooperation occurs when the ratio of costs to benefits becomes too high.

Finally, some notes about assumptions made throughout this post:

* I do not deal with the "[dep gets the latest](https://research.swtch.com/cargo-newest.html)" argument, at all; [as I noted in the introductory post](https://sdboyer.io/vgo/intro/#high-fidelity-builds), I have a design for something equivalent in dep/gps and gps2. Because that's the goal, and skipping it simplifies this post considerably, I'm going to just assume that gps[2] and MVS can exhibit the same "high fidelity" behavior, at least when initially adding dependencies. If I'm wrong about the viability of preferred versions and can't get it working in the next few months, I'll eat my hat.
* While there are some direct comparisons between MVS and gps[2] in this post, they are not full explanations, and not intended to be. A full alternative strategy is for a later post; the references here are to highlight where a more appropriate algorithm can improve on poor outcomes.
* Forking is an absurdly costly strategy, and is thus omitted from the mainstay of discussion in this post. It does have its own section towards the end of this post, explaining why.
* This post is full of illustrative examples. In them, module names/paths will be given as single, capital letters, like `A`,  instead of writing out a full module path. Modules will have a corresponding author, whose name begins with the same latter. So, Aparna is the author of `A` , and you could imagine its full module path might be `github.com/aparna/a`.

For process transparency: I first shared an earlier version of this post with Russ at the beginning of April, as part of the twice-weekly discussions he and I have had scheduled since last December. In the intervening weeks, I have rewritten much of the article, most notably including the addition of game theoretic concepts. 

## The Mechanics of Failure

What is a failure, in vgo's world?

With dep, it's usually easy to point to failures - they're explicit, verbose (and, currently, often difficult to understand, and printed out at the end of a `dep ensure` run.

The primary failure mode in vgo, however, is silent false positives - a `vgo {get,test,run,build}` command changes your dependency graph, and exits 0. Maybe everything's fine, maybe it isn't, but it's incumbent upon you to take additional steps to understand that your build is broken. 

Let's look at examples to see how that works, starting with modules `A` and `C`:

![fail-two-init](/vgo/fail-two-init.png)

Let's also assume that `A` contains at least one non-`main` package, and that some other module might depend on it. This does not necessarily mean it's public/OSS - it could be private, but within an organization that maintains multiple modules spread across multiple teams.

Now, let's imagine that there's some kind of incompatibility between `C@v0.9.0` and `C@v1.0.0`:

![fail-incompat](/vgo/fail-incompat.png)

The nature of this incompatibility is very important: is it an API change, or just behavior? Is it allowing new behaviors, or changing or restricting old ones? That gets into the nature of compatibility, though, which is the topic of the next post. Right now, we're just establishing basic mechanics, so all that matters is that this incompatibility exists (at least, from `A`'s perspective).

Still, I've chosen to represent the incompatibility gap as existing between `C@v0.9.0` and `C@v1.0.0`, because incompatibilities in the v0 range are expected, so it helps keep my mind from twitching. (Behavior in the v0 range is also crucial; we'll return to it at the end of this post.)

At this point, Aparna could easily encounter a problem - she runs `vgo get -u`, and receives `C@v1.0.0`:
![fail-up-broke](/vgo/fail-up-broke.png)

Now, the problem here might be subtle, and it's perfectly plausible that Aparna might not notice it. (Whether or not she notices, though, tells us nothing about the significance of the problem. If obviousness was correlated with severity, software would have no security holes.)

But, let's assume she does notice. This is an ephemeral failure - Aparna can easily roll it back with a `vgo get C@v0.9.0`:

![fail-ephfix-down](/vgo/fail-ephfix-down.png)

At this point, Aparna has solved the problem for herself. But, having discovered this incompatibility, the expectation under MVS is that Aparna will do one of the following:

- Change `A`'s code to fix the incompatibility, and release `A@v1.0.1` 
- Negotiate with Carla about reverting the change and releasing a `C@v1.0.1`, restoring compatibility with `A@v0.9.0` 
- Nothing, ignoring the incompatibility because it's not her immediate problem

Let's refer to these strategies as **Refactor**, **Bargain**, and **Ignore**, respectively, and look at each of them in terms of three dimensions: labor costs for Aparna, ecosystem externalities (costs or benefits experienced by people who did not choose to incur those effects), and whether Aparna can complete the strategy on her own:

| Strategy | Labor Cost                                                   | Positive Externalities                                       | Negative Externalities                                       | Unilateral |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------- |
| Refactor | Moderate to prohibitive (depending on complexity of the fix) | MVS works as intended: `A`'s dependers can just update, and everything works | None                                                         | Yes        |
| Bargain  | Moderate to prohibitive (depending on complexity of the change, and Carla's agreeableness to it) | Compatibility continuity is restored                         | Compatibility may be broken for anything already relying on `C@v1.0.0` | No         |
| Ignore   | Zero                                                         | None                                                         | Any depender on `A` that updates `C`, whether by `vgo get -u` or adding `B`,  would silently get `C@v1.0.0`, a known-bad combination | Yes        |

Ignore is free for Aparna, but lays a land mine for others. Refactor and Bargain will have manageable labor costs sometimes, but there's no guarantee. Other times, they could represent months of work.

Ideally, Aparna would have a middle ground strategy here - a way of declaring the existence of this incompatibility that she has discovered, so that anything depending on A won't be surprised by it if they attempt to update `C`. Furthermore, it'd be ideal if that strategy had a low, predictable labor cost, and if the positive externalities consistently outweighed the negative ones.

Let's refer to any strategies that simply make an incompatibility known under the umbrella term **Declare**. This could conceivably take a few forms:

- Ragetweeting
- Big, bolded text in a README
- Writing a test that fails on `C@v1.0.0`
- Making a precise symbolic declaration that dependency management tooling can understand

Tests are interesting. We'll revisit them in a later post. But it's the last approach that's relevant here.

Under dep/gps, Aparna could play Declare by creating a version constraint. This `Gopkg.toml` entry would do the trick:

```toml
[[constraint]]
  name = "C"
  version = "<1.0.0"
```

Of course, this approach also excludes a vast swathe of potential versions. In other words, it's predicting the future - something that humans are usually bad at. This is an area where dep would really benefit from reducing its expressiveness - a goal for gps2.

vgo does not allow such declarations, as respecting them would entail that MVS cross Schaefer's dichotomy into an NP-hard search. However, while discussing an earlier draft of this post, Russ agreed that a service to track incompatibility declarations would be beneficial for vgo. In the interest of being as fair as possible to vgo and MVS, I'm going to pretend like its general behavior part of the specification already.

With such a service, Aparna could then Declare the incompatibility. MVS itself would not know or care that such a declaration exists, but the logic in  `vgo get`  surrounding MVS would. If MVS were ever to select  `A@v1.0.0` and `C@v1.0.0` together, vgo would be able to warn the user about it.

Let's bring this back in line with the other strategies by adding to our table:

| Strategy    | Labor Cost | Positive Externalities                                       | Negative Externalities                                       | Unilateral |
| ----------- | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------- |
| Declare-gps | Trivial    | Dependers on `A` will never encounter this known-bad version combination. | `A@v1.0.0` may work "well enough" with `C@v1.0.0` for some of `A`'s dependers; the solver may pick an absurdly old version | Yes        |
| Declare-MVS | Trivial    | Dependers on `A` will be aware that someone thinks the version combination is bad. | Manually remediating simple warnings is [toilsome](https://landing.google.com/sre/book/chapters/eliminating-toil.html); manually remediating complex sets of warnings is mind-numbing | Yes        |

It is essential to note that, in this situation, if Aparna were to Declare, it would unequivocally improve the health of the ecosystem. Russ [has often asserted](https://github.com/golang/proposal/blob/master/design/24301-versioned-go.md#ecosystem-fragmentation) that constraints with global scope harm the ecosystem, but that is only the case if they are erroneous. Declarations that accurately describe an incompatibility do not help _as much_ as fixing the underlying problem, but they do improve the accuracy of our collective understanding of the ecosystem's compatibility properties - and at a fixed, tiny cost to the Aparna.

Let's walk the example forward to see the effects of these strategies in practice.

Imagine we have another module, `B`, which requires `C@v1.0.0`, across the incompatibility gap:

![fail-base](/vgo/fail-base.png)

Let's also imagine that a fourth module, `D`, into which Deon is trying to incorporate both `A` and `B`. This creates a classic diamond dependency on `C`. Under MVS' build list algorithm (Algorithm 1 in [the blog posts](https://research.swtch.com/vgo-mvs)), the `require` declarations are minimums, so `D`'s build will end up with `C@v1.0.0`:

![fail-diamond](/vgo/fail-diamond.png)

Now, if `A@v1.0.0` truly can't work with `C@v1.0.0`, and `B@v1.0.0` truly does require `C@v1.0.0`, then there's simply no resolution here - this build will never work. What the tool can tell Deon about that reality, however, depends on the strategy that Aparna chose.

* If Aparna Ignored (or if she didn't notice herself - these are indistinguishable to Deon), then neither MVS nor gps could have informed Deon that this depgraph is unworkable. He has to notice for himself. That's likely more difficult for him than her; as the author of `A`, she almost certainly knows the `A→C` boundary much better than he does.
* If Aparna Negotiated, it is effectively equivalent to Ignore until some unknown time in the future when the negotiation ends and a new release of `C` is made.
* If Aparna Refactored, then the immediate problem goes away, though it does require a new release of `A`.
* If Aparna Declared under gps, then Deon's `dep ensure -add B` would result in a failure with a conflict message indicating the problem. This is a frustrating outcome, but it's also optimal: the tool immediately carried Deon to the inevitable conclusion. (Declaring in this way would also require a new release of `A`.)
* If Aparna Declared under MVS, then Deon's `vgo get B` will exit successfully, but show a warning about the `A→C` combination. Deon could then take manual action, eventually coming to the conclusion that he has to abandon `B` entirely. (As the mechanism for an incompatibility service is TBD, it's not clear whether a new release of `A` would be required.)

(Aside: what we know about [loss aversion](https://en.wikipedia.org/wiki/Loss_aversion) suggests that it would be considerably more infuriating to have the command complete successfully, only to discover later that `B`  cannot be incorporated at all, than to simply have it rejected up front.)

Things change a bit when additional releases of either `A` or `B  `are in play. If there are multiple versions of `B`, and one `require`s  `C@v0.9.0`:

![fail-get-twob](/vgo/fail-get-twob.png)

With this set of possibilities, it's now possible to find a working build combining both `A` and `B ` - we just need `B@v0.9.0`, instead of `B@v1.0.0`. This is a straightforward jump to make, and if Aparna Declared under gps, then Deon's `dep ensure -add B` would select the workable solution: `A@v1.0.0`, `B@v0.9.0`, `C@v0.9.0`.

vgo's behavior would be the same, though - pick `B@v1.0.0`, emit a warning, and Deon has to remediate. He can do so with a `vgo get B@v0.9.0` - though figuring out that that's the right move is toilsome.

If there are more versions of `A`, the current iteration of gps can make some bad choices. Let's say that Aparna issued a new release, `A@v1.0.1`, in which she Declared incompatibility with `C@v1.0.0`:

![fail-twoa-incompat](/vgo/fail-twoa-incompat.png)

Aparna has put her knowledge of this incompatibility out there, which is great. But when Deon tries to pull in `B`:

![fail-get-twoa](/vgo/fail-get-twoa.png)

It's the same as the previous situation. There's no way this build can work, but Deon doesn't know that. And dep, as it is implemented today, would unfortunately make a bad decision: it would see that `A@v1.0.1` can't work with `C@v1.0.0`, but that `A@v1.0.0` has no such restriction, and allow selection of the latter. This would provide Deon with the broken combination `A@v1.0.0`, `B@v1.0.0`, and `C@v1.0.0` - a regression to vgo-esque false positives.

This is the actual, practical drawback of using a solver, far more so than a SAT bogeyman that's coming to eat your CPU: walking back from information-rich areas to information-poor areas. A solver may see versions about which an incompatibility is known (here, `A@v1.0.1 → C@v1.0.0`) and walk back in the version history until it finds something that does not have a reported incompatibility (here, `A@v1.0.0 → C@v1.0.0`). But "no information" is not the same as "is compatible."

In theory, this is one of the things MVS helps with - it guarantees that if its algorithm changes the version for some module, at least one other module in the dependency graph is already using that module at the newer version. That's interesting information, but it's also not terribly relevant: it says more about the newer version's _bug-freeness_ than it does about _backwards compatibility_. In fact, it's not even obvious to me that compatibility and bug-freeness are correlated. If indeed they're not, MVS' guarantee loses much of its utility. (I'd love to see someone gather some empirical data on this!). It's still a reasonable place to start, as there's strictly fewer novel version combinations being used in the system - but we're now talking about a difference of degree, not kind.

Either way, we know this safe-to-update assumption will inevitably fail. When it does, the user is stuck in the same position of uncertainty as they would be with a solver. But they're even worse off - if Declare-MVS information causes vgo to warn, it creates the same problem with user expectations that solvers have: after Deon toils through the rollbacks until the tool stops spitting out warnings, he _still_ has to differentiate between "no more warnings because they're compatible" and "no more warnings because nobody has yet noticed they're incompatible." And, if there's more than a a couple versions to manually evaluate and rollback through, he's probably already exhausted his cognitive budget.

That both systems have this problem with ambiguity between "is compatible" and "compatibility unknown" indicates that this is an essential domain problem, not some incidental artifact of SAT. As such, dealing with it is a central design goal of gps2. I currently intend for the basic mechanism to be look something like this: when Aparna publishes `A@v1.0.1` and Declares an incompatibility with `C@v1.0.0`, that incompatibility is projected _back in time_ to previous versions of `A`:

![fail-twoa-backintime](/vgo/fail-twoa-backintime.png)

There are costs to this, and it has to work in concert with other mechanisms to be properly effective. But it's simple and intuitive, and squarely addresses the aforementioned problem of overly-aggressive solver search. And, by attaching it to releases instead of a standalone service, we can maintain a useful invariant about new constraint informatino only appearing at the _end_ of the timeline.

It also mirrors an incontrovertible reality: future changes to the ecosystem can affect the compatibility properties of pre-existing releases. If we can find a targeted way of allowing future knowledge to improve on our current decisions, while still retaining the spirit of immutable releases and avoiding new negative externalities, it's a major win.

### Contagion Failure

Let's backtrack a bit in our examples to when Deon first entered the story, so that we can explore an alternate path:

![fail-diamond](/vgo/fail-diamond.png)

On the first pass through, we assumed that the requirement from `B@v1.0.0` was true. But what if it isn't? That is, what if `B@v1.0.0` actually _is_ compatible with `C@v0.9.0`, and its `require` declaration points to a newer version than it truly needs?

The next section will explore why this is quite plausible. For now, though, let's say this happened because Bjorn ran `vgo get C`, ending up with `C@v1.0.0`, but he only used a function from `C`  that was unchanged across `v0.9.0` as in `v1.0.0`.

If Deon notices all this, then he's in a bind. He can fix the problem for himself via a `replace "C" v1.0.0 => "C" v0.9.0`. But, if he then makes a new release:

![fail-replace-pub](/vgo/fail-replace-pub.png)

`D` will get the necessary version of `C`. But the `replace` fix is "local" - vgo only applies it when `D` is the module being built. Thus, `D@v1.0.0` has baked in what I'll refer to as a **contagion failure** - one that will spread to any dependers on `D` (and their dependers, recursively).

Let's look at Deon's strategy table for this decision:

| Strategy          | Labor Cost                                                   | Positive Externalities                                  | Negative Externalities                                       | Unilateral |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------- | ------------------------------------------------------------ | ---------- |
| Bargain (`A`)     | Moderate to prohibitive (depending on complexity of the change, and Aparna's agreeableness to it) | Ecosystem is fully restored to compatible state         | None                                                         | No         |
| Bargain (`B`&`C`) | High to prohibitive (depending on complexity of the change, and both Bjorn and Carla's agreeableness to reverting) | Compatibility continuity is restored                    | Compatibility may be broken for anything already relying on `C@v1.0.0` | No         |
| Replace           | Trivial                                                      | The new, improved version of `D`  can ship to the world | Contagion failure                                            | Yes        |
| Give up           | Zero                                                         | None                                                    | The world doesn't get Deon's new, planned feature            | Yes        |

Deon has more Bargaining strategies available to him than Aparna did, but they're also necessarily more work, as one of them involves negotiating with both Bjorn and Carla. Crucially absent is the unilateral Refactor strategy - Deon cannot fix this entirely on his own. It's also worth noting that his Bargain strategies qualify as [yak shaving](https://en.wiktionary.org/wiki/yak_shaving), although it's someone else's yak.

If Deon does choose the easy Replace option available to him and issues a release, his depender, Emiko will not reap the benefit of his research. She will have to repeat both flashes of insight about `B→C` and `A→C`, because MVS will push `A` back onto the incompatible version of `C` until the `replace` declaration is duplicated:

![fail-contagion](/vgo/fail-contagion.png)

We'll refer to the act of copy/pasting `exclude` and/or `replace` declarations from `D` into `E` as **hoisting**. 

When faced with contagion failure, Emiko's options are even worse than Deon's were.

| Strategy                | Labor Cost                                                   | Positive Externalities                                  | Negative Externalities                                       | Unilateral |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------- | ------------------------------------------------------------ | ---------- |
| Bargain (`D` & `A`)     | Moderate to prohibitive (depending on complexity of the change, and both Deon and Aparna's agreeableness to it) | Ecosystem is fully restored to compatible state         | None                                                         | No         |
| Bargain (`D` & `B`&`C`) | Moderate to prohibitive (depending on complexity of the change, and Deon, Bjorn and Carla's agreeableness to reverting) | Compatibility continuity is restored                    | Compatibility may be broken for anything already relying on `C@v1.0.0` | No         |
| Hoist                   | Trivial                                                      | The new, improved version of `E`  can ship to the world | Contagion failure, redux                                     | Yes        |
| Give up                 | Zero                                                         | None                                                    | The world doesn't get Emiko's new, planned feature           | Yes        |

This is a general pattern, one almost entirely skirted by the vgo writings: problems get worse the further removed the user is from them in the depgraph, and they have no real ability to contain such problems. They can only Give Up.

### Other Diamonds

It's important to contrast contagion failure modes with the shape of the cases that are more commonly discussed in writings about vgo. Those cases tend to present the reader as the author of `A`:

![fail-fixable-contagion](/vgo/fail-fixable-contagion.png)

Let's refer to this case as a **half diamond**.

Because it's our own module that's relying on the pre-breakage version of `C`, the situation is more tractable than normal contagion failure. It's our code, so it's within our sphere of influence to fix. If we were in charge of `B` in this half diamond, though, then the only code-level change in our direct control would be to _downgrade_ our logic to work with `C@v0.9.0`. That's usually a non-starter, so we'd have to consider it another case that's out of our hands.

Half diamonds are nicer, but there's no reason to believe that they would be the norm in the wild. Things could well go in the other direction, with multiple intermediate modules:

![fail-deepdiamond](/vgo/fail-deepdiamond.png)

We'll call these **deep diamonds**.

Deon wouldn't notice the problem between `A` and `C` when he first runs `vgo get A`, because there's nothing forcing  `C` forward to `v1.0.0`. It's only when `F` connects the diamond that the incompatibility becomes a problem in a real build. But Frank is two degrees removed from the details of the `A → C` relationship. As such, he's the _least_ likely to have knowledge about how to approach the problem - but he's the one saddled with addressing it.

## Minimum Versions and Information Loss

Many of the other parts of this analysis establish the ways in which MVS is at least no better than a SATful solution, and often worse - that is, makes unduly difficult to work on Go software. The argument, usually implicit, is that while asceticism in tool design may make sense for individual work, when it comes to tools that shape community interactions, the same techniques are often not applicable. (This is somehow controversial, despite the profound differences between individual and group dynamics being a subject of literary and academic exploration for millenia)

This section is a bit different. It is concerned with what may be the single most foundational problem with MVS: crucial information that MVS discards. In discarding this information, MVS laces every possible updating strategy with harmful externalities, and renders vgo unsuitable as an intermediate layer on which a more robust tool could build to address its deficiencies.

vgo's design centers around the `require` declaration. When such a declaration appears in a `go.mod`:

```go
module my/thing

require "other/thing" v1.0.0
```

it is, by design, compacting two bits of information into one declaration:

1. The _minimum_ version of `other/thing` that must be used in any build involving `my/thing`
2. The _actual_ version of `other/thing` used when building any package contained in `my/thing`

Russ makes the argument that this is a beneficial design choice because it prevents the minimum version constraint from becoming stale and inaccurate. And, certainly, this is a real concern: he points to some plans that Cargo has to test crates against the bottom of their version ranges at publication time. The goal of this testing is to verify that the minimum declarations have not gone stale - a tractable exercise, but still costly. When testing is complete, we can be reasonably assured that the declaration will at least be closer to "true minimum" - the actual lowest version for a dependency at which a package works correctly.

But the knife cuts both ways. Vgo's minimum versions can lie in a manner that is dual to Cargo's: they can indicate a minimum version that is _newer_ than true minimum. When everything's working well, this is fine, but in anything other than an ideal environment, the loss of that information can be a problem. In fact, the harms arising from this situation are much worse than those arising from stale minimum versions, because instead of allowing potentially bad combinations, they cut off potentially good ones.

The [Contagion section](#contagion-failure) was built around an instance of this problem, where Deon created a contagion failure by using `replace` declarations to compensate for `B`'s too-new `require`  declaration on `C`. In that case, we assumed this skew was a result of Bjorn simply running `vgo get C` and getting the latest version, even though his actual use of `C` didn't necessitate it.

If the problem were limited to first time use, then it would be worrisome, but not awful. But that is not the case; it infests the day-to-day choices we make as maintainers. There are structural  reasons that drive this problem after some examples that more thoroughly illustrate how it works.

Let's extend our running example by adding a few more versions in one of the intermediate modules. Dotted lines represent the `require "C"` declaration in each version's `go.mod`:

![iloss-universe](/vgo/iloss-universe.png)

Let's enhance this a little more by also using gold arrows to mark true minimum. If, for a given module version, the  `require "C"`  declaration also happens to be true minimum, then there will be only a gold arrow:

![iloss-univ-truemins](/vgo/iloss-univ-truemins.png)

This is saying that all of the module versions have `require "C"` equal to true minimum except `B@v1.0.1`, which has a true minimum of `C@v0.9.0`, but a `require "C" v1.0.0`.

OK, that's the setup. Now, we're going to pretend we're acting as `mymod`, which already uses `A`, and wants to add `B` via `vgo get B`:

![iloss-truemins-contagion](/vgo/iloss-truemins-contagion.png)

That will give us `B@v1.1.0`, which leads MVS to pick `C@v1.0.0`, thereby breaking `A`. Let's assume this is an API-level breakage, so the type checker catches it immediately when we run `go test`, and tips us off that it's the `A@v1.0.0 → C@v1.0.0` link where the problem lies. There are three possible courses of action we can take ourselves: use a `replace` (and create a contagion failure), fork `A` and `C`, or try downgrading to `C@v0.9.0`.

In the current setup, the downgrade (`vgo get C@v0.9.0`) will succeed by also rolling back `B@v1.0.0`. However, because `B@v1.0.1` has divergent `require` and true minimums, this rollback went further than was actually necessary.

Now, if we tweak the module version sets so that the same divergence existed in `B@v1.0.0`:

![iloss-univ-truemins2](/vgo/iloss-univ-truemins2.png)

then MVS will determine that no versions of `B` will work, and `vgo get C@v0.9.0` will fail (either explicitly, or by 'succeeding' with the sentinel version `C@none`, indicating that the formula is unsatisfiable). The only recourse we'd have is to not use `B` at all, or manually search through the versions of `B` with a `replace "C" v1.0.0 => "C" v0.9.0 ` until we find `B@v1.0.0`.

But how would we even know to attempt this manual search? This is the key problem of information loss: we're showing "true minimum" in these diagrams, so it seems obvious, but in reality, true minimum is generally unknown, possibly even unknowable.[^1] Only the `require` is visible, and with these module version sets, `require` tells us that no versions of `B` work with anything older than `C@v1.0.0`. Iterating through these versions manually is groping around in the dark. Worse still is the fact that, if this effort succeeds, the only tool we have to record what we've learned is a `replace`, and that means contagion failure: our dependers have to go through the same process all over again.

The information loss is also harmful to an algorithm like gps2, were it to attempt to operate atop vgo and automate the search process for us - in those unhappy scenarios where upgrading doesn't work, the algorithm can't know which versions _below_ the `require` are worthwhile candidates to attempt for solutions. A separate minimum version, a la Cargo or dep, is vastly more likely to include true minimum. An explicit minimum can become stale, of course, but that is certainly preferable to obscuring true minimum: allowing for a downgrade space that includes true minimum would have made finding a solution for the above case trivial, without entailing rollbacks.

Additionally, gps2 could mitigate the stale minimum problem. The other major underpinning concept in gps2, in addition to the backwards-projecting incompatibility declarations discussed previously, would be doing (heavily cached) package-to-package type checking during the solving process. This can be thought of as an alternative to Cargo's strategy of running tests to find minimum versions, except that we can do it on the fly. "API minimum" is thus something the tool could give us for free - no explicit declarations required, and no possibility of it being incorrect. This isn't sufficient - as the next post will cover, API and behavioral compatibility are separate things - but it's an enormous step in the right direction.

### Update Strategies

Whatever we might be able to do with gps2, as long as MVS is still the foundational algorithm behind `go {get,test,run,build}`, all of the available strategies have negative externalities.

By "general update strategy," I mean a small ruleset that users can reliably apply so that they can feel that they are satisfying both their own needs, and being helpful to the community. All such strategies are going to fall between two endpoints:

* **Ride the Bottom**: use the oldest version possible (true minimum), updating only as much as is required to receive specific, known changes.
* **Ride the Top**: update constantly, as soon as new versions of dependencies become available.

At first glance, riding the top seems like it should be the generally recommended course of action, at least for any module that's not a pure `main` package. Certainly, [Russ appears to believe](https://research.swtch.com/vgo-mvs#upgrade-speed) this is a normal course of action:

> Upgrading all modules is perhaps the most common modification made to build lists. It is what `go get -u` does today.

And riding the top is what [maintainers should probably be doing](https://github.com/golang/proposal/blob/master/design/24301-versioned-go.md#ecosystem-fragmentation), as it's how they would discover if there are some incompatibilities that MVS expects them to address through refactoring. However, as [Aparna's strategy table](#the-mechanics-of-failure) indicates, Refactor is either the first or second most costly strategy available. Now, she may be deriving some benefit from fulfilling a sense of duty to the community, but if she's reflexively performing Refactor, it's unlikely to be furthering any of her own goals for `A`. "Always Refactor" is thus, in game theoretic terms, a [purely](https://en.wikipedia.org/wiki/Strategy_(game_theory)#Pure_and_mixed_strategies) [altruistic strategy](https://en.wikipedia.org/wiki/Evolutionary_game_theory#Routes_to_altruism). (Bear with me now - we're gonna go a little deeper on the game theory.)

Evolutionary game theory holds that for systems built on [altruistic strategies of indirect reciprocity](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3279745/pdf/nihms49939.pdf) to find stability - that is, for all players to stick with an altruistic strategy - then there needs to be some kind of reputational scoring system for tracking peoples' adherence to altruism.[^2]

If we think about what a reputational scoring system for contributions might look like, there's a remarkably direct real-world analogy: the contributions chart on every GitHub profile page. That's the very same chart [from which they eliminated streaks](https://blog.github.com/2016-05-19-more-contributions-on-your-profile/) in part because [gamifying](https://www.mxsasha.eu/blog/2016/04/01/how-github-contribution-graph-is-harmful/) [contributions](https://blog.jcoglan.com/2013/11/15/why-github-is-not-your-cv/) [can be harmful](https://zachholman.com/posts/streaks/) and - especially if the express intent is to produce labor, i.e., induce maintainers to play Refactor - [unethical or even downright exploitative](https://www.ashedryden.com/blog/the-ethics-of-unpaid-labor-and-the-oss-community).

To make matters worse, the stability of evolutionary systems rooted in indirect reciprocity are [especially vulnerable to "defection"](https://en.wikipedia.org/wiki/Evolutionary_game_theory#Routes_to_altruism) - where players select a strategy that favors themselves over the common good. The Ignore strategy fits this description exactly - and it is also the default strategy for Aparna, by virtue of it requiring no action at all. With such a strong default to defection, we have to expect that the system will _never_ stabilize on altruistic strategies. 

Intuitively, that means that the system will never reach the idealized goal of an ecosystem in which it is reliably true that "the semantics of a Go source file are not dependent on `go.mod`." It may not even converge on it. Basing a system design on an unreachable goal is not usually the best way to go.

Finally, riding the top has the negative externality that's the core concern of this section: information loss. With MVS conflating current and minimum version, updating anywhere past true minimum cuts off viable build paths. Of course, that's the design goal - a ratchet to drive people towards updating. But if the structural factors in the system suggest that that eventual goal is usually unreachable, then reflexively ratching itself may be a harmful strategy.

OK, that's riding the top. But riding the bottom has harmful externalities, too.

For one, of course, avoiding exploration of the upper ranges is shirking Refactor, the core strategy MVS assumes. Over time, more subtle incompatibilities will accumulate along the future paths, and `vgo get -u` turns into a ticking time bomb.

Moreover, as all of the examples so far illustrate, that ratchet primarily applies pain to the Deons, Franks and Emikos, who are integrating other modules - not the Aparnas, who are actually empowered to make the change. Instead of insulating the consumers, the end users, from the vagaries of the dependency management ecosystem, MVS sticks them right in the middle. The _actual_ ratchet is humans excoriating other humans for not performing the labor MVS expects of them. Feature, not a bug.

But there's another thing we haven't touched on at all yet - it's also an important benefit for authors of modules to have people actually use their new releases:

<blockquote class="twitter-tweet" data-cards="hidden" data-lang="en"><p lang="en" dir="ltr">🔥😍🔥😍 <a href="https://twitter.com/hashtag/webpack?src=hash&amp;ref_src=twsrc%5Etfw">#webpack</a> 4.8.0 😍🔥😍🔥 <br><br>Is out!!!!! Update those deps!!! Report 🐛🐛🐛🐛🐛🐛<br><br>Awesome new feature:<br>* webpack now leverages instansiateStreaming which allows for large wasm binaries to be bundles. Hats off to <a href="https://twitter.com/svensauleau?ref_src=twsrc%5Etfw">@svensauleau</a> and <a href="https://twitter.com/wSokra?ref_src=twsrc%5Etfw">@wSokra</a> collaboration on this. cc <a href="https://twitter.com/linclark?ref_src=twsrc%5Etfw">@linclark</a> <a href="https://t.co/wBxXIa3w58">pic.twitter.com/wBxXIa3w58</a></p>&mdash; Sean Thomas Larkin (@TheLarkInn) <a href="https://twitter.com/TheLarkInn/status/993469063769702401?ref_src=twsrc%5Etfw">May 7, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Not everybody wants to spelunk for bugs in their dependencies, and that's perfectly fine. But that feedback is a crucial part of operating within a community, and at least _some_ is needed - "Given enough eyeballs, all bugs are shallow." As such, riding the bottom is a [free rider behavior](https://www.investopedia.com/terms/f/free_rider_problem.asp) - some people can get away with it, but if too few people update their dependencies until the last possible minute, then bugs aren't caught and incompatibilities aren't discovered.

The narrow strategies actors have available to them are mitigated by adding Declare strategies to the maintainer's arsenal. Such strategies are cost-comparable to Ignore, and while they're not as beneficial for the ecosystem long-term as a Refactor, a Declare, made through a well-designed mechanism, is still an absolute improvement over not having them.

But Declares don't help with information loss. The only way to truly make that better is to split "current" and "minimum" into separate declarations. That would allow the best of both worlds, canceling out the harmful externalities of each: some people can tend towards riding the top, while providing others with an escape hatch when their locally-risky, globally-eventually-beneficial behavior proves harmful. (This would be the aim of gps2).

However, it's impossible to separate minimum and current, and keep MVS at the same time. Distentangling any kind of explicit minimum from `require` would entail injecting new information into the build list algorithm. I don't believe it would be enough to push the algorithm over Schaefer's dichotomy into NP-hard search, but uniqueness and minimality would necessarily be lost, which would unravel other guarantees that make MVS remotely viable.

### Cascading Rollbacks

One way of observing the harms arising from information loss is to look at what happens when a rollback is required.

Let's reimagine our ongoing example with an extra intermediate module, `D`, where it's `A` that we're trying to introduce:

![iloss-cascading-rollback](/vgo/iloss-cascading-rollback.png)

The first fix attempt is still the same - `vgo get C@v0.9.0`. MVS will force `B` to roll back to `B@v1.0.0`, even though it's not actually necessary. That will entail a cascading rollback on `D`, which will be a real failure, because all versions of `D` _actually_ need `B@v1.0.1`, but based on the artificial rollback of `B`. Again, we have to resort to a fix involving `replace`, resulting in contagion failure, but now the blind search may involve versions from both `B` and `D`.

Rollbacks are obviously never optimal. But cascading rollbacks that occur when they're not truly necessary are dangerous. The cascade risks making it more like taking your entire project back in time, rather than narrow reverts. It re-exposes us to bugs that we'd already accepted as fixed, and potentially reverts features. And old/recurring bugs are not the same as new/ongoing bugs: they're more infuriating, not just for us, but also to the expectations of our coworkers and customers, especially if we, say, issued a new SLA on the basis of thinking a particular problem solved. It's one thing if a mistake i make causes an old bug to resurface, but it's quite another to be forced to reintroduce a bug because my package manager is insufficiently expressive.

Cascading rollbacks are also the quickest way to see why building a searching tool on top of vgo/MVS would be unwise, at best. Once we accept the reality that, at any given time, it is _good_ and _expected_ that current version and true minimum might diverge when people ride the top, it follows that it is _good_ that, when the circumstances demand it, we can move a current version backwards towards its true minimum while changing as little of the rest of the depgraph as possible. That's a simple matter of variable isolation.

## Phantom Rules

vgo treats the module graph - that is, the graph constructed from `go.mod` `require` declarations - as being entirely separate from the package import graph. The module graph is always explored fully and unconditionally. That is, if `A/go.mod` contains a `require "B" v1.0.0` declaration, then vgo will always visit`B/go.mod` and its `require`d modules, recursively, independent of whether or not `A` or `B`'s packages' have any `import` statements pointing to packages in the named modules.

When things are working well, this is somewhere between "nice" and "ok, whatever". The "nice" bit is that it obviates the need for something like dep's [`required` property](https://golang.github.io/dep/docs/Gopkg.toml.html#required), because simply having an extra `require` in `go.mod` for, say, the program you use to compile your protobuf means that running `go get import/path/for/protobuf` from within the module will pull down what you want.

The "ok, whatever" bit is how this behaves when looking at the `go.mod` files of dependencies. Let's say that we have this package import graph, where the boxes are modules, the ovals are packages, the arrows represent imports, and `A` is the module we're working on:

![phantom](/vgo/phantom.png)

Assuming that `B/go.mod` is complete, it'll have a `require` declaration in it for `C`. However, `B` only imports a package in `C` (the root and only package, also named `C`)  from its subpackage, `B/foo`, which we do _not_ import from `A`. Nevertheless, because the module and import graphs are independent, vgo will incorporate `B`'s `require "C"` declaration, and it will be factored into our build up in `A`.

When a `require` rule is applied even though it's not necessary, we can call it a **phantom rule**. Now, as long as all compatibility assumptions hold in `C`, phantom rules don't really matter. When compatibility fails, though, problems ensue.

Say that `B` - more properly,`B/foo` - claims to need a version of `C` that's across an incompatibility gap from what we need in `A`. Given this module graph, that would mean we're facing contagion failure, with the cause being a phantom rule that has no proper business affecting our dependency graph at all.

Phantom rules also combine with information loss to create triply absurd conditions for failure. Say that `B`'s true minimum for `C` would work for `A`. Now, Aparna is facing a circumstance where she's tempted to use a Replace strategy to achieve her goals, resulting in a contagion failure. That's happening as a result of a rule that has no business being in the module graph - and Bjorn was riding the top, so the code doesn't actually need that newer version.

gps [dealt with phantom rule problems early on](https://github.com/sdboyer/gps/pull/36) by always using the import graph to determine which paths to follow. That also has some drawbacks - added complexity of course, and the chicken-or-egg problem with [using `dep ensure -add` on a package you have yet to import](https://golang.github.io/dep/docs/daily-dep.html#adding-a-new-dependency) - but we accepted the latter problem because it was one of the many that could be fixed once something dep-shaped was in the toolchain.

vgo, on the other hand, cannot fix this problem while retaining MVS. The uniqueness properties of MVS depend on the set of reachable modules in the module graph remaining being exactly the same, whether a given module is acting as a dependency or the root. If there was a possibility of certain paths in the module graph not being explored, then Algorithm R would not, in general, be able to compute a declaration set that is both a) unique and b) guaranteed to be visited in all possible graph traversal variants.

This also amounts to another reason for an explicit lock file; if uniqueness is abandoned as a goal, then by far the simplest thing to do to guarantee information availability to all traversal variants is just roll up the transitive closure into a file. That, in essence, is a lock file.

### Forking

Whenever someone forks a project - including changing `go.mod` and internal import paths to the new namespace - that fork will lie somewhere on a spectrum between two points:

- A temporary throwaway
- The carefully cared-for product of a long-considered discussion by an organization about whether they have the bandwidth to properly support a fork, all of which occurs only after a protracted discussion with the original authors of the upstream project about what their plans are, and whether the original functionality can be restored

Certainly we'd prefer to see more of the latter. No matter where a fork lies on this spectrum, however, the following will be true:

- The fork will likely be severed from any infrastructure we may build for distributing vulnerability notifications. Motivated attackers, however, will have no problem making the leap.
- Forking entails changing the `go.mod` file, and therefore at least one new commit. That commit is necessarily not in the original module, which means mechanically relating the two (e.g., to try to distribute the aforementioned vulnerability notifications) enters a different category of difficulty.
- The fork's types are now incompatible with the original module's types. Type aliases also probably can't help here, because it is likely unsafe to make direct reference to the original module from the forked module due to whatever incompatibility problem induced the fork in the first place.
- The original project is also cut off from feedback that it might otherwise have received from dependers that switched to the fork. This is especially harmful in the pre-`v1.0.0` period; more on that later in this post.

When forks lean towards the well-considered end of the spectrum,  all of these costs are more or less acceptable. But it seems more likely that well-considered forks will be a vanishingly small exception, because MVS allows only three avenues for permanently addressing incompatibilities:

- Change the importing module to work with the dependency.
- Change the dependency to work with the importing module.
- Fork.

Of the three, forking is the only one _guaranteed_ to be within the unilateral control of the person encountering the problem. If forking is the only escape valve afforded to Go developers that lets us protect our dependees from contagion, then it's what we'll use. And we'll use it regularly, with little of the careful consideration that, in an ideal world, we'd all prefer.

When forks are used as a band-aid, yet more challenges come into view:

- While machines will treat the original module and any forks of it as discrete without difficulty, there will still be a significant name dilution problem for humans ("wait, who's fork are we talking about?").
- Unless someone figures out a reasonable way to make some kind of fork registry to avoid duplicate forks, releases of sufficiently popular modules with iffy compatibility properties will see multiple forks that excise problematic new releases, even though one would be sufficient.
- If we are taking the "names have consistent semantics across versions" rule seriously - and surely we must be, because that's what necessitates forking gymnastics in the first place - then fork namespaces can't be reused. For example: 
  - Bob forks `module github.com/a/foo`  to `module github.com/b/foo` to temporarily deal with a breaking change in `a/foo@v1.0.0`.
  - In a later version, Bob's module returns to `a/foo@v1.1.0`, having come back into alignment with upstream.
  - Still later, Bob needs to fork `a/foo` _again_ for another incompatibility in `v1.2.0`, this second fork must be careful about reusing the `b/foo` namespace. Unless `a/foo` reverted the breaking change in `v1.0.0`, then reusing `b/foo` by adding new versions to the end of the timeline will end up recreating the breakage that the initial fork was created to circumvent.

The big kicker with forks, however, is that unlike `replace` or `exclude` declarations, forking can't skip over intermediate dependencies. That means, in a deep diamond scenario, instead of just declaring a `replace "C"`, Frank would have to fork all of `D`, `A`, and `C`:

![fail-fork](/vgo/fail-fork.png)

To make matters worse, vgo also allows modules to have circular `require` declarations:

> Declaring this kind of cycle can be important when singleton functionality moves from one module to another. Our algorithms must not assume the module requirement graph is acyclic.

If there are `require` cycles in any of the forked modules, each cycle will need to be audited independently, and potentially forked:

![fail-loopfork](/vgo/fail-loopfork.png)

These cycles may exist to manage code relocation and/or global state (`D@v1.0.0 <-> D@v2.0.0` is a likely candidate for these), or simply because two separate modules have interwoven logic. For example, there are project-level cycles between Kubernetes and Docker, but no package-level cycles.

In terms of the `A <-> Q` cycle shown, that might involve a shared subpackage in `A`.  That is, imagine package `Q` contains this file:

```go
package Q

import "A/cfg"

func DoThing(cfg cfg.Config) {}
```

And package `A` contains this file:

```go
package A

import (
	"A/cfg"
	"Q"
)

func doAThingQ() {
	Q.DoThing(cfg.Config{})
}
```

Both `Q` and `A` import `A/cfg`, so if `A` is forked, then `Q` must be as well, so that `Q'.DoThing()` can properly expect an `A'/cfg.Config`.

It's crucial to note a difference of kind between the fork of `C`, and the knock-on forks of everything else (`Dv1`, `Dv2`, `A`, and `Q`). The former exists for the intended purpose of forks: to create a new version timeline of `C` that won't break `A`. The others, however, exist solely to make the original fork work, and are more or less useless outside of that context. And it's not necessarily desirable that their version timelines be curtailed, either, so Frank now needs to keep track of new releases for all of the knock-on forks and merge them in manually. That, of course, goes double for releases with security fixes.

Sometimes, forking might be an adequate stopgap. But in some expected situations, forks will metastasize through the dependency graph. While a well-resourced organization may well be able to responsibly handle some forks here or there, the upper limit on meaningful forks is going to be relatively low, and knock-on forks will never be anything other than dead weight. 

Forking will necessarily always have a place in a developer's stable of strategies, but it is a drastic, harmful action that creates costs for everyone. It is dangerous - and more than that, simply unrealistic - for a system design to treat forking as a near-to-hand strategy.

## MVS and Experimentation

While the dominant example in this post relied on a gap between `C@v0.9.0` and `C@v1.0.0` to calm my nattering mind, all discussion to this point has essentially assumed that good-faith attempts at backwards compatibility are being made, and breakages are an exception. When we look at the experimental v0 range, though, that goes out the window, and these problems become far, far worse.

v0.x.x versions have no backwards compatibility guarantee at all, according to both the semantic versioning specification and - more importantly - common sense. The purpose of v0 releases is to allow for sharing during an initial experimentation period, _before_ the author fully understands what their code is actually trying to do. 

Go itself spent more than two years in this period. And while we hypothetically could just spend this time in cave, emerging once the software is "ready," actually figuring out what your API is all about usually entails getting feedback. So, you "release early, release often," hope that you pick up some users, and Once you've got your promises worked out, you're ready to roll your `v1.0.0`.

The crucial word there is "feedback". We already know it's important even for post-v1 code, but that's even more true in the experimental period. If people aren't actually using your software - that is, for some real, production purpose - before `v1.0.0`, then it's quite difficult to get the feedback you actually need. Without feedback, it's unlikely that you'll actually have figured out what the software is, and what its core promises are - typically prerequisites for a solid `v1.0.0`.

In this sense, the OAuth example given in the [semantic import versioning post](https://research.swtch.com/vgo-import) is subtly misleading, because it is atypical that there is no overlap between the software's authors, and the authors of the software's specification. Usually, the latter is a subset of the former, to the extent that any specification even exists. That's the case for Kubernetes, all HashiCorp project, and `sirupsen/logrus`, just to name a few.

When OAuth is the example in your mind, the existence of that spec makes it easier to forget about just how much uncertainty is involved in the creation of software. Getting to v1 seems like a question of ticking boxes, rather than exploring problems. You can get a long way on your own, but having users - that is, a community - is the only way to really be sure that your code isn't just completely stuck inside your own head.

So it's a problem that the semantic import versioning post contains this throwaway line:

> The idea here is that by using a v0 dependency, users are explicitly acknowledging the possibility of breakage and taking on the responsibility to deal with it when they choose to update.

In 2018, i imagine it probably qualifies as common knowledge that you're signing up for breakage if you rely on a module in v0, no matter what language you're working in. That's not news. What this is really saying, though, is that if you're using a v0 module, that's your problem. Vgo's not doing anything special to help. (We knew this already.)

The problem is, the primary risk doesn't actually sit with the person who made the choice, but with their dependers. We've seen this before, but one more time:

![fail-incompat](/vgo/fail-incompat.png)

The worst that Aparna faces in this configuration is an ephemeral failure, and her easiest move is to Ignore, followed by Declare. Deon is the one who's left with no easy options. Let's imagine he ran a `vgo get A@v1.0.0 Bv1.0.0`, expressly naming the versions he wanted:

![fail-diamond](/vgo/fail-diamond.png)

Now, despite the fact that Deon made an explicit choice to have only versions in v1, he's got a failure because Aparna chose to use `C` in its experimental period.

All of this amounts to a significant incentive to _never_ allow any v0 into your project's dependency graph, directly or transitively, when operating under MVS. And, if nobody wants to touch v0 modules, then it deprives them of the very feedback they need to develop properly into v1s with clear promises. Module authors will feel the need to rush to v1 simply to get attention, leading to a profusion of v2, v3, v4 as these premature v1s go through the evolutionary process that they should have in v0.

While no solver can actually fix true API-level incompatibilities, the type-checking behaviors gps2 would have can mitigate a lot of the problem here. Unlike human-declared rules, these checks can't be wrong, any more than Go's type system can be wrong - and they could still be refined by human rules, when API-level checking isn't sufficient. This would be the difference between night and day for the v0 range.

Of course, semantic import versioning is supposed to address all that - and if it were that easy, then it might be worth undermining the widely-understood semantics of v1. However, as we'll explore in the fourth post of the series, semantic import versioning is not a zero-cost abstraction. The less it happens, the better off the ecosystem will be.



## "Welcome to Go! Here's Your Yak"

When Russ first announced vgo, one of the early replies on the mailing list came from David Anderson [He noted](https://groups.google.com/d/msg/golang-nuts/jFPz5yZCPcQ/ToB5CpzSBAAJ) that a problem with the global constraint declarations in systems like dep or glide is how they effectively provided "authority without responsibility." Russ has echoed this sentiment in various ways, such as [the strawman Kubernetes YAML example](https://github.com/golang/proposal/blob/master/design/24301-versioned-go.md#build-control) that I referenced in the first post.

As we've seen here, MVS has the very same kind of problem. What it adds, however, is the reverse structure: "responsibility without authority."  That's the bit where, when the compatibility state of the ecosystem is in a degraded mode - and that is "ALWAYS," according to game theory, the adage that [complex systems run in degraded mode](https://blog.acolyer.org/2016/02/10/how-complex-systems-fail/), and Russ' own statements[^3] - it's your job to work to fix other peoples' problems, where "problem" is often "they didn't do enough work to keep up with everyone else."

In other words: "Welcome to Go! Now, shave this yak somebody left lying around."

This is why I find [the notion only code fixes are a permanent solution to incompatibility](https://github.com/golang/go/issues/24301#issuecomment-390766926) to be fundamentally uninteresting. Obviously it's true. And we'll get there, eventually. But we're a community, not a pool of interchangeable workers operated by a distributed scheduler.

Looking over the hamfisted incentives for developers, harmful externalities, and assorted additional failure modes, it's hard to see why it's worth giving up the automation that a solver provides us. The argument I've yet to hear from Russ is exactly why allowing SAT will _necessarily_ lead to unmanageable complexity growth. All I see here is yet another dependency manager that is set up to make people scared to update their dependencies - that is, scared to experiment, grow, and learn - because it's optimizing for the wrong thing.

In the next post, we'll focus in on the idea of compatibility. Turns out, it's a turtles-all-the-way-down problem.

[^1]: True minimum is readily knowable in exactly one situation: when a module references an identifier from another module's package, and that identifier was introduced in the version that is currently selected. Barring that, it's quite difficult to know if we've accurately identified true minimum.

[^2]: It's "indirect reciprocity" because Aparna's benefits derive from the environment - a compatible ecosystem - rather than any one single other actor. Module graphs don't fulfill the requirements for [network reciprocity](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3279745/pdf/nihms49939.pdf) because, while cycles are possible in the module graph, in practice it is overwhelmingly directed.

[^3]: From [the MVS blog post](https://research.swtch.com/vgo-mvs#upgrade-speed): "But I think in practice dependencies will move forward at just the right speed, which ends up being just the right amount slower than Cargo and friends." 'Just the right speed' is necessarily slower than 'everything updated all the time,' which means the ecosystem is in a degraded mode.
