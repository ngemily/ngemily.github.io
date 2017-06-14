---
layout: post
title:  "Partials and closures in C++"
date:   2017-06-13 20:02 -0400
categories: cpp ptree partial closure
---

# Problem Statement

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

# Attempts

## A procedural approach

This is one way I might do this procedurally.

``` cpp
void procedural(const ptree_t& root)
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

## Partial application

First, we take the body of the loop and make it into a function.  This is
accomplished by using _lambda functions_, which in C++ is
`[](<params>){<stmnts>}`.  It's just a function, but unlike conventional C++
functions, we can store in a variable.  It is called like a regular function.

``` cpp
// string -> string -> string -> ptree_t -> void
auto print_instance = [&](string inst, string ip, string prefix, const ptree_entry_t& tp){
    cout << "(" << inst << "," << ip << "," << prefix << tp.second.data() << ")" << std::endl;
};
```

The `string -> string -> string -> ptree_t -> void` notation comes from Haskell,
and shows the function signature.  It shows the argument and return types.

_Partial application_ means that we take a function and give it only some of the
arguments it expects. In functional programming, calling a function `f(x, y)` is
called _application_ -- one is _applying_ the function `f` to the values `x` and
`y`. If we instead only apply `f` to `x`, then that is called _partial
application_. In C++, this is done with the function `std::bind`.

``` cpp
using namespace std::placeholders; // Provides _1, _2, ...

// string -> string -> ptree_t -> void
auto print_inout = std::bind(print_instance, "foo", _1, _2, _3);
```

`print_instance` is the name of the function we're applying in the rest of the
arguments are the values are what we're applying `print_instance` to. The `_1,
_2, _3` are necessary in C++ and show that `print_inout` is a function that
expects three arguments.  Calling `print_inout(...)` is equivalent to
`print_instance("foo", ...)`.

> Aside: One way you can think of this is like the image of a vector in a lower
> dimension. The function `print_instance` is a higher dimensional object, which
> has been projected into a lower dimension where one dimension got pinned to
> the value `"foo"`. Then, `print_inout` is the image of `print_instance` in
> `inst = "foo"`.

We can use this to simplify our inner loop code.

``` cpp
void partial1(const ptree_t& root)
{
    // string -> string -> string -> ptree_t -> ()
    auto print_instance = [&](string inst, string ip, string prefix, const ptree_entry_t& tp){
        cout << "(" << inst << "," << ip << "," << prefix << tp.second.data() << ")" << std::endl;
    };

    for (auto& instance : root)
    {
        // string -> string -> ptree_t -> ()
        auto print_inouts = std::bind(print_instance, instance.first, _1, _2, _3);

        // inputs
        auto inputs = instance.second.get_child("INPUTS");
        for (auto& input : inputs)
        {
            // ptree_t -> ()
            auto print_input = std::bind(print_inouts, input.first, "IN_", _2);
            for (auto& port : input.second)
            {
                print_input(port);
            }
        }
        ...
```

Notice that our inner loop is comprised of a single statement, which is a
function call where the one and only argument is the loop variable.  This can be
replaced with a call to `std::for_each`.

``` cpp
std::for_each(input.second.begin(), input.second.end(), print_input);
```

In one line, one logical statement, I've said what _computation_ I intend to do,
and over what _range_ of items in a way where the computation itself is defined
separately.

Note that this can be done simply by writing a "normal" C++ function, without
the use of lambdas or partials, but the lambda is arguably more convenient.  If
we had not used partial application to reduce our `print_instance` function to a
function of one argument, we could instead write:

``` cpp
std::for_each(input.second.begin(), input.second.end(), [](auto x){
        print_instance(instance.first, input.first, "IN_", x);});
```

which accomplishes the flattening, but not the improved readability.


Altogether, we can consider this our partial application solution:

``` cpp
void partial(const ptree_t& root)
{
    // string -> string -> ptree_t -> ()
    auto print_instance = [&](string inst, string ip, string prefix, const ptree_entry_t& tp){
      cout << "(" << inst << "," << ip << "," << prefix << tp.second.data() << ")" << std::endl;
    };

    for (auto& instance : root)
    {
        // string -> ptree_t -> ()
        auto print_inouts = std::bind(print_instance, instance.first, _1, _2, _3);

        // inputs
        auto inputs = instance.second.get_child("INPUTS");
        for (auto& input : inputs)
        {
            // ptree_t -> ()
            auto print_inputs = std::bind(print_inouts, input.first, "IN_", _1);
            std::for_each(input.second.begin(), input.second.end(), print_inputs);
        }

        // outputs
        auto outputs = instance.second.get_child("OUTPUTS");
        for (auto& output : outputs)
        {
            // ptree_t -> ()
            auto print_outputs = std::bind(print_inouts, output.first, "OUT_", _1);
            std::for_each(output.second.begin(), output.second.end(), print_outputs);
        }

    }

}
```

There's one less level of nesting, but there's still a lot of duplication going
on.

## More partials

What we're left with is similar to the code we started with, but with one level
less of nesting.  Let's try writing another lambda for the currently innermost
loop (previously second most inner loop).

``` cpp
void partial1(const ptree_t& root)
{
    // string -> string -> ptree_t -> ()
    auto print_instance = [&](string inst, string ip, string prefix, const ptree_entry_t& tp){
        cout << "(" << inst << "," << ip << "," << prefix << tp.second.data() << ")" << std::endl;
    };

    for (auto& instance : root)
    {
        // string -> ptree_t -> ()
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
loop body in an inline lambda.  However, this is arguably much worse than the
code we started with.  We've simply traded nested for-loops for nested
`for_each` calls.

Let's refine our attempt to use more partials to modularize our inner loop
code.

## Closure

``` cpp
void closure(const ptree_t& root)
{
    // string -> string -> ptree_t -> ()
    auto print_instance = [&](string inst, string ip, string prefix, const ptree_entry_t& tp){
        cout << "(" << inst << "," << ip << "," << prefix << tp.second.data() << ")" << std::endl;
    };

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
        std::for_each(inputs.begin(), inputs.end(), print_inputs); // O(n^2)

        // outputs
        auto outputs = instance.second.get_child("OUTPUTS");
        auto print_outputs = std::bind(print_inouts, "OUT_", _1);
        std::for_each(outputs.begin(), outputs.end(), print_outputs);
     }

}
```

Although we have "flattened" our source code, the time complexity is the same.

Finally, the code for dealing with "inputs" and "outputs" are similar enough to
warrant some refactoring.

``` cpp
void closure1(const ptree_t& root)
{
    // string -> string -> ptree_t -> ()
    auto print_instance = [&](string inst, string ip, string prefix, const ptree_entry_t& tp){
        cout << "(" << inst << "," << ip << "," << prefix << tp.second.data() << ")" << std::endl;
    };

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

> Aside:
> ``` cpp
> auto print_nodes = std::bind(print_inouts, key.second, _1);
> ```
> would possibly be rewritten as:
> ``` cpp
> auto print_nodes = print_inouts(key.second)
> ```
> if curried functions were supported in C++.


## A note on performance

It turns out the procedural code is the fastest.

|--
| Function | Time (s) |
|--
| `procedural` |  0.863397 |
| `partial`    |  0.980094 |
| `partial1`   |  1.01364  |
| `closure`    |  0.934624 |
| `closure1`   |  0.961    |
|--

I currently do not know much about the implementation or limitations on compiler
optimizations when using lambdas and partial application in C++.

# Conclusion

1. Functional programming concepts can be mixed in with "regular" code.
1. Lambdas help use separate container traversals from the actual work.
2. `std::bind` gets you to a whole new level of complexity with respect to
   compiler optimizations
3. C++ functional programming is very verbose compared to other languages
