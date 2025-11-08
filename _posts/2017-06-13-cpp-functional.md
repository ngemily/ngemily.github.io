---
layout: post
title:  "Partials and closures in C++"
date:   2017-06-13 20:02 -0400
categories: cpp ptree partial closure
---

The goal of code shown in the post in to read data from a json file, of the
following format:

```json
{
    "foo" : {
        "INPUTS" : {
            "a" : ["foo_a0", "foo_a1"]
        },
        "OUTPUTS" : {
            "x" : ["foo_x"]
        }
    },
    "bar" : {
        "INPUTS" : {
            "b" : ["bar_b0", "bar_b1"]
        },
        "OUTPUTS" : {
            "x" : ["bar_x"]
        }
    }
}
```

and produce this output:
``` text
(foo,a,IN_foo_a0)
(foo,a,IN_foo_a1)
(foo,x,OUT_foo_x)
(bar,b,IN_bar_b0)
(bar,b,IN_bar_b1)
(bar,x,OUT_bar_x)
```

using C++.  The capital "INPUTS" and "OUTPUTS" are a fixed part of the schema
and everything else is variable.

## 1 | A procedural approach

This is one way I might do this procedurally, using boost property tree.

``` cpp
void procedural(const ptree& root)
{
    for (auto& instance : root)
    {
        // inputs
        auto inputs = instance.second.get_child("INPUTS");
        for (auto& input : inputs)
        {
            for (auto& port : input.second)
            {
                cout << "(" << instance.first << "," << input.first << "," << "IN_" << port.second.data() << ")" << std::endl;
            }
        }

        // outputs
        auto outputs = instance.second.get_child("OUTPUTS");
        for (auto& output : outputs)
        {
            for (auto& ports : output.second)
            {
              cout << "(" << instance.first << "," << output.first << "," << "OUT_" << ports.second.data() << ")" << std::endl;
            }
        }

    }

}
```

There are two things about this that bother me:

- deep nesting
- duplicated code

One of the problems with writing for loops is that the work (body of the
loop), is mixed with the traversal (`for (...)` part).  This makes the work code
non-reusable.  Let's try to separate the working logic from the container
traversal.

First let's improve this by refactoring the inner loop into a function

``` cpp
void print_instance(string inst, string ip, string prefix, const ptree_entry_t& tp)
{
    cout << "(" << inst << "," << ip << "," << prefix << tp.second.data() << ")" << std::endl;
}
```

And call it

``` cpp
void procedural1(const ptree& root)
{
    for (auto& instance : root)
    {
        // inputs
        auto inputs = instance.second.get_child("inputs");
        for (auto& input : inputs)
        {
            for (auto& port : input.second)
            {
                print_instance(instance.first, input.first, "IN_", port);
                ...
```


We can use `std::for_each` to reduce one of the for-loops.

``` cpp
void procedural2(const ptree& root)
{
    for (auto& instance : root)
    {
        // inputs
        auto inputs = instance.second.get_child("inputs");
        for (auto& input : inputs)
        {
            std::for_each(input.second.begin(), input.second.end(),
                          [&](auto x){print_instance(instance.first, input.first, "IN_", x);});
        }
        ...
```

but this doesn't look significantly more readable.

> C++ lambda syntax:
> ``` cpp
> [<capture style>](<params>){<stmnts>}
> ```
> In functions like `std::for_each`, the function you pass it will be called
> with each of the elements in the range.  Therefore in this example, you should
> pass it a function that takes one argument, of type `ptree`
> ``` cpp
> std::for_each(input.second.begin(), input.second.end(), 
>     [&](auto x){print_instance(instance.first, input.first, "IN_", x);});
> ```
> Note that `(auto x)` is C++14.

## 2 | Partial application

One reason that last solution look ugly is because we need a lambda, instead of
just passing a named function like this:

``` cpp
std::for_each(input.second.begin(), input.second.end(), print_inputs);
```

However, in order to do that, we need a function `print_inputs` that has the
function signature that we need.

We can use partial application to pare down our `print_instance` function into a
`print_inputs` function.  _Partial application_ means that we take a function
and give it only some of the arguments it expects. In functional programming,
calling a function `f(x, y)` is called _application_ -- one is _applying_ the
function `f` to the values `x` and `y`. If we instead only apply `f` to `x`,
then that is called _partial application_. In C++, this is done with the
function `std::bind`.

``` cpp
using namespace std::placeholders; // Provides _1, _2, ...

// string -> string -> ptree -> void
auto print_inout = std::bind(print_instance, "foo", _1, _2, _3);
```

`print_instance` is the name of the function we're applying in the rest of the
arguments are the values are what we're applying `print_instance` to. The `_1,
_2, _3` are necessary in C++ and show that `print_inout` is a function that
expects three arguments.  Calling `print_inout(...)` is equivalent to
`print_instance("foo", ...)`.

The `string -> string -> string -> ptree -> void` notation comes from Haskell,
and shows the function signature.  It shows the argument and return types.

> Aside: One way you can think of this is like the image of a vector in a lower
> dimension. The function `print_instance` is a higher dimensional object, which
> has been projected into a lower dimension where one dimension got pinned to
> the value `"foo"`. Then, `print_inout` is the image of `print_instance` in
> `inst = "foo"`.

``` cpp
void partial(const ptree& root)
{
    for (auto& instance : root)
    {
        // string -> ptree -> ()
        auto print_inouts = std::bind(print_instance, instance.first, _1, _2, _3);

        // inputs
        auto inputs = instance.second.get_child("INPUTS");
        for (auto& input : inputs)
        {
            // ptree -> ()
            auto print_inputs = std::bind(print_inouts, input.first, "IN_", _1);
            std::for_each(input.second.begin(), input.second.end(), print_inputs);
        }

        // outputs
        auto outputs = instance.second.get_child("OUTPUTS");
        for (auto& output : outputs)
        {
            // ptree -> ()
            auto print_outputs = std::bind(print_inouts, output.first, "OUT_", _1);
            std::for_each(output.second.begin(), output.second.end(), print_outputs);
        }

    }

}
```

In one line, one logical statement, 
`std::for_each(input.second.begin(), input.second.end(), print_inputs);`
I've said what _computation_ I intend to do, and over what _range_ of items in a
way where the computation itself is defined separately.

There's one less level of nesting, but there's still a lot of duplication going
on.

## 3 | More partials

What we're left with is similar to the code we started with, but with one level
less of nesting.  Let's try writing another lambda for the currently innermost
loop (previously second most inner loop).

``` cpp
void partial1(const ptree& root)
{
    for (auto& instance : root)
    {
        // string -> ptree -> ()
        auto print_inouts = std::bind(print_instance, instance.first, _1, _2, _3);

        // inputs
        auto inputs = instance.second.get_child("INPUTS");
        std::for_each(inputs.begin(), inputs.end(), [&](auto x){
                auto print_inputs = std::bind(print_inouts, x.first, "IN_", _1);
                std::for_each(x.second.begin(), x.second.end(), print_inputs);
                });

        // outputs
        auto outputs = instance.second.get_child("OUTPUTS");
        std::for_each(outputs.begin(), outputs.end(), [&](auto x){
                auto print_outputs = std::bind(print_inouts, x.first, "OUT_", _1);
                std::for_each(x.second.begin(), x.second.end(), print_outputs);
                });

    }
}
```

What I've done is essentially replaced loops with `for_each` calls, and put the
loop body in an inline lambda.  This is arguably much worse than the code we
started with.  We've simply traded nested for-loops for nested `for_each`
calls.  This is **not** a good way to use `for_each`.

Let's refine our attempt to use more partials to modularize our inner loop
code.

## 4 | Closure

The lambdas passed to `for_each` in the last example are basically the same,
except for the arguments `"IN_"` and `"OUT_"`.  We can extract that lambda into
a variable.

``` cpp
auto print_inouts = std::bind(print_instance, instance.first, _1, _2, _3);
auto print_inouts2 = [&](string prefix, ptree_entry& x) {
        auto print_outputs = std::bind(print_inouts, x.first, prefix, _1);
        std::for_each(x.second.begin(), x.second.end(), print_outputs);
        });
```

We no longer need `print_inouts` to exist separately, so the two lines can be
combined.

``` cpp
auto print_inouts = [&](string prefix, ptree_entry& x) {
        auto print_outputs = std::bind(print_instance, instance.first, x.first, prefix, _1);
        std::for_each(x.second.begin(), x.second.end(), print_outputs);
        });
```

Putting that together,

``` cpp
void closure(const ptree& root)
{
    for (auto& instance : root)
    {
        // ptree_entry_t -> ()
        // Performs a closure on `instance.first`
        auto print_inouts = [&](string prefix, const ptree_entry_t& x) {
            auto print_port = std::bind(print_instance, instance.first, x.first, prefix, _1);
            std::for_each(x.second.begin(), x.second.end(), print_port);
        };

        // inputs
        auto inputs = instance.second.get_child("INPUTS");
        auto print_inputs = std::bind(print_inouts, "IN_", _1);
        std::for_each(inputs.begin(), inputs.end(), print_inputs);

        // outputs
        auto outputs = instance.second.get_child("OUTPUTS");
        auto print_outputs = std::bind(print_inouts, "OUT_", _1);
        std::for_each(outputs.begin(), outputs.end(), print_outputs);
     }

}
```

It's important to remember that although we have "flattened" our source code,
the time complexity is the same.  This is true whether you're using modern C++
features or not.

Finally, the code for dealing with "inputs" and "outputs" are similar enough to
warrant some refactoring.

``` cpp
void closure1(const ptree& root)
{
    for (auto& instance : root)
    {
        // ptree_entry_t -> ()
        // Performs a closure on `instance.first`
        auto print_inouts = [&](string prefix, const ptree_entry_t& x) {
            auto print_port = std::bind(print_instance, instance.first, x.first, prefix, _1);
            std::for_each(x.second.begin(), x.second.end(), print_port);
        };

        vector<pair<string, string>> keys = {
            {"INPUTS", "IN_"},
            {"OUTPUTS", "OUT_"}
        };

        for (auto key : keys)
        {
            auto node = instance.second.get_child(key.first);
            auto print_nodes = std::bind(print_inouts, key.second, _1);
            std::for_each(node.begin(), node.end(), print_nodes);
        }

    }
}
```

In my opinion, had we done this with the procedural code and incurred a fourth
level of nesting, that would not be more readable.

|--
| Function | Time (ms) |
|--
| `closure`         | 26 |
| `partial`         | 26 |
| `procedural`      | 27 |
| `partial1`        | 27 |
| `procedural2`     | 28 |
| `procedural1`     | 29 |
| `closure1`        | 30 |
|--

These results are based on compiling with `-O3`, using `std::chrono` to profile
run time, averaged over 10 runs.  The result is that the performance is
practically the same across the board.  In fact, the order of fastest to slowest
is inconsistent across runs.

(Without `-O3`, procedural code is the fastest, by far.)

## Conclusion

Arguably, the procedural code was fine, and none of the fancy C++ footwork is
significantly more readable.  But what if the loop body was longer?  What if
there were more levels of nesting?  This was an exercise in extracting the work
code out of the container traversal, thinking differently, and exploring some
of the tools and techniques available.

1. Functional programming concepts can be mixed in with "regular" code.
1. Lambdas help use separate container traversals from the actual work.
1. C++ functional programming is very verbose compared to other languages.  Use
   of `_1, _2, ...` is not necessary is most other languages.  Use of
   `std::bind` is not necessary in languages that support currying.
1. Compilers are pretty good!  Code for clarity, optimizing only when needed.
