# The chronic irritation of using Cargo with embedded Rust


## Cargo.

_**[Applause.]**_

_**[More applause. Plaudits to all involved, even.]**_

It's great, right? A build system that doesn't suck.
As long as you're writing code for desktop Rust, that is. 

Because on embedded, while probably still better than CMake, you're probably run into the limitations of the Cargo build system _really_ quickly. 

And while it's far from _unusable_, it's certainly _annoying_ in ways that you run into with relative frequency. 
You still get the benefits of having good package management, after all.

So, rather than sharp painful misery, it is thus better described as chronic irritation.

## Background: how an embedded Rust project is structured

To understand how the typical embedded Rust application differs from a typical desktop Rust application, we have to talk about how they're all structured.

A good example is the [cortex-m-quickstart](https://github.com/rust-embedded/cortex-m-quickstart/tree/ac02415275d0190a1a7aa730ec2b0bdf7c3ef88f), which I'd say is more representative of existing projects and easier to understand than Knurling's `app-template` for this purpose despite the former being deprecated in favor of the latter, but like, they're pretty similar overall.

It's still a binary application with a `main.rs`, but there's a couple more mandatory bits. 
Namely:

* a `memory.x` linkerscript that contains information about how the target chip's memory, both volatile and nonvolatile is laid out
* a `build.rs` that at minimum copies in the `memory.x` to `OUT_DIR` and signals its inclusion along with other relevant linkerscripts, e.g.
  `flip-link` or `defmt`. The `memory.x` typically gets picked up by something like `cortex-m-rt` that provides the prelude that sets up your stack/initializes statics/etc. 
   * `embassy` has the option of generating this `memory.x` for you.
* a `.cargo/config.toml` that typically contains three elements:
  * a runner argument containing the `probe-rs` incantation to flash and run the compiled binary
  * a `build.target` specifying the target architecture (e.g. `thumbv7em-none-eabihf` for Cortex-M4F)
  * rustflags specifying the use of `flip-link` and other linkerscripts
* a `.vscode/settings.json` that tells rust-analyer to stop trying to pass `--all-targets` into cargo invocations and failing miserably
* `#![no_std]` at the beginning of your `main.rs` along with `use [package] as _` imports to provide panic handlers and `defmt` impls

These elements are near-ubiquitous for embedded Rust projects that use the typical `cortex-m-rt` + `probe-rs` combo (or similar), regardless if they use embassy or RTIC or neither or both.  Some central project sets up the linkerscripts for you and you make some config adjustments to Cargo's behavior.

## Issue 1: the assumption that you can just build things on all architectures all the time

Turns out passing `--all-targets` into `cargo` doesn't work well for embedded projects, and it becomes a frequently disabled feature in embedded setups, along with other recommendations commonly seen [here](https://github.com/rust-lang/vscode-rust/issues/729).
This isn't particularly user-friendly and even I keep forgetting this.

### Workspaces

But more frustratingly is _workspaces_ and how they interact with targets.
Workspaces just kind of don't work well the moment you start mixing multiple architectures.
And this ends up as a pretty common use case if you're developing both the firmware for a device, a host application to talk with it, and a shared protocol crate. 

A minimal example can be found [here](https://github.com/jamesmunns/bad-workspace), but common issues end up being:

* `cargo check` tries to build the workspace for the wrong architecture
* feature unification ends up building features in ways that don't work together (e.g. both host-specific and device-specific features of a shared crate at once)
* general additional `rust-analyzer` LSP misbehavior

### `per-package-target`

What ends up happening in practice in response is that you [get directed to use `per-package-target`](https://users.rust-lang.org/t/cargo-workspace-members-with-different-target-architectures/122464) on nightly, but this too has some issues:

* [Even Better TOML doesn't support the unstable cargo schema](https://github.com/tamasfe/taplo/issues/38)
* [`supported-targets`](https://github.com/rust-lang/rfcs/pull/3759) is straight up a better approach, even if it's not implemented in Cargo,
  but because everyone agrees its way better there's no real attention given to `per-package-target` anymore so you have a pseudo-deprecated approach and a not implemented approach.

Who knows when this will all shake out, stabilize, or fizzle because of the (reasonable) concern crate maintainers might go overboard with the restrictiveness of `supported-targets`?
Maybe if there was actually money in hiring people to develop languages rather than praying the open source community will still exist off the back of VC-backed firms with ZIRPbux to burn, Cargo would have people work on this.

## Issue 2: feature flags are insufficient

So remember how every embedded Rust project has its own `.cargo/config.toml` and a `build.rs` that copies a `memory.x`?
This setup is great, as long as your binary crate is intended for not just one target architecture, but one target _microprocessor_. 

You might be like, well, why don't I add features to gate this? And sure. maybe you do.

But how are you going to change the `runner` argument in `config.toml` to not pass `--chip stm32h743aiix` instead of `--chip stm32h747` or to `probe-rs` or whatever? Oh, that's right. You can't. You _can_ at least make `build.rs` copy in a different `memory.x` to reflect the differences in memory layout, but now you're probably constantly twiddling the runner arguments for different boards. 
Or perhaps even different _revisions_ of the same board which use different MCUs for whatever reason.

```toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "probe-rs run --chip STM32H743ZITx --log-file /dev/null" # how do we change --chip?
```

I mean, you can work around this, e.g.

* shove all the logic into a different crate and then `use` it into binary packages for each board variant (clunky)
* use `cargo run --features [device] --cfg [device]_config.toml` and cover it up with...aliases? Shell scripts? Pain.
* leaving the `--chip` argument off of `probe-rs run` in config and using `cargo run -- --chip stm32h743zitx` instead

This still has limits, of course. What if you're building your project to run on both Cortex-M0 and Cortex-M4?
Those are different architectures, so how are you gonna manipulate the `--target` config?

You end up with pretty similar issues across the board, but most importantly, you end up having to couple your feature flags tightly with external Cargo configs that have to be specified externally and managed carefully, and LSPs like `rust-analyzer` typically only assume support for one of these configs at once. 

This also gets really annoying for crates shared between a microcontroller and a host that may have unit tests runnable on the host but features that will not compile on the host because they're pulling in something like `cortex-m` or a HAL crate behind a feature flag.
Most tooling assumes that your `test` and target architectures are one and the same, but they may not be.

Point is, you now have a lot of board specific config being managed out of Cargo, either by yourself manually editing files or by external tools and aliases. Or end up opening more VSCode windows for specific binary crates in your monorepo. Pain.


---- 

Also worth mentioning that apparently Embassy's STM32 hardware support crates briefly delayed a release because it exceeded crates.io's limit of **1500** features.
Most of those features are mutually exclusive because they denote a specific MCU.

Dread them, avoid them, mutual exclusive needs arrive all the same.

## How do we fix all this?

Fine. Okay, You report the lack of Cargo `cfg` configurability from feature flags [github.](https://github.com/rust-lang/cargo/issues/14314).
It gets closed as a duplicate of issue [#8170](https://github.com/rust-lang/cargo/issues/8170).

Except now that issue is getting proposed to get closed too, in favor of global feature flags...?
Which seem cool, but seem to have [sat in the drafts folder](https://internals.rust-lang.org/t/pre-rfc-mutually-excusive-global-features/19618) for a solid 2 years with people saying "yeah this seems cool" but with no other progress, and no _explicit_ mention of embedded microcontroller use cases.

So like, did we just get sent to the shadow realm? Maybe that's just the cost of a volunteer project -- you can't focus on everything at once.
And that's okay. 

-----

But, yknow, there's a lot of us, and the first step to fixing these issues is to start thinking about them and shaping what a better future should look like. I'm unsure if the suggested solutions in here make sense. Maybe what Rust needs is yet another massive, ambitious, tries-to-do-everything-but-is-half-baked-at-everything-because-it-assumes-everything-is-the-least-useful-common-denominator-and-thus-has-really-frustrating-bugs-and-limitations framework tool that wraps Cargo _specifically_ for embedded use cases, in the Rustacean tradition. 

Or maybe we need to invest more money into actual language and infra development.
(Who would ever want to do _that_, though? Won't the AIs just work around all those shortcomings for us?)

In the meantime, I would encourage Cargo maintainers to try writing some embedded Rust just for fun.
Go blink an LED with Embassy. Experience all the fun of programming without heap allocation.

And there's definitely more corner cases that haven't been mentioned here, but I figured it's worth talking about.