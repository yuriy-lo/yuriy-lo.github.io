---
layout: post
title: "Command line arguments parsing with C++26 reflection"
date: 2025-11-30 00:00:00 +0000
---

If you ask Copilot to generate a basic REST server in C++, it will produce times more code than a similar Python example. The difference is boilerplate that registers endpoints and serializes/deserializes arguments. Good news is that C++ committee voted reflection into draft C++26. So, everything is going to change. The feature is so exciting that I couldn't resist playing with it and implementing something that I miss in C++ world: automatic command line arguments parsing and dispatching.

Imagine, you can write CLI program like this:

```c++
#include "argparse.h"

int sum(int x, int y);
int sqr(int v);

int main(int argc, char *argv[]) {
    return argparse::run<sum, sqr>(argc, argv);
}
```

And then run it from command line:

```bash
my_cli sum --x=1 --y=2
my_cli sqr --v=3
```

Looks fantastic, right? Let's see how it works under the hood.

You can find the full code example in my [GitHub repository](https://github.com/yuriy-lo/articles/tree/main/args-reflect). It works on Linux with Clang fork supporting P2996 reflection proposal (see readme on GitHub for instructions how to build).


```argparse::run``` function is simple: it calls ```argparse::parse_cmd``` to convert ```argc``` and ```argv``` into a map of key-value pairs and then passes result to another ```argparse::run```. ```argparse::parse_cmd``` is an example of what coding agents excel at:

```c++
// Parse command line of the form:
//   program command --param1 val1 --param2 val2 ...
//   program command --param1=val1 --param2=val2 ...
// Values are required
std::pair<std::string_view, std::unordered_map<std::string_view, std::string_view>> parse_cmd(int argc, char *argv[]);

template <auto... Functions>
int run(int argc, char *argv[])
{
    const auto arguments = parse_cmd(argc, argv);
    return run<Functions...>(arguments.first, arguments.second);
}
```

Magic starts here. Another ```argparse::run``` selects a function by name and passes control to it:

```c++
template <auto... Functions>
int run(std::string_view command, const std::unordered_map<std::string_view, std::string_view> &params)
{
    // Create an array of reflected function infos by unpacking functions from template parameters pack 
    // and calling std::meta::reflect_function on each of them
    constexpr auto function_infos = std::array{std::meta::reflect_function(*Functions)...};
    // Iterate through all reflected functions and find one that matches the command name
    template for (constexpr auto function_info : function_infos)
    {
        if (std::meta::identifier_of(function_info) == command)
        {
            // Try to get parameter values from the params map and call the function if successful
            if (auto values = get_parameters_values<function_info>(params))
            {
                return std::apply([:function_info:], *values);
            }
        }
    }
    return 1;
}
```

That's small and clear! If I wanted to run functions without parameters I could stop here. Just write ```[:function_info:]();``` instead of converting parameters values and calling ```std::apply```. 

Code snippet above already uses a lot of new C++26 reflection features: ```std::meta::reflect_function``` returns function info, ```std::meta::identifier_of``` returns function name, ```template for``` is an expansion statement which allows compile-time for-loops, ```[: :]``` is a slicer that evaluates callable function from its reflection. ```std::array``` is here because my compiler can't handle ```std::vector``` in compile time. Also there's a lot from C++11 and follow-up standards which make this code possible: variadic templates and parameter packs, list-initialization, range-for, ```std::apply``` to unpack tuple into function parameters, ```constexpr```, ```auto```, ```std::array```, ```std::unordered_map```, ```std::string_view```.

```argparse::get_parameters_values``` function is little big bigger. It creates ```std::tuple``` containing values of function parameters by converting them from ```std::string_view``` and returns empty ```std::optional``` if any parameter is missing or can't be converted:

```c++
template <auto Function>
auto get_parameters_values(const std::unordered_map<std::string_view, std::string_view> &parameters)
{
    bool all_params_parsed = true;
    // Produce a tuple by expanding function parameters reflections and converting each parameter from string
    auto values = [:__impl::expand(std::meta::parameters_of(Function)):].make_tuple([&]<auto param>
    {
        constexpr auto param_name = std::meta::identifier_of(param);
        auto it = parameters.find(std::string_view(param_name));
        if (it != parameters.end())
        {
            auto value = convert_string<typename[:std::meta::type_of(param):]>(it->second);
            if (value)
            {
                return *value;
            }
        }
        all_params_parsed = false;
        // Tuple of correct types is needed in compile time, even if parsing failed
        return typename[:std::meta::type_of(param):]{};
    });
    using Result = std::optional<decltype(values)>;
    return all_params_parsed ? Result{values} : Result{};
}
```

New reflection features: ```std::meta::parameters_of``` returns a list of function parameters, ```std::meta::type_of``` returns reflection of type and ```typename[: :]``` slicer allows to use that type. ```__impl::expand``` was inspired by example from C++ proposal paper:

```c++
template <auto... vals>
struct replicator_type
{
    // Creates tuple from values generated by template function object
    // for every funcion parameter reflection in vals
    template <typename F>
    constexpr auto make_tuple(F body) const
    {
        return std::make_tuple(body.template operator()<vals>()...);
    }
};

template <auto... vals>
replicator_type<vals...> replicator = {};

// Creates an instance of replicator_type from funcion parameter reflections
template <typename R>
consteval auto expand(R range)
{
    std::vector<std::meta::info> args;
    for (auto r : range)
    {
        args.push_back(std::meta::reflect_constant(r));
    }
    return std::meta::substitute(^^__impl::replicator, args);
}
```

This snippets introduce more reflections features: ```std::meta::substitute``` creates an instance of a template by substituting its template parameters with values from the ```args``` array, ```^^``` operator produces reflection from grammar construct, ```std::meta::info``` is the reflection datatype. Calling ```std::meta::reflect_constant``` on function parameter reflections and storing results in a vector shouldn't be necessary, but my compiler can't call ```std::meta::substitute``` with output of ```std::meta::parameters_of``` in compile time. Also don't forget about C++11 and later features: lambda functions with template parameters, initializer lists,  ```std::optional```, ```decltype```, ```std::make_tuple```.

Finally, ```argparse::convert_string``` function converts string values into required types. One more example of coding agents in action:

```c++
template <typename T>
std::optional<T> convert_string(std::string_view s);

template <>
std::optional<int> convert_string<int>(std::string_view s)
{
    int result;
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), result);
    if (ec == std::errc())
    {
        return result;
    }
    return std::nullopt;
}
```

Even this uses C++17 features: ```std::from_chars```, structured bindings, ```std::optional```, ```std::string_view```.

And that's it! The whole project is about 200 lines of code, half of which is converting ```argc``` and ```argv``` to easy-to-use data structures and converting string values to required types. Sure it misses error logging, help message generation, support for more argument types and value-less command line arguments, default function arguments are not supported by reflections at all, but argument parser library already does the job and demonstrates how powerful C++26 reflections can be. REST APIs generation, serialization/deserialization, automatic binding to scripting languages shouldn't be more difficult to implement.

## Summary

* C++26 reflections are already powerful enough to solve real-world tasks. Hope everything will stabilize and become widely available soon.
* It's time to master all the template and compile time evaluations black magic from C++11 and later standards, because they are essential to use reflections effectively.
* Support in compiler and standard library still has a room for improvement. Anyway, I used un-official fork of Clang.
* Most time-consuming task was to build custom Clang. I failed to build ```libcxx``` target on Windows, to build on WSL2 required increase of VM memory to 12GB. And build of Clang takes LONG.

## References

* [Reflection for C++26 (P2996R12)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r12.html)
* [Expansion Statements (P1306R5)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p1306r5.html)
* [Function parameter reflection (P3096R12)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p3096r12.pdf)
* [Clang/P2996 fork (Reflection for C++26) by Bloomberg](https://github.com/bloomberg/clang-p2996/tree/p2996)
* [Building and Running Clang](https://clang.llvm.org/get_started.html)
* [Code from the article on GitHub](https://github.com/yuriy-lo/articles/tree/main/args-reflect)
