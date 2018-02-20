---
title: Thoughts on vgo and dep
date: "2018-02-20"
---



It's an odd day for me, today.

Russ and i have been meeting weekly since December, discussing what has evolved into the vgo prototype that [he's announced publicly today](https://research.swtch.com/vgo-intro). There's a lot to be excited about there:

> This proposal keeps the best parts of `go get`, adds reproducible builds, adopts semantic versioning, eliminates vendoring, deprecates GOPATH in favor of a project-based workflow, and provides a smooth migration path from `dep` and its predecessors.

These are excellent goals. They cover a lot of what the Go community has been clamoring for, and i'm glad to see them front and center in the official plan for the toolchain. i am concerned, however, about the specific plan to realize them - vgo - in the form it currently exists. While i have found agreement with Russ on the great majority of things in this problem space, including some of dep's shortcomings - for example, dep currently provides users with some dials that are too expressive, resulting in unnecessary confusion and footguns - there are still some foundational points on which we differ.

At the heart of our disagreement is whether the expressiveness reductions vgo must impose in order to make its core algorithm work will yield an environment that is bearable for Go developers. As with many things in package management, this is difficult to address with precision, because the proper answer depends on amorphous questions about what sort of frictions felt by users are worse, and what level of effort from package authors is reasonable.

i also have process concerns. vgo, as currently conceived, is a near-complete departure from dep. It was created largely in isolation from the community's work on dep, to the point where not only is there no shared code and at best moderate conceptual overlap, but a considerable amount of the insight and experience gleaned from dep as the  "official experiment" is just discarded.

Now, maybe the benefits of vgo's model will be so profound that these losses, and the difficulties of more experimental churn will be justified. i can't prove that *won't* be the case. And, if it does turn out that way, it would be stupefyingly hypocritical of me to protest, having spent the last year and a half asking the authors of other Go package management tools to gracefully bow out in favor of dep. All i can say for sure is that i'm sorry: to expose the community to yet more churn, and to anyone who feels like i've misled them about the process, and *especially* to dep contributors, who may find this announcement to feel rather like a punch in the gut. For whatever it's worth, none of that was ever my intent, and this is not the path i believed we would be walking.

i am writing more detailed documents about my concerns, and i will start publishing the technically-focused ones soon - hopefully next week. But, this is vgo's first taste of sunlight, so i don't want to go too far into these concerns today. Even without me piling on, it's going to catch plenty of flak just for being so different - and it's a genuinely interesting proposal, deserving of breathing room. At the very least, the algorithm (Minimal Version Selection, MVS) is a significant find: it is a simple (in the Big-O sense) and to-my-knowledge novel algorithm for version selection, *and* directly mirrors a powerful design principle for managing the evolution of software (renaming instead of breaking). If the history of computer science teaches us nothing else, it would be that such symmetries are often indicative of something noteworthy.

To that end, i gave a talk at FOSDEM earlier this month that touched on the (handwaving-handwaving) future of package management. Afterwards, one of the attendees asked me what i thought about specifically designing languages in a way that is friendly to package management. The properties entailed by MVS immediately sprang to mind. Which is to say: i am certain that MVS will be an important touchstone in my package management thinking going forward, independent of whether we do ultimately find the tradeoffs in this initial version of vgo to be too costly for the Go ecosystem's particular needs.

Having reviewed the vgo documents for a little while now, I think there are some unproductive paths that discussions could take, which I'd like to pre-empt here:

* vgo is, in my view, neither obviously right, nor obviously wrong. If it appears to be either to you, then either you've missed something, or i have (quite possible!).
* To the extent that they are possible, direct, experiential comparisons between vgo and dep as they exist today are certainly important, but it's equally important not to jump to conclusions on the basis of them. What really matters is, given the constraints in each model, what is *eventually* possible - that is, what kind of Go ecosystem and experience they might give rise to.
* i am certain folks will have the impulse to gut-compare vgo and dep as being more or less idiomatic Go. (Hint: vgo will win: it approaches a problem by intentionally omitting expressiveness allowed in other languages in pursuit of simplicity). But that's the start of a discussion - [not the end](https://en.wikipedia.org/wiki/Representativeness_heuristic).

Finally, a word about dep's future.

The publication of the vgo prototype doesn't change any of our immediate plans for dep, nor does it change dep's status as the recommended tool. Many of the priorities we have been targeting in dep for months are ones that - if a binary choice is made to change vgo's model - will suddenly become quite salient again. If that choice is not made, then dep (or an adapted successor) will likely need to live on in a community space, operating atop the new vgo. Either way, though, migrating to dep now is still likely to yield you the easiest path to vgo later, and ongoing improvements to dep will help dep's users in the immediate term. And our present pain matters.

i will continue to work on dep as i have for the last couple years - in my "spare time" (lol). i am also - to be fully transparent -  being contracted by Google for work on vgo. Specifically, the contract covers a thorough technical review of vgo, ongoing meetings with Russ, and creating automated migration tools that map dep, glide, govendor, and godep to vgo. i have taken care to ensure that the work specified in this contract does not incentivize me towards any particuar technical outcome, but rather focuses on things that Russ and i both agree are important for this next phase of experimentation.

The package management for Go space continues to be nothing if not…interesting! i'd encourage everyone to curl up with your favorite hot beverage and check out Russ' posts this week about vgo.