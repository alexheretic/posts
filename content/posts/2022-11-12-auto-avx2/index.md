---
title: "Getting rustc to use AVX2 SIMD"
---
SIMD roughly means CPU instructions that do multiple things at once. Making use of them can make algorithms faster. The rust compiler actually uses them automatically, but it _doesn't_ for newer SIMD instructions like SSE4.2 & AVX2.

| SIMD Feature | Coverage |
| --- | --- |
| SSE2 | 100% |
| SSE4.2 | 99.1% |
| AVX2 | 89.2% |
| AVX512F | 8.73% |

_Steam survey October 2022_

Rustc can't use them by default because not every CPU will work with them. But these days most CPUs _do_ support these newer instructions and so certain tasks can be a **lot** faster for a lot of people.

## What code can be optimised?
Without a good understanding of SIMD it can be hard to know what kind of tasks should benefit, but at a high level what I look for is **loops with maths in the middle**. If we're looping over a load of data and doing maths there's a chance this could be vectorized, which means the data gets packed together and requires lower total number of SIMD instructions to reach the same result.

For example my crate [ab_glyph_rasterizer](https://github.com/alexheretic/ab-glyph/tree/main/rasterizer) rasterization logic I know is an expensive operation. This involves drawing a font glyph's outline onto a 2d coverage grid. There are some loops with maths in the middle there so _can rustc make it faster on my CPU?_

## Benchmarking
My first step to optimising code is to write a benchmark. Optimising is _hard_. A benchmark can tell you if an optimisation has actually worked. This is important as optimisations often bloat the code and make it less readable in the name of performance, so you really should be proving that the performance has improved!

I already had some benches for _ab_glyph_rasterizer_ so lets fire one up using default rustc settings.

```sh
$ cargo bench --bench rasterize rasterize_outline_ttf_biohazard -- --save-baseline default

rasterize_outline_ttf_biohazard
        time:   [18.383 µs 18.385 µs 18.388 µs]
```
![](biohazard-180px.png "biohazard glyph")

So rasterizing a 294x269px biohazard glyph on my 5800x takes **~18.4µs**.


> Aside: Writing benchmarks is also hard. I ran mine 4 times and got 19.3µs, 18.8µs, 18.5µs, 18.4µs so I'll need to take the noise into account before I start celebrating a win that was just noise.

The cool thing about having a benchmark is we can start investigating if SIMD is going to help us **without writing any clever code**.

## Benching with target-cpu=native
I can tell rustc to use all the SIMD instructions my CPU supports and benchmark again.

```sh
$ RUSTFLAGS='-C target-cpu=native' cargo bench --bench rasterize rasterize_outline_ttf_biohazard -- --baseline default

rasterize_outline_ttf_biohazard
        time:   [14.224 µs 14.242 µs 14.264 µs]
        change: [-22.094% -21.920% -21.752%] (p = 0.00 < 0.05)
```

We can draw the same glyph in **~14.2µs** now. A nice speedup for no extra code! This is an indication that the code we're benchmarking could benefit from auto-vectorization.

We can also try to generalize a bit. Lets try just enabling AVX2 since I know that's one of the newest SIMD features my CPU supports.

```sh
$ RUSTFLAGS='-C target-feature=+avx2' cargo bench --bench rasterize rasterize_outline_ttf_biohazard -- --baseline default

rasterize_outline_ttf_biohazard
        time:   [14.412 µs 14.419 µs 14.427 µs]
        change: [-21.843% -21.726% -21.613%] (p = 0.00 < 0.05)
```
So we get pretty much the same benefit by simply enabling AVX2. 

But we're not done. If I compile, for example, my game with `RUSTFLAGS='-C target-feature=+avx2'` it will be faster. But it will also stop working for ~11% of my game's players, which is obviously not ok. We want to enable AVX2 for CPUs that support it and use the old code for everyone else.

## Targeting the functions to auto-vectorize
As a first step to runtime enabling AVX2 I started looking for particular functions that I'd like to auto-vectorize. I found [`Rasterizer::draw_line`](https://github.com/alexheretic/ab-glyph/blob/65064cf8d21affe27cce0e23535bc9a8cb02e8a4/rasterizer/src/raster.rs#L97) which is the core fn for this task and it has loop with tons of maths inside.

```rust
impl Rasterizer {
    pub fn draw_line(&mut self, p0: Point, p1: Point) { 
        /* maths-loops */ 
    }
    ...
}
```

We can instruct rustc to compile this using AVX2.

```rust
#[target_feature(enable = "avx2")] // doesn't compile!
pub fn draw_line(&mut self, p0: Point, p1: Point) {
```
This doesn't work because:
> `#[target_feature(..)]` can only be applied to `unsafe` functions

Actually that makes sense, after all it won't work on all CPUs. Lets just do it anyway.
```rust
pub fn draw_line(&mut self, p0: Point, p1: Point) {
    unsafe { self.draw_line_avx2(p0, p1) }
}

#[target_feature(enable = "avx2")]
unsafe fn draw_line_avx2(&mut self, p0: Point, p1: Point) {
    /* maths-loops */ 
}
```

Now we've enabled AVX2 for this function we can bench again without any RUSTFLAGS.
```sh
$ cargo bench --bench rasterize rasterize_outline_ttf_biohazard -- --baseline default

rasterize_outline_ttf_biohazard
        time:   [14.273 µs 14.276 µs 14.280 µs]
        change: [-22.566% -22.456% -22.352%] (p = 0.00 < 0.05)
```

So indeed this function is the place to be SIMD-ing! We're still not there yet though since the change _still_ breaks any non-AVX2 CPU.

## Runtime detection
We could wrap this optimisation in a compile time feature. But it isn't usually very useful to do so. All usage of this dependency would have to wire up the feature and ultimately something like my game would need a single binary to work for many different CPU feature levels.

What we want is runtime detection. We can do that with **[std::is_x86_feature_detected](https://doc.rust-lang.org/std/macro.is_x86_feature_detected.html)**.

```rust
pub fn draw_line(&mut self, p0: Point, p1: Point) {
    if is_x86_feature_detected!("avx2") {
        unsafe { self.draw_line_avx2(p0, p1) }
    } else {
        self.draw_line_scalar(p0, p1)
    }
}

#[target_feature(enable = "avx2")]
unsafe fn draw_line_avx2(&mut self, p0: Point, p1: Point) {
    /* maths-loops */ 
}

fn draw_line_scalar(&mut self, p0: Point, p1: Point) {
    /* the same maths-loops */ 
}
```

Now we're actually getting somewhere. Our code is actually safe so should work everywhere.
```sh
$ cargo bench --bench rasterize rasterize_outline_ttf_biohazard -- --baseline default

rasterize_outline_ttf_biohazard
        time:   [14.415 µs 14.419 µs 14.424 µs]
        change: [-21.784% -21.665% -21.553%] (p = 0.00 < 0.05)
```

Benching the code shows we're still getting the benefit.


## Inlining
One issue with this version however is the bulk of my code, the loops, has been duplicated. So there's a bunch more code. We can fix this fairly easily with inlining.

```rust
pub fn draw_line(&mut self, p0: Point, p1: Point) {
    if is_x86_feature_detected!("avx2") {
        unsafe { self.draw_line_avx2(p0, p1) }
    } else {
        self.draw_line_scalar(p0, p1)
    }
}

#[target_feature(enable = "avx2")]
unsafe fn draw_line_avx2(&mut self, p0: Point, p1: Point) {
    self.draw_line_scalar(p0, p1)
}

#[inline(always)]
fn draw_line_scalar(&mut self, p0: Point, p1: Point) {
    /* maths-loops */ 
}
```

This did look kinda funny to me the first time. The implementation of `draw_line_avx2` is just calling `draw_line_scalar`. It looks a bit pointless from a logical point of view, but this is for the compiler rather than for us. Note `#[inline(always)]` is important because we need rustc to duplicate the compilation to get 2 compiled versions of draw-line, one with AVX2 instructions.

Now we have a legit optimisation to the code. Not too much extra code, just some feature detection wiring and inlining. 

## Optimising feature detection & SSE4.2
An issue you may have spotted with the latest code is we call `is_x86_feature_detected` every time `draw_line` is called. Tbf the benchmark is showing this isn't a huge deal, but its still unnecessary. If we wanted to add more SIMD feature levels we'd be calling `is_x86_feature_detected` perhaps multiple times too.

Talking of SIMD levels, SSE4.2 is supported by almost everyone, so lets try that.
```sh
$ cargo bench --bench rasterize rasterize_outline_ttf_biohazard -- --baseline default

rasterize_outline_ttf_biohazard
        time:   [15.143 µs 15.148 µs 15.154 µs]
        change: [-17.860% -17.746% -17.633%] (p = 0.00 < 0.05)
```
SSE4.2 provides a great speedup, not quite as effective as AVX2 but definitely worth having for those without the newer instruction.

We can do feature detection, including SSE4.2, earlier saving the best function path to use and just calling that pointer in `draw_line`.

```rust
type DrawLineFn = unsafe fn(&mut Rasterizer, Point, Point);

impl Rasterizer {
    pub fn new(width: usize, height: usize) -> Self {
        // runtime detect optimal simd impls
        let draw_line_fn: DrawLineFn = if is_x86_feature_detected!("avx2") {
            draw_line_avx2
        } else if is_x86_feature_detected!("sse4.2") {
            draw_line_sse4_2
        } else {
            Self::draw_line_scalar
        };

        Self {
            width,
            height,
            a: vec![0.0; width * height + 4],
            draw_line_fn,
        }
    }

    pub fn draw_line(&mut self, p0: Point, p1: Point) {
        unsafe { (self.draw_line_fn)(self, p0, p1) }
    }

    #[inline(always)]
    fn draw_line_scalar(&mut self, p0: Point, p1: Point) {
        /* maths-loops */ 
    }
}

#[target_feature(enable = "avx2")]
unsafe fn draw_line_avx2(rast: &mut Rasterizer, p0: Point, p1: Point) {
    rast.draw_line_scalar(p0, p1)
}

#[target_feature(enable = "sse4.2")]
unsafe fn draw_line_sse4_2(rast: &mut Rasterizer, p0: Point, p1: Point) {
    rast.draw_line_scalar(p0, p1)
}
```
Now when calling `Rasterizer::new` we'll pick an AVX2 draw-line fn or a SSE4.2 fn or, if neither are supported, the default scaler code. Now we have 3 compiled versions of this function selected during runtime providing almost everyone with more optimal performance and breaking no-one.

> It would be cool to use [once_cell](https://github.com/matklad/once_cell) for this so the function was picked just once. But in my case I wanted to avoid dependencies for this crate. I'd love once_cell to be in std!
> 
> _Update 2023-01-12: This can also be done without additional dependencies using `std::sync::Once`. See [ab-glyph#71](https://github.com/alexheretic/ab-glyph/pull/71)._

## no_std & non-x86 compatibility
My CPU and my optimisations are targeting x86_64 arch. However, _ab_glyph_rasterizer_ is used on other CPUs and in no_std environments.

* `#[target_feature(enable = "avx2")]` doesn't compile outside x86/x86_64
* `is_x86_feature_detected!` is in std, not available for no_std.

I've made my code work for all x86 & x86_64 CPUs but broken compilation elsewhere :(

We can fix this with more conditional compilation.
```rust
impl Rasterizer {
    pub fn new(width: usize, height: usize) -> Self {
        // runtime detect optimal simd impls
        #[cfg(all(feature = "std", any(target_arch = "x86", target_arch = "x86_64")))]
        let draw_line_fn: DrawLineFn = if is_x86_feature_detected!("avx2") {
            draw_line_avx2
        } else if is_x86_feature_detected!("sse4.2") {
            draw_line_sse4_2
        } else {
            Self::draw_line_scalar
        };
        #[cfg(any(
            not(feature = "std"),
            not(any(target_arch = "x86", target_arch = "x86_64"))
        ))]
        let draw_line_fn: DrawLineFn = Self::draw_line_scalar;

        Self {
            width,
            height,
            a: vec![0.0; width * height + 4],
            draw_line_fn,
        }
    }
    // draw_line, draw_line_scalar unchanged
}

#[cfg(all(feature = "std", any(target_arch = "x86", target_arch = "x86_64")))]
#[target_feature(enable = "avx2")]
unsafe fn draw_line_avx2(rast: &mut Rasterizer, p0: Point, p1: Point) {
    rast.draw_line_scalar(p0, p1)
}

#[cfg(all(feature = "std", any(target_arch = "x86", target_arch = "x86_64")))]
#[target_feature(enable = "sse4.2")]
unsafe fn draw_line_sse4_2(rast: &mut Rasterizer, p0: Point, p1: Point) {
    rast.draw_line_scalar(p0, p1)
}
```

Finally we're up to date with where [_ab_glyph_rasterizer_ is now](https://github.com/alexheretic/ab-glyph/blob/main/rasterizer/src/raster.rs). We have AVX2 & SSE4.2 auto-vectorized versions of `draw_line` while continuing to support no_std & non-x86.

In future it'll be simple enough to add add paths for AVX512 and for other arch SIMD. These just require testing to prove they are worthwhile. My 5800x sadly does not support AVX512.

## An easier way: multiversion
Btw this concept of runtime selected pre-compiled optimised functions is called **multiversioning**. And yes, there is a crate to help reduce the noise a bit. [multiversion](https://github.com/calebzulawski/multiversion) can help compress our code.

So lets go back to the start and optimise this function for AVX2 & SSE4.2 similarly to how we just did manually.
```rust
impl Rasterizer {
    pub fn draw_line(&mut self, p0: Point, p1: Point) { 
        /* maths-loops */ 
    }
}
```
`cargo add multiversion`
```rust
impl Rasterizer {
    #[multiversion::multiversion]
    #[clone(target = "[x86|x86_64]+avx2")]
    #[clone(target = "[x86|x86_64]+sse4.2")]
    pub fn draw_line(&mut self, p0: Point, p1: Point) { 
        /* maths-loops */ 
    }
}
```
Instead of the manual impl we use 3 lines of proc-macro DSL.

```sh
$ cargo bench --bench rasterize rasterize_outline_ttf_biohazard -- --baseline default

rasterize_outline_ttf_biohazard
        time:   [14.479 µs 14.482 µs 14.485 µs]
        change: [-21.446% -21.335% -21.230%] (p = 0.00 < 0.05)
```
And it works (always worth checking!).

The expanded code seems to be equivalent to when we were doing simple `if is_x86_feature_detected!` inside `draw_line` fn. We don't have the same control over the feature detection to optimise it. Another issue here is that example is no longer no_std compatible, but it should be possible to do so by sprinkling the required conditional compilation flags.

[multiversion](https://github.com/calebzulawski/multiversion) seems a pretty nice way to introduce runtime selected auto-vectorization to your code with minimal noise.

> Note: For _ab_glyph_rasterizer_ I kept a manual implementation, primarily to keep _ab_glyph_rasterizer_ at zero dependencies.

## Hand-written SIMD
I find the auto-vectorized code to be fairly easy to maintain as there isn't any new "logic" bits to handle, just conditional compilation to wrangle. Writing the SIMD intrinsics by hand is much harder and more difficult to maintain.

On the other hand, reading [Nick Wilcox's Auto-Vectorization for Newer Instruction Sets in Rust](https://www.nickwilcox.com/blog/autovec2/) we can see it can also yield significantly better results. Try it if you dare!

## Conclusion
To summarize: If you want to SIMD optimise some code:
* Write a benchmark.
* Try vs `RUSTFLAGS='-C target-cpu=native'`.
* If that's promising, try targeting functions with `#[target_feature`
* If that works, try properly multiversioning either manually or with the crate.

Good luck optimising, lets make the most of our CPUs!

