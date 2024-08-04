# Embedded Rust

It really shouldn't be good, but it's cooks anyway.


## The background

### The ESP32

Redux's first (and aborted second) products were written using ESP-IDF in C++. 
The codebase started life as C. And then we started writing C with classes to modularize everything. 

Oops, classes are a clunky abstraction that make it hard for subsystems to interact especially on initialization, let's rewrite it all as C++ namespaces.

The ESP32c3 is quite nice. It has 4MB QSPI flash so you can write to it while reading from it to execute code, which lets you re-zero the Canandmag encoder live without having to do all sorts of `burnFlash()` type shenangians which stall code execution.
It comes with an OTA stack because the MCU was meant for IoT devices that were supposed to be able to OTA themselves over HTTP. Except instead of HTTP, we were doing this over CAN. 

Thing is, the ESP32 platform has its limits. They don't have a ton of GPIO, lack a proper FPU, and the 8-bit ADCs leave a lot to be desired. Can't really make a motor controller or a gyroscope with one, at least not to the level we want.

However, when writing this firmware, we learned a lot. We learned about the importance of having infrastructure. 
We wrote a ton of Python that maintains protocol coherency between the ReduxLib vendordep, the Alchemist tuner, our DBCs and documentation, and the firmware itself.
It lets us prototype new concepts quickly while making sure that everything speaks the same language in five different places.

Problem is that generating code for Java and C++ vendordep is annoying enough. Codegen for the ESP32 was becoming increasingly overspecified to the platform and was a bit of a headache.


### The C++ Nitrogen firmware
Naturally, we initially started to write the Nitrogen motor controller firmware in C++. We used the Segger embOS RTOS that costs you $7500 per product to use...assuming you're selling it and not using it for evaluation. 
It wasn't bad and the debugging experience is better than `probe-rs` (more on that in a bit) but that particular iteration of the firmware was...Haunted and had tons of mysterious Heisenbugs. 
In theory we would be ahead of where we are now but realistically we'd have been bogged down trying to hunt down difficult-to-debug issues that likely came from memory incoherency issues.

## Rust itself

Embedded rust has decent potential. 

### Memory safety
I like slices.

I do wish there was a more ergonomic way of writing 
```rust
fn copy_from_array_ref(a: &mut [u8; 32], b: &[u8; 16]) {
    a[..16].copy_from_slice(b.try_into().unwrap())
}
```
like I know ths compiles to a single memcpy but it feels weirdly unergonomic for something where you _know_ the size of the slice at compile time.

I don't think it can really be changed at this point but I wish we could do more with statically checking known-size slices at compile time instead of letting users fat-finger slice sizes and finding out when it explodes at run-time.


### RTIC: the best "RTOS" 

[RTIC](https://rtic.rs) dares to ask: "what if we let the MCU's nested interrupt controller handle task scheduling for an RTOS?" The answer is good things.

Basically you take a bunch of your unused (and used) interrupts, assign them priorities, and the RTIC macro stack will generate interrupt handlers and NVIC manipulation code in a way that:
* eliminates data races and deadlocks
* expresses critical sections in ~3 instructions and zero wait (as software tasks and interrupts that depend on a shared resource are unable to preempt while said resource is held)

It also wraps shared state initialized from the init entry point in locking critical-section mutexes. rtic-sync also provides mutexes and MPSC channels that can be awaited on by software tasks but still interacted with from interrupts. The critical section mechanism can be expressed in ~3 instructions and neatly maps priority inversion to the Cortex-M NVIC. 

The end result is a neat, low-overhead way of dealing with global mutable state that also supports `async` in your software tasks. 
The way global state gets organized though tends to be very conducive to writing annoyingly tightly-coupled code though.

There are some downsides to RTIC though:
* Due to macros not reaching across file boundaries, if you want to define tasks outside of your main app macro you have to header-define them in an `extern "Rust"` block, which gets really annoying to synch
* Macros will never not feel like duct-tape for language problems
* If you forget to use the consumer of an MPSC channel in some task after init you can deadlock your firmware (footgun)
* I wish rtic-sync had semaphores


### Proc macros
Why are they so jank and why are they allowed to be so..."unhygenic", anyway? 
Why do I have to print-debug these things? If I screw up the syntax in a `quote!` half the time I can't even see the output with `cargo expand` anymore.
I often find myself wishing I could see the generated code from a macro but those intermediates only ever existed in rustc's heap.

...I'm still mad about what happened to that rustconf keynote. Compile-time reflection seems like a really good way to like, make higher-level constructs than raw syntax.
Proc macros are still better than C-style preprocessor macros but less than I think they should be.

To add insult to injury the fallout from what happened to the keynote resulted in one of the packages we use being unmaintained.


### Lack of async closures

**:(**

### const fn
is undercooked. please stabilize more things

### Hardware support
We're using two platforms that don't have publically available hal crates ([ok now one of them now does](https://github.com/guineawheek/gd32c1x3-hal)) so we're basically playing on hard mode. But we make do.
Probably spent way too much time doing this though.

### Iteration speed
Anecdotally, embedded rust doesn't really suffer as much from the issues that say, game dev does. Probably because we don't really deal with dynamically allocated memory or synchronization beyond critical sections and channels,
whereas gamedev has tons of both.

`cargo` makes it easy to organize your code such that you can isolate your state machines (e.g. for OTA decryption or flash management) and develop/test them on desktop against mock objects. cmake is always a headache.

End result? Much easier to develop and test your logic without always needing the hardware there. 

Also, anecdotally, embedded rust is easier to read. You get better data structures and better type information and less red squigglies that shouldn't exist because functional intellisense for C++ is a leading unsolved problem in computer science.

### Orphan rule
is stupid. also trait specialization pls

### Enums and repr(C)
are both great and terrible in Rust.
As algebraic data/result types they're great. Sure. I love them.

But the _**moment**_ you have to map them to a binary protocol of any sort you basically need to start reaching for proc macros. `repr(u8)` is half-baked duct-tape that everyone makes do with by throwing macros at so you can get `From` and `TryFrom` impls for what should be a core language feature.

Like, `num-enum` and `bitflags` is great and all but I wish Rust had better built-in stuff for mapping memory to/from fixed structures, especially ones with `repr(C)` (yeah yeah i know that C struct layout rules can be incredibly complicated).

Fun fact: most peripheral access crates (PACs) work by defining a `repr(C)` struct over the relevant registers in memory whose members have read/write functions that act on the bits of individual registers.

## cargo and rust-analyzer
is a lot like the rest of Rust: really good but missing some really annoying things

### nested workspaces
should be a thing when you're vendoring edited versions of crates. "But just exclude the workspace from the top-level workspace and edit the sub-package in a different VSCode window!" But I don't want to do that. That breaks my flow. I have a dozen vscode windows open at any given time and they're already a nightmare to manage.

### build targets
This is extremely annoying because I want to be able to have code in a shared `canandkitchen` library that I write for embedded but can test parts of it on desktop. This is important because my tests will do things like use `std` but the actual crate itself won't. I often find myself twiddling `Cargo.toml` just so I can have rust-analyzer work accurately for what I'm working on at the current moment, but while I can make `cargo` aliases that will build and test correctly, rust-analyzer broadly assumes you're only worrying about one compilation target and one default feature set at a time.

Sometimes I need the `thumbv7em-none-eabihf` target enabled because parts of `canandkitchen` that are feature-gated will use things from RTIC or a particular HAL crate. /r/rust commentators will probably say "split your crate into more crates that are more specialized" but these additional bits are often tightly-coupled to the otherwise platform-agnostic code, that, again, are feature gated. I see little reason to atomize code that should logically go together and sacrifice readability because of tooling limitations.

The best solution I can seem to find is to have a separate executable crate for desktop tests that takes in my common code library as a path dependency and open it in a new vscode window. I hate that for similar reasons why I hate the nested workspaces solution.

Really, I wish I could just be like "this is the feature set and target platform I want for `#[test]` and this is the feature set and target platform i want otherwise"


## Doing math in embedded rust

### thumbv7em-none-eabihf
is a weird target. It's one of the few targets in existence that people care about that often only does single-precision floating point. And the rest of the Rust ecosystem does not really vibe with that.
But it's a very common target if you're dealing withs something STM32-shaped that's running a Cortex-M4 or M7. 

Many libraries will assume the target platform has doubles and thus just not be performant. Or they'll assume there's some sort of simd going on. These work for desktop platforms and wasm but don't for the poor Cortex-M4F.

### The Cortex-M4F FPU

is kinda quirky. It has a hazard where if you have two instructions where the later instruction depends on the result of the first instruction, it has to stall for a CPU cycle. 
This means it's _generally_ optimal to use fused-multiply-add as it takes the same number of clock cycles but benefits in code density. 

Despite being a mostly "in-order" core, division and square root have an optimization where  it gets calculated during the instructions between when the vsqrtf or div instruction was issued and when the result is actually used, stalling on hazard until the operation completes.

I haven't stared long enough at the assembly to tell you if LLVM actually orders math-heavy code in a way that is cycle-optimal, and my gut says this is some sort of NP-hard problem so you'd need some sort of solver. I do think it'd be interestng to look at though.

### `#![feature(core_intrinsics)]`
So fun fact: you can't actually use vsqrtf or fmaf without using intrinsics. Which are an unstable feature...

To the compiler's credit, rustc will produce the correct variation of fused-multiply-add depending on the signedness of the arguments, so `core::intrinsics::fmaf32(x, y, -z)` will produce a `vfms.f32` instruction rather than a `vfma.f32`. 

### libm and trignometry - unscientific benchmarks

is modeled after the musl C libm library. It values accuracy above all else, including performance. 
Many of the routines were written in the 90s by Sun Microsystems presumably for some forgotten-about Unix or BSD running on a SPARC and translated to Rust.

The trig and sqrt routines in particular are _awful_ for performance.
They will do things like use 64-bit width doubles and integer types for f32 operations and just _tank_ performance to unusable levels if you're running with any sort of frequency.

Trig is kinda needed to do park/clark transforms so you can do field-oriented motor control. Also useful to convert quaternions to euler angles to pull out yaw. I still can't get over how we named a rotation matrix after someone.

We found that the best way to go was just straight-port sin/cos/atan2 routines from ARM's CMSIS-DSP to Rust and call it a day. 
In our unscientific internal benchmarks, CMSIS-DSP's combined `sin\_cos` routine is **50x faster than libm** while having accuracy within a magnitude of libm.

We also looked at `idsp` and `micromath` and found that:
* micromath was 1.5x faster than CMSIS-DSP but **14000x** worse error than cmsis-dsp 
* idsp was 3x faster than CMSIS-DSP but had much higher rmse, but idsp's sin/cos routine is entirely implemented in integer math and is likely not really intended for our usecase.

```
sin_cos performance (conv overhead subtracted out):
idsp  sin_cos:   14205525 ns (197x faster)
umath sin_cos:   34953984 ns (80.0x faster)
cmsis sin_cos:   54901192 ns (50.9x faster)
libm  sin_cos: 2796446500 ns (libm)

sin_cos accuracy compared to uniform points on desktop f64::sin_cos():
idsp      rmse: 2.331236156716059 (5.5e+7x worse than libm)
micromath rmse: 0.0005966708156419408 (14000x worse than libm)
cmsis-dsp rmse: 0.0000000618519595006204 (1.4x worse than libm)
libm      rmse: 0.000000042181551381723033

atan2 performance (conv overhead subtracted out):
// umath   atan2:    77000 ns
// cmsis   atan2:   245525 ns
// idsp    atan2: 46133091 ns
// libm    atan2: 99952575 ns

atan2 accuracy compared to f64::atan2():
micromath rmse: 0.0019775500686665076 (32000x worse than libm, NaNs at 0/0)
idsp      rmse: 0.0000037610584004532416 (61x worse than libm)
cmsis-dsp rmse: 0.0000001126557323795005 (1.8x worse than libm)
libm      rmse: 0.00000006121479384946725

```

The script:
```rust
fn bench() {
    let start = DwtSystick::now();
    let mut prevent_opt = 0_f32;

    // let (sin, cos) = libmoo::sin_cos_f32(theta);
    // let (sin, cos) = libm::sincosf(theta);
    // let (cos, sin) = idsp::cossin((theta * IDSP_CONV) as i32);
    // let (sin, cos) = (theta, theta + 0.0001);
    // let (sin, cos) = micromath::F32(theta).sin_cos();

    //const CONV: f32 = (core::f32::consts::TAU / 65536.0_f32);
    const CONV: f32 = (core::f32::consts::TAU / 256.0_f32);
    const IDSP_CONV: f32 = core::i32::MAX as f32 / core::f32::consts::PI;
    const INV_IDSP_CONV: f32 = core::f32::consts::PI / core::i32::MAX as f32;
    for i in -128..127 {
        for j in -128..127 {

            let sin = (i as f32) * CONV;
            let cos = (i as f32) * CONV;
            //let atan2 = idsp::atan2((sin * IDSP_CONV) as i32, (cos * IDSP_CONV) as i32) as f32 * INV_IDSP_CONV;
            //let atan2 = micromath::F32(sin).atan2(micromath::F32(cos)).0;
            //let atan2 = sin + cos;
            let atan2 = libmoo::atan2f(sin, cos);

            prevent_opt += atan2;
        }
    }
    let end = (DwtSystick::now() - start).to_nanos();

    defmt::println!("bench: {} ns, opt={}", end, prevent_opt);
    defmt::println!("Benchmark end, stalling cpu.");
    loop { }
}
```

Translating C routines from cmsis-dsp delivers performance not _too_ far off of micromath

The one major downside of cmsis-dsp is that it uses a rather large 2Kib LUT, so if flash is at a premium that may give some pause. But our motor controller silicon had plenty so we didn't really worry.


### micromath

micromath is an oft-cited solution for libm's speed issues. 
It's written by a guy whose primary background is writing cryptography code in Rust.
It's full of interesting mathematical tricks to try and make things faster but outside of the libm's extremely slow sin/cos/tan functions it's [debatable the extent it provides a speedup over libm](https://github.com/tarcieri/micromath/issues/97#issuecomment-1570995506).

I personally think the project is somewhat misguided as it will emphasize theoretical speedups at the cost of significant accuracy; the 2-3x speedup you get over other solutions comes at five or more magnitudes worse accuracy.

### nalgebra 
nalgebra advertises itself as a no_std no-nonsense linalg library. And it seems to be pretty reasonable at this...mostly. 
It depends on the simba crate for it to figure out the optimal use of SIMD instructions for the platforms its used on which works well for desktop x86 but picks `libm` for embedded.  Oops!

Rather than fix this, since we weren't doing anything fancy like eigenvector or spectral decomposition I just elected to hand-write quaternion and affine vector/matrix operations for the pretty limited sizes of vector and matrix we cared about for gyroscope computation. Doing so yielded a 100% speed improvement in our core gyro code. The core now only spends a quarter of its time doing math rather than half of its time doing math.


## HALs, PACs, and embedded-hal
it's very STM32-brained. probably because the most actively-developed HALs are for STM32s. 
PACs are nice because they give you a zero-costish-abstraction over MMIO registers. Instead of worrying about the bit manipulation required to read or write some 5-bit segment of a register you can just refer to that register by name and use `modify` or `read` to deal with it. 

I don't get why people say that C is "better" because it's "closer to hardware" because like, PACs also let you twiddle the same registers?? 
In a way that's infinitely more readable?

There's one footgun though. `write` overwrites an entire 32-bit register while `modify` is what you actually typically want when you just want to flip a few select bits.


## defmt, probe-rs, and debugging Rust
They're...okay...ish. 

The defmt-rtt/probe-rs stack feels really fragile and if the RTT block ever gets corrupted it's basically over.

An easy mistake to make when developing for embedded from higher-level software is the assuption that you can always handle errors by just giving up/exiting the processor/hanging the device to preserve "correctness"; however doing so could quite literally potentially set a device on fire if it's in the wrong state. What is correct for a SaaS app is not necessarily correct for something managing power electronics and/or a robot actuator.

Segger Ozone is very mature; it is seemingly able to operate under high levels of EMI interference with SWD, lets you watch and graph variables, and lets you dump the current state of peripheral and CPU registers to CSV files. **No real affordable equivalent seems to exist for Rust.** Or you could pay an arm and a leg to Lauterbach.

Also probe-rs doesn't seem to really like it if you set your `memory.x` FLASH segment to _not_ start at `0x0800_0000` (e.g. because you're leaving room for a custom bootloader). I ended up just editing `build.rs` such that I just have a `memory_dev.x` and a `memory_release.x` where `memory_dev.x` is direct-flash to a board while `memory_release.x` must be OTA'ed on via the bootloader to function. 
