# Footguns with Embedded Rust and Memory-Mapped I/O

## Update:

Go read [this post](https://old.reddit.com/r/rust/comments/1gcz2ni/footguns_with_embedded_rust_and_memorymapped_io/lty8yo1/) by /u/AlphaModder as well. These problems are not unique to Rust but are something to be aware of more broadly.

## Background

Let's say you've read all the puff pieces that big companies are putting out about and decide "hey, let's write some Rust code for our embedded projects!"

So you do, but you first want to benchmark some math functions that you want to run on your dinky little ARM Cortex-M4F on your [STM32G431](https://www.st.com/resource/en/datasheet/stm32g431rb.pdf) that you found from searching by "most in stock" on LCSC. 

More precisely, you want to count the number of cycles that, say, some trig function takes. 
On a Cortex-M chip you can accomplish this with the help of a peripheral called the [Data Watchpoint and Trace Unit](https://developer.arm.com/documentation/ddi0337/h/data-watchpoint-and-trace-unit/dwt-functional-description) henceforth referred to as the DWT.

The DWT exposes a register called `CYCCNT` which is a 32-bit unsigned integer that counts how many clock cycles have passed, rolling over to zero after 2^32-1 cycles.  
You read from `CYCCNT` by issuing a read to the "physical" memory address `0xE0001004`, and the helpful maintainers of the [`cortex-m`](https://crates.io/crates/cortex-m) crate have already built an elegant way to do this via [`cortex_m::peripheral::DWT::cycle_count()`](https://docs.rs/cortex-m/0.7.7/cortex_m/peripheral/struct.DWT.html#method.cycle_count) function which issues a [`core::ptr::read_volatile`](https://doc.rust-lang.org/core/ptr/fn.read_volatile.html) to that address.

## Benchmarking woes

So you (naively) write up the following benchmark function:

```rust
use core::sync::atomic::{Ordering, compiler_fence};
fn run_bench() {

    compiler_fence(Ordering::SeqCst); // try and guarentee start happens before end
    let start = cortex_m::peripheral::DWT::cycle_count();
    compiler_fence(Ordering::SeqCst);

    function_under_test();

    compiler_fence(Ordering::SeqCst);
    let cycles = cortex_m::peripheral::DWT::cycle_count() - start;
    compiler_fence(Ordering::SeqCst);

    defmt::println!("Cycle count of function: {}", cycles);
}
```

And you run it, except you get this:

```
Cycle count of function: 1
```

One cycle? What? No. This can't be. 

So you disassemble the binary and you get something like this:
```arm
; [function_under_test code here]

ldr r0, =0xE0001004
; let start = ...
ldr.w r1, [r0]
; let end = ...
ldr.w r2, [r0]
subs r2, r2, r1
```

In Rust terms, the compiler basically gave us:
```rust
function_under_test();
let start = cortex_m::peripheral::DWT::cycle_count();
let cycles = cortex_m::peripheral::DWT::cycle_count() - start;
```

which is Not what We Wanted At All.


But then you hear about [`core::hint::black_box()`](https://doc.rust-lang.org/beta/core/hint/fn.black_box.html) which is touted to Solve This Problem by making the compiler Extra Pessimistic, so you wrap your `function_under_test()` call in it and lo and behold it Works.

Mostly. 

## black_box isn't enough? Or is it?

One of the functions you're testing is a bit longer. You expect it to take about 200 cycles based on some benchmarks that sum the time taken by multiple calls and averaging the numbers. 

With your newfound knowledge of black_box, you now have a new benchmark function, running on the latest `nightly-2024-10-26` and using `opt-level=z, lto=true, codegen-units=1`:

```rust
use core::sync::atomic::{Ordering, compiler_fence};
use core::hint::black_box;
fn run_bench() {

    compiler_fence(Ordering::SeqCst); // try and guarentee start happens before end
    let start = cortex_m::peripheral::DWT::cycle_count();
    compiler_fence(Ordering::SeqCst);

    black_box(big_function_under_test());

    compiler_fence(Ordering::SeqCst);
    let cycles = cortex_m::peripheral::DWT::cycle_count() - start;
    compiler_fence(Ordering::SeqCst);

    defmt::println!("Cycle count of function: {}", cycles);
}
```

Except when you run it, you get:

```
Cycle count of function: 60
```

Wait, that's too good to be true, right? 
So you open up the disassembly again and lo and behold yup it's too good to be true:
```arm
; [first half of function_under_test code here]
ldr r0, =0xE0001004
; let start = ...
ldr.w r1, [r0]

; [second half of function_under_test code here]

; let end = ...
ldr.w r2, [r0]
subs r2, r2, r1
```

Despite the usage of `black_box`, the Rust compiler has decided to interleave the start of the timing measurement _into_ the code under test.

In Rust terms, we basically got:
```rust
first_half_of_big_function_under_test();
let start = cortex_m::peripheral::DWT::cycle_count();
second_half_of_big_function_under_test();
let cycles = cortex_m::peripheral::DWT::cycle_count() - start;
```

Still not what we want.

---

Update: 


After some flailing around, the only solution you seem to find that _consistently_ produces the correct placement is to write the first instance to a `static`:

```rust
use core::sync::atomic::{Ordering, compiler_fence};
use core::hint::black_box;
use core::cell::UnsafeCell;

// SAFETY: presumably only a single thread is running run_bench from a single entry point
static mut START: UnsafeCell<u32> = UnsafeCell::new(0);
fn run_bench() {

    compiler_fence(Ordering::SeqCst); // try and guarentee start happens before end
    unsafe { *START.get_mut() = cortex_m::peripheral::DWT::cycle_count() };
    compiler_fence(Ordering::SeqCst);

    black_box(big_function_under_test());

    compiler_fence(Ordering::SeqCst);
    let cycles = cortex_m::peripheral::DWT::cycle_count() - unsafe { *START.get() };
    compiler_fence(Ordering::SeqCst);

    defmt::println!("Cycle count of function: {}", cycles);
}
```

The compiled assembly shows that the start count gets written to the memory address of `START` before the function under test consistently, presumably because the Rust compiler realizes that the `static mut` could be mutated from different threads and thus can't make any rearrangements. 
But is that really a guarentee you want to rely on?

## So like, why _does_ this happen?

We sprinkled [`compiler_fence`](https://doc.rust-lang.org/core/sync/atomic/fn.compiler_fence.html)s everywhere, why didn't they do anything?

Well, they kinda did, but only to the extent that they ensured that `start <= end` by ensuring the calculations of `start` and `cycles` were correctly ordered relative to each other, given they are both `read_volatile`s.

It doesn't guarentee anything about reordering anywhere else, including **before, into, or after the function we are trying to test.**

The core of the issue is that **rustc doesn't (and can't) understand that the number of cycles of execution affects `CYCCNT`.** 
It thinks you're just doing a memory operation on something like RAM where as long as `function_under_test()` doesn't touch that address, reordering the memory reads is fine. 
But the `DWT` peripheral _isn't_ in RAM, it's a peripheral that happens to be accessed through memory load/store operations. It's being touched by forces outside the typical memory model. 
It breaks invariants that you might expect of RAM, but isn't that just how all hardware peripherals work?

I'm not a language designer, but perhaps we need some sort of `code_fence` that prevents reordering of calls past it. I don't know.

## Conclusions and broader implications

Now this is a bit of a corner case that you really only run into trying to benchmark code on an embedded platform, especially since we're only taking reads of a peripheral that just counts upwards, and because for places that ordering _does_ matter, most MMIO peripherals will require multiple volatile reads/writes to peripherals such that `compiler_fence` is sufficient.

But there are some worrying implications here for all of embedded and low-level development in Rust not just on these tiny microcontrollers but also for things like Linux kernel driver development because memory-mapped I/O accesses are generally considered the standard way to access hardware peripherals.

It seems less than ideal to have to look at disassembly to verify that your MMIO reads are happening in the right place all the time though, and I do think some way to do code fences is necessary. 
Maybe the solution already exists in some RFC and I haven't looked hard enough, but if it does, please let me know.

But for now, I guess I will just have to use statics and make sure the compiler isn't screwing me over.


## Shameless self-plug

If you liked what you are reading and know of opportunities, I am looking for full-time embedded/systems jobs in the United States. I have experience shipping hardware products using C++ and Rust firmware and Python and Java for testing and user-facing APIs. If you are interested in hiring me, send me a line at `guineawheek` AT gmail (.) com

This whole corner case was also discovered by me trying to benchmark a series of math libraries for the Cortex-M platform, and I also do intend to publish the results of that at some point as well.

(Apologies for the less-than-optimal CSS, web design is not my strong suit.)