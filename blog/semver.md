# SemVer 1.0.0 Releases Are Undefined Behavior

or **`0.y.z` vs. `>=1.y.z` is absolutely meaningless in the existing ecosystems**

## Please read the whole thing before:

* accusing me of not knowing what semantic versioning is
* accusing me of not having read the SemVer spec because I mostly referred to it as SemVer
* being against encoding API breaks in versioning
* linking me https://0ver.org
* linking me a "relevant XKCD"
* sending me an AI generated summary of SemVer and `1.0.0` versioning practices

And yes, I hand-wrote this entire thing without AI.

## Preface

[Semantic versioning or "SemVer"](https://semver.org/) gives you a machine-readable way of encoding "this update makes API-breaking changes" or
"this update makes backwards-compatible additions to the API" or "this update is a bugfix patch and doesn't change the API surface"

This is a **(generally) good idea**, and influenced how more competent package management like Rust's Cargo works.
Whatever hellscape existed beforehand must've sucked. Is this library bump backwards compatible? Who knows?

I think that encoding this sort of information in your versioning should be the **bare minimum for _any_ versioning scheme.**

But I think that's about as good as SemVer gets, at least the way it's implemented in practice.
It is _only_ good at describing API compatibility within versions.
It says **nothing** about the maturity of the software, its update churn rate, how _severe_ breaking API changes actually are between versions, despite the attempts of both its authors and its implementors to try and ascribe such to it.

## SemVer (the spec) != SemVer (the practice)

The SemVer specification attempted to distinguish between "initial development" versions of software and "production-ready/stable" versions of software with a `0.y.z -> 1.0.0` transition point.

While the SemVer spec says that major version 0 has **no API stability guarantees:**

> Major version zero (0.y.z) is for initial development. Anything MAY change at any time. The public API SHOULD NOT be considered stable.

that's not how, say, Rust's `cargo` interprets it.
If you have a package with major version 0, it interprets crates `0.y.0` and `0.y.1` as being "semver-compatible" as having the same `0.MINOR` version to `cargo` means that they are "compatible."

However, it treats bumping the minor version of 0ver crates as a breaking change the same way going from `1.2.3` to `2.0.0` would be a breaking change.

A _lot_ of people do not seem to understand this, so please [read this section](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#default-requirements) before telling me about how Cargo thinks `0.1.1` and `0.2.0` are actually "semver compatible", because it doesn't, unless you are doing something really dumb and using `0` as your version specifier.

## The 1.0.0 release point threshold is Undefined Behavior

The SemVer spec itself does not say when a 1.0.0 release should happen. The FAQ, however, [recommends this:](https://semver.org/#how-do-i-know-when-to-release-100)

> **How do I know when to release 1.0.0?**
> 
> If your software is being used in production, it should probably already be 1.0.0.
> If you have a stable API on which users have come to depend, you should be 1.0.0.
> If you’re worrying a lot about backward compatibility, you should probably already be 1.0.0.

However, in practice, this is just a recomendation and not how software in Rust's packaging exist in practice.
Plenty of Rust crates exist in the world that are not `>=1.0.0` that are:

* used in production
* have a stable API
* worry a lot about backwards compatibility with each update with particular regard to how Cargo will pull dependencies in based on the typical practice of `crate-name = "MAJOR.MINOR.MINIMUM_PATCH_VERSION"`

To that end, the Rust community's general ideas of "stable" are such:

* If you're a `>=1.0.0` crate you are "stable".
* If you are "stable", that means [the public dependencies your crate has must also be stable](https://rust-lang.github.io/api-guidelines/necessities.html#public-dependencies-of-a-stable-crate-are-stable-c-stable)
  * ...but in practice, many widely-used crates aren't stable so that means either you aren't stable or you just ignore this guideline for interoperability reasons. The orphan rule exacerbates this.
* Having a "stable" API is some nebulous idea that your crate is "mature" and doesn't really plan on changing the API much afterwards. 

However, by the standards of that SemVer FAQ answer, **every crate uploaded to crates.io qualifies for a `>=1.0.0` release,** regardless of how one feels about the API.

If you're uploading to [crates.io](https://crates.io), that means:

* you intend for other people to pull this crate in via Cargo, which will then reason about said crate's versions with care for API compatibility
* said other people MAY use the crate in production, whether you like it or not
* said other people MAY come to depend on the published crate's API as a "stable API", even if from a developer side it is not fully out of the churn cycle yet.

This sounds a lot like "production use", "dependent API", and "backwards compatibility concern" to me.

## 1.0.0+ Is Not An Indicator Of API "Stability"

(In Rust, at least)

The SemVer FAQ encourages software past `1.0.0` to be conservative in issuing breaking API changes:

> **If even the tiniest backward incompatible changes to the public API require a major version bump, won’t I end up at version 42.0.0 very rapidly?**

> This is a question of responsible development and foresight.
> Incompatible changes should not be introduced lightly to software that has a lot of dependent code.
> The cost that must be incurred to upgrade can be significant.
> Having to bump major versions to release incompatible changes means you’ll think through the impact of your changes, and evaluate the cost/benefit ratio involved.

And there's this implication/cultural norm (which is sometimes outright stated in comment sections) that if you're past your `1.0.0`, you shouldn't intend to change your API in backwards-incompatible ways much, if ever. 

* Some projects recognize they can achieve this bar and thus bump themselves to `1.0.0`.
* Other projects don't feel like they can _ever_ achieve this and thus stay in the `0.y` range indefinitely...but still end up paying special attention towards backwards-incompatible changes anyway in determining whether to bump to `0.(y+1).0` or `0.y.(z+1)`. 
* The original author of SemVer, Tom Preston-Werner, thinks that [major version numbers are not sacred](https://tom.preston-werner.com/2022/05/23/major-version-numbers-are-not-sacred) and that there shouldn't be shame in having a high major version.

----

And ultimately, it seems that actual in-practice API "stability" isn't really correlated with being `0.y.z` or post-`1.0.0` at all:

* `log`, a Rust Project-maintained crate, has been `0.4.x` for nearly a decade. It has over a billion downloads.
* `libc` another Rust Project-maintained crate, has been at `0.2` since 2015. It has a [roadmap to 1.0](https://github.com/rust-lang/libc/milestone/1) that has been ongoing for nearly as long.
  * The implication, of course, is that the API needs to be deemed finalized to some standard to be 1.0 worthy.
* `zip` and `which` are at `8.y.z` and have reached major version 8 in the span of ~3 years.

It's hard to run desktop production Rust code without running into crates like `log` or `libc`, and they are some of the most stable crates in the entire ecosystem.

But the SemVer doesn't tell you that; going to the project pages does. 

Similarly, `zip` and `which` have evidently had more API churn than either of these crates; but this doesn't necessarily imply they are badly written or badly managed. Which is the implication that people like to make if your SemVer major version gets too high.

In ecosystems where package management systems let you upload pre-`1.0.0` software, SemVer alone **_cannot_** give you a guarantee on how mature the API actually is. You actually have to examine the project itself to make that determination.

To that end, having that distinction in the SemVer spec itself seems spurious.
Thus, we must either:

* accept SemVer as recommended by its FAQ/as Tom wants it, and conclude the entire ecosystem is wrong for allowing `0.y.z` software to be in wide distribution or dependency use
* accept the existence of `0.y.z` software in popular circulation, leading to there ultimately being no real signalling difference between a popular dependency versioning itself `0.10.z` or `10.y.z` beyond which ranges of versions they are compatible with.
  * ...after all, once again, SemVer didn't define the `1.0.0` release threshold; it just left it as as an FAQ recommendation.


As establishhed previously, a Rust crate being "semantically" versioned `0.y.z` vs. `(x > 0).y.z` is ultimately meaningless beyond an API compatibility range. All it boils down to is having two namespaces of versioning with slightly different semantics involving backwards-compatible API changes: one with two digits and one with three.

Thus, SemVer in Cargo:

* can't tell you which of the `0.10.z` or `10.y.z` packages are more mature
* can't tell you how big the API change between `0.10.z -> 0.11.0` versus `10.y.z -> 11.0.0` is
* can only tell you that `0.10.z -> 0.11.0` and `10.y.z -> 11.0.0` are API-incompatible.

But people love pretending SemVer can and should signal that additional information about API stability and maturity.

## So is the major number sacred or not?

If I were to go back in time to change SemVer, I would probably define the line at which a `1.0.0` release comes out as something like:


```
Software that is prepared for wide distribution MUST have a major version larger than 0
```

and adjust expecations such that it is the norm that all package distribution systems reject pre-`1.0.0` software from being uploaded.
To that end, I take Tom Preston-Werner's position that the major version shouldn't be sacred if you're doing SemVer for everything as Cargo does.

After all, when you distribute software to the wider public, you don't control if other people will use it or not.
You don't control whether or not its public API surface will be used to create a [spacebar heater.](https://xkcd.com/1172/) 
So let's just pick the conservative option, and semantically version our software as `>=1.0.0` and we get to use all three digits whenever we add things that don't break the public API.

## Splitting the major version into two components

But people don't really agree that the major version isn't sacred, apparently. 
Again, many projects like `rust-lang/libc` view `1.0.0` as a huge developmental milestone, and having big major version numbers seems both noisy to many people and an indicator of poor quality. 

But at the same time, the major number is supposed to also be just an API compatibility break number.
It was never intended to be sacred. I don't think you can have it both ways.

I think there's a good case to have a fourth "marketing" or "epoch" number. Sometimes you're wrapping some standard that's year-based, because it's used in the context of an environment that refreshes yearly, like F1 regulations:

```toml
formula-one-regulations = "2027.1.0.0"
```

To preserve API compatibility guarantees, you treat `2027.1` as a composite "major number"; for example `2027.1.0.0` would be incompatible with `2026.1.0.0` as `2027.1 != 2026.1`.
[Haskell's package versioning policy](https://pvp.haskell.org/) basically does this exact system, while also encoding API compatibility and breakage.

The other thing you can do with this is use this "marketing" number to signal when you reach some major developmental milestone; be it "this is the productionizable release" or "this is a new generation of this software that will be non-trivial to port to." You bump it purely as API milestones, and you don't have any expectations of previous generation software compatibility; hence why it's considered part of the "major version."

You could hack your way into implementing this into existing SemVer systems by just having an obscenely large major version, e.g.

```toml
formula-one-regulations = "20270001.0.0"
```

but that's ugly. It would work, though, and again, lets you both have a marketing-significant major version that also encodes API compatibility. In these yearly scenarios, it's usually not unreasonable to say that the 2026 version should be considered breaking relative to the 2027 version anyway.

I think for now, I'm probably going to end up taking this latter path when I publish yearly-dependent crates to crates.io.
There are other articles that talk about this, but I'm just leaving this here for now.

## Conclusion

This was largely inspired by [triggering a flamewar on /r/rust](https://www.reddit.com/r/rust/comments/1vjbcj9/i_kinda_hate_semver/) and then reading more into it.
