---
layout: post
title:  "Overload Sets with O(n) Compile Time Complexity"
# subtitle: "What you can do with it!"
date:   2016-12-18 01:00:00
categories: [Cpp]
---

Recently, I've been working on a project that involves a lot of `std::variant<>` kind of things. A `variant` is most useful for visiting! In order to do so, you need a function object that can be called with a value of any of the types in the `variant` parameter list.

## The Conventional Way

So, for instance, if you have a `std::variant<int, std::string>`, you can visit it using an instance of the `my_visitor` class:

``` cpp
class my_visitor {
public:
    void operator()(std::string s);
    
    template<class T>
    void operator()(T t); // This overload will receive the int variant
};

std::variant<int, std::string> my_variant;
std::visit(my_visitor{}, my_variant);
```

## The Lambda Way

You can of course also use a C++11 lambda function to visit the variant, as long as the lambda accepts all the variant types:

``` cpp
std::visit([](const auto & in){std::cout << in;}, my_variant);
```

The trouble is that it's not always possible, or practical, to cram all the possible types into a single lambda definition. Overload sets come to the rescue! The idea is to build a collection of function objects in such a way that the function call operator of the collection forwards the call to the call operator of one of its elements.

``` cpp
std::visit(
    some_impl::overload(
        [](std::string s) {std::cout << s;},
        [](int i) {std::cerr << "But that's a number!?!";}
    ),
    my_variant
);
```

That's very nice, we're building a local scope for overload resolution to take place.

## How is it Implemented?

The first time I felt the need for the `overload` utility, I tried to come up with how it could be implemented. The following definition would do the job fine, if only it were valid C++!!!

``` cpp

template<class ... Funcs>
class overload_t: public Funcs... {
    using Funcs::operator()...;
};

// And a utility function to construct overload sets
template <class ... Funcs>
overload_t<Funcs...> overload(Funcs ... funcs){
    return {funcs...};
};
```

The problem with that code is the `using Funcs::operator()...;` bit, which isn't valid C++. We need something along those lines, because otherwise the call operator of `overload_t<...>` would be ambigious (since more than one of its bases define it).

I've found three implementations of this `overload` concept:

 * [std::overload proposal](https://github.com/viboes/std-make/blob/master/include/experimental/fundamental/v3/functional/overload.hpp)
 * [boost::hana](https://github.com/boostorg/hana/blob/master/include/boost/hana/functional/overload.hpp)
 * [vrm_core](https://github.com/SuperV1234/vrm_core/blob/437a0afb35385250cd75c22babaeeecbfa4dcacc/include/vrm/core/overload/make_overload.hpp)

And they all rely on the same mechanism to achieve the intention of `using Funcs::operator()...;`

``` cpp
template<class Func, class ... Funcs>
class overload_t: Func, overload_t<Funcs...> {
    using Func::operator();
    using overload_t<Funcs...>::operator();
};
```

So, you see, instead of bringing the call operators into scope all at once, they build a ladder-like inheritance chain that brings the individual call operators into scope one by one. That's a nice solution to the problem with the simple initial attempt.

## The Complexity Though...

For normal use cases, the above implementation should be fine; so I've no problem with it. However, I sometimes need to visit variants with many alternative types, and the above implementation bugs me in such cases. You see, if you instantiate `overload_t<>` above with 20 type arguments, it's going to recursively inherit from an instantiation of itself with 19 type arguments, and so on!

Who knows what the compiler is doing with our metaprograms, so it's very hard to reason about the compile-time performance of our "metacode", but one thing is for certain; the above implementation has an O(n<sup>2</sup>) memory and processing burden on the compiler. For instance, for each instantiation of the `overload_t` type, it has to check whether it's seen that combination of type parameters before, so it has to look in some kind of map with the list of arguments, and that operation will at least be linear in the number of parameters. Then it has to repeat this operation for each of the recursive bases.

## A Possible Linear-Complexity Alternative

This got me thinking about whether it'd be possible to implement something similar with better compile-time performance. I have to admit that I wasn't able to come up with an alternative that has the exact same behavior as the implementations above, but I actually ended up implementing a variation with different overload resolution behavior:

``` cpp
template<class Func>
struct overload_base {
    Func f;

    template <class ... Args>
    friend auto call(overload_base * b, Args && ... args) 
      -> decltype(f(std::forward<Args>(args)...)) {
        return b->f(std::forward<Args>(args)...);
	}
};

template<class ... Funcs>
struct overload_impl: overload_base<Funcs>... { // No recursive bases
    overload_impl(Funcs...funcs): overload_base<Funcs>{funcs}...{}

    template <class ... Args>
    auto operator()(Args && ... args) {
        return call(this, std::forward<Args>(args)...);
    }
};

// A utility for constructing overload_impl
template<class ... Funcs>
overload_impl<Funcs...> overload(Funcs...funcs){
    return {funcs...};
}
```

The trick here is that `overload_impl<...>` delegates the job of `operator()` to a free function called `call`. This `call` function is then defined locally by each of the `overload_base<F>`s as a `friend` function. See, we've worked around the need for a `using` declaration by relying on `friend` functions, since they're discovered via argument dependent lookup (ADL).

Now we can visit our variant with:

``` cpp
std::visit(
    overload(
        [](std::string s) {std::cout << s;},
        [](int i) {std::cerr << "But that's a number!?!";}
    ),
    my_variant
);
```

And it'll work.

## The Downside

It's not all roses though; this implementation will not behave exactly like the others. We're relying on SFINAE in order to dispatch a given call to the right `overload_base`. I.e, in the definition of `overload_base<>`: 

``` cpp 
auto call(...) -> decltype(f(args...))
```

This function declaration will only be considered for overload resolution if `f(args...)` is valid, and that's why it works in the example above. But it won't work in the example below:

``` cpp
std::visit(
    overload(
        [](std::string s) {std::cout << s;},
        [](auto i) {std::cerr << "But that's not a string!?!";}
    ),
    my_variant
);
```

The trouble is that if you try to invoke the `operator()` of this `overload` with `std::string`, the `call` friend function from both `overload_base`s will be valid (neither will be discarded via SFINAE). This is not a problem with the other implementations, since they transparently rely on the regular overload resolution rules of C++, but with this implementation the signature of the `call` function hides all the details needed for such resolution.

## Conclusion

OK, I've promised overload sets with linear compile-time complexity in the title, and it turned out that I was lying about everything that implied. First, it's not an "overload set" implementation like others. It blocks the overload resolution rules of C++, so you have to make sure that only one of the function objects will be viable in any call site. Second, we don't really know whether any compiler will actually instantiate this overload set with linear complexity, but they at least have a chance!

I don't at all claim this `overload` implementation is superior to the others, but I still hope it'll be useful to some people.
