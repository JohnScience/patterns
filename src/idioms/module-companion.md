# Module-companion for a \[standalone\] function

## Description

In Rust, functions belong to the value [namespace], while modules belong to the type namespace. This results in a possibility of having a function and a module with the same name in the same scope [^1].

This idiom - *in the sense of a group of code fragments sharing an equivalent semantic role for specific language rather than a collection of code fragments that are considered conventional and supported by the community of that language* - is similar in spirit to

* ["companion object"] idiom in Scala and Kotlin,
* `TraitName` and `derive(TraitName)` idiom in Rust. For example, `serde::Serialize`&nbsp;(&nbsp;[trait](https://docs.rs/serde/latest/serde/trait.Serialize.html)&nbsp;|&nbsp;[derive macro](https://docs.rs/serde/latest/serde/derive.Serialize.html)&nbsp;).

Having a module with the same name as a function can be useful for containing items (e.g. helper functions, constants, and types) that are related to that function. [^2]

Notably, it can contain

* a struct for the "parameter object" pattern,
* an error enum for the *accompanied function* [^3].

## Degenerate example

```
pub mod my_fn {}

pub fn my_fn() {
    unimplemented!()
}
```

## Less-contrived example

**a.rs** (definition site):

```
pub mod my_fn {
    pub enum Error {
        AlreadyExists,
        ImATeapot,
        // ...
    }

    // Can implement Default for the "parameter object"
    pub struct Args {
        pub first_name: String,
        pub last_name: String,
        pub is_awesome: bool,
        pub is_lovely: bool,
        // ...
    }
}

pub fn my_fn(arg: my_fn::Args) -> Result<(), my_fn::Error> {
    // Star import from the module-companion is nearly always harmless
    use my_fn::*;

    // Destructuring the "parameter object"
    let Args {
        first_name,
        last_name,
        is_awesome,
        is_lovely,
        // ...
    } = arg;

    // ...

    Ok(())
}
```

**b.rs** (call site):

```
// imports both the accompanied function and the module-companion `my_fn`
use crate::a::my_fn;

fn another_fn() -> anyhow::Result<()> {
    let args = my_fn::Args {
        first_name: "Amandine".to_string(),
        // TODO: change the last name
        last_name: "Cerruti".to_string(),
        is_awesome: true,
        is_lovely: true,
        // ...
        // ..my_fn::Args::default()
    };

    my_fn(args)?;

    Ok(())
}
```

## Advantages

### Grouping related items

The *module-companion* can contain items that are related to the *accompanied function*, such as constants, helper functions, and types. This can help in organizing the codebase and making it more readable.

### Encapsulation

The *module-companion* can be used to encapsulate the implementation details of the *accompanied function*. This can help in reducing the cognitive load on the developers who are reading the code.

### Clean call sites

The *module-companion* can be used to define a struct for the "parameter object" pattern, which can help in reducing the number of arguments passed to the *accompanied function* with the help of [default idiom]. This can make the call sites cleaner and more readable due to the "syntactic" parallelism (`my_fn::Args { ... }` and `my_fn()`).

### Availability in the function signature

There is also an idiom where helper items for the function are defined in its body. Compared to that idiom, module-companion idiom has the advantage that the items from the *module-companion* are available in the function signature.

### Ability to name exclusively function-centric items

This pattern allows to give a sensible name to the items that are strictly function-centric and do not make sense on their own. This can help in making the code more readable and maintainable.

## Drawbacks

### Lacking language support

The language support for this idiom is limited, which is the root cause for the problems below.
While pragmatically related, from the Rust language's perspective, the *module-companion* and the *accompanied function* are unrelated items.

### Verbosity in the function signature

Using items from the *module-companion* in the signature of the *accompanied function* requires explicitly writing the shared name.

```
pub fn my_fn(arg: my_fn::Args) -> Result<(), my_fn::Error> {
    // function body
}
```

It can have a negative impact on the readability of the function signature when the function name is long. Unfortunately, it is often the case with complex functions, which are the ones that benefit the most from this idiom.

The "workaround" for this problem - which is strictly worse - is polluting the [namespace]s of the module where they are defined with the items from the *companion module*.

### Cross-namespace name collision

There are two distinct items with the same name in the same scope but different [namespace]s:

* the *module-companion* `my_fn` in the type namespace,
* the *accompanied function* `my_fn` in the value namespace.

This can be unexpected for developers and tools. However, derive macros and the traits that they implement also often share the name, so it is not too unexpected.

### Module-companion for an associated function for a type is poorly supported

While *module-companions* for standalone functions are well-supported, the same is not true for associated functions on types (structs and enums). This is because the *module-companion*

* for an inherent function on a type would have to be a submodule of the type and
* for a trait-associated function for a type would have to be a submodule of the trait,

neither of which is allowed in Rust.

### Rustdoc documentation could be improved

The documentation for the *module-companion* is not directly associated with the *accompanied function* in the generated documentation. This can make it harder for developers to understand the relationship between the two. However, fixing rustdoc to support "Same-name items" does not seem to be a difficult task.

### Arcane definition site

Considerable amount of people who were asked about this idiom mentioned that the definition site of the *module-companion* looks "odd" or "arcane".

The oddness of the definition site is counterbalanced by the readability of the call site.

### Limited good use cases even when "possible"

* In many - but not all - cases, you can...

    1. pick a reasonable name for the arguments that would not be function-centric,
    2. "invert" the association relationship and thus use a struct with associated items.

* If the "inversion" of the assocation relationship is impossible but the associated items can be given a reasonable name, you can consider use a module-per-function approach.

One can argue that this pattern is undesirable because there are often better alternatives.

However, it is not always the case, and the pattern can be useful for example when

* The associated items are strictly function-centric and do not make sense on their own,
* The function is complex and the benefits from the pattern outweigh the drawbacks.

*Note: little awareness about the idiom is not considered a drawback in this entry but may considered such by a considerable amount of developers. At the time of writing the article, this idiom is far from being widely known.*

## Footnotes

[^1]: Within this article, the term "scope" - unless stated otherwise - is used loosely to refer to the collection of items (e.g. constants, structs, and functions) that belong to any of the Rust's [namespace]s and that are "visible" as a result of being defined or imported.
[^2]: Note that [procedural macros] are implemented as functions, so this idiom can be used to group the implementation details of individual procedural macros.
[^3]: For an error enum in a *companion-module*, you can consider using the [`thiserror`] crate to derive [`Error`] and [`Display`] traits. Also see the [comment about "library-like" and "application-like" errors][errors-comment] on reddit by `@dtolnay`.

["companion object"]: https://docs.scala-lang.org/overviews/scala-book/companion-objects.html
[namespace]: https://doc.rust-lang.org/reference/names/namespaces.html
[procedural macros]: https://doc.rust-lang.org/reference/procedural-macros.html
[`Error`]: https://doc.rust-lang.org/std/error/trait.Error.html
[`Display`]: https://doc.rust-lang.org/std/fmt/trait.Display.html
[`thiserror`]: https://crates.io/crates/thiserror
[errors-comment]: https://www.reddit.com/r/rust/comments/dfs1zk/comment/f35iopj/
[default idiom]: https://rust-unofficial.github.io/patterns/idioms/default.html
