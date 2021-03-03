by\_row(): An alternative to map() or rowwise()
================
Matthew Kay
3/3/2021

This is a short proposal for a new alternative to `rowwise()` or `map()`
for rowwise operations on data frames inside mutate.

``` r
library(tidyverse)
library(rlang)
```

It’s useful to be able to perform rowwise operations on data frames. As
a trivial example (obviously these shouldn’t be list columns in
practice, but ignore that for now), this fails:

``` r
tibble(a = list(10, 11), b = list(3, 4)) %>%
  mutate(c = a + 2 * b)
```

    ## Error: Problem with `mutate()` input `c`.
    ## x non-numeric argument to binary operator
    ## i Input `c` is `a + 2 * b`.

One way to do rowwise operations is to use `lapply()` or `map()` inside
`mutate()`, but this requires you to pass through all the variables you
want to use in an awkward way:

``` r
tibble(a = list(10, 11), b = list(3, 4)) %>%
  mutate(c = map2_dbl(a, b, ~ .x + 2 * .y))
```

    ## # A tibble: 2 x 3
    ##   a         b             c
    ##   <list>    <list>    <dbl>
    ## 1 <dbl [1]> <dbl [1]>    16
    ## 2 <dbl [1]> <dbl [1]>    19

With `rowwise()` we can avoid managing all these variables:

``` r
tibble(a = list(10, 11), b = list(3, 4)) %>%
  rowwise() %>%
  mutate(c = a + 2 * b)
```

    ## # A tibble: 2 x 3
    ## # Rowwise: 
    ##   a         b             c
    ##   <list>    <list>    <dbl>
    ## 1 <dbl [1]> <dbl [1]>    16
    ## 2 <dbl [1]> <dbl [1]>    19

But `rowwise()` has significant drawbacks. It changes the meaning of
`mutate()` in a way that I have found confuses students—and, frankly,
experts (if I can be called an expert). If you don’t know the data frame
is in a `rowwise()` state, you can write `mutate()` calls that will
raise errors that don’t make sense. Or if you insert a `rowwise()` call
into a pipe for a `mutate()` and forget to ungroup the data frame
afterwards, it can break existing `mutate()` calls downstream.

The root of the problem, I think, is that `rowwise()` forces people to
keep track of state in a way that humans aren’t great at: we have to
remember if a data frame is in “rowwise” or “not rowwise” state for
every `mutate()` call, and the call that puts you in `rowwise()` mode
can be arbitrarily far from the corresponding mutate call. In
human-computer interaction we call errors due to thinking a system is in
a different state a [mode
error](https://en.wikipedia.org/wiki/Mode_(user_interface)#Mode_errors);
it’s the reason why people hate the CapsLock key—we’re not good at
keeping track of modes!

I think it would be better to explicitly “mark out” expressions intended
to be executed in a row-wise manner (similar to how `map()` works), but
not have to deal with managing variables (similar to how `rowwise()`
works). The best of both worlds!

My modest proposal is a function like this (which no doubt needs work to
handle some NSE corner cases I haven’t thought of):

``` r
by_row = function(expr) {
  .expr = enquo(expr)
  .data = cur_data()
  .env = parent.frame()

  simplify(lapply(seq_len(nrow(.data)), function(i) {
    eval_tidy(.expr, data = lapply(.data[i,], `[[`, 1), env = .env)
  }))
}
```

Which can be used like this:

``` r
tibble(a = list(10, 11), b = list(3, 4)) %>%
  mutate(c = by_row(a + 2 * b))
```

    ## # A tibble: 2 x 3
    ##   a         b             c
    ##   <list>    <list>    <dbl>
    ## 1 <dbl [1]> <dbl [1]>    16
    ## 2 <dbl [1]> <dbl [1]>    19

The other bonus here is that this can be intermixed with non-rowwise
calls in the same `mutate()` call. This is important, because *rowwise
operations are slow and you should try to avoid them at all costs*!
Putting the entire dataframe into a rowwise mode to perform these
operations encourages people to do other operations rowwise that they
don’t need to do.
