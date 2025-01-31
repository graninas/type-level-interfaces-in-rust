# Type-level interfaces in Rust

### Table of Contents

* **Intro**
* **Type-level eDSLs**
* **Why type-level eDSLs?**
* **Type-level domain modeling**
* **Domain modeling with type-level interfaces**
* **Type-level interfacing mechanism**
* **Interpretation and the universal evaluation mechanism**
* **Conclusion**
* **Contact me**
* **Intro**

I invented type-level interfaces for Haskell, Rust, and Scala 3 while working on my third book, [*Pragmatic Type-Level Design*](https://leanpub.com/pragmatic-type-level-design) (Leanpub, 2024). My goal was to make type-level programming simple, approachable, and practical—and I succeeded in all three languages. Type-level programming is no longer dark magic. With my universal methodology, you can start crafting useful applications based on powerful, compile-time, statically verifiable, and truly extensible type-level eDSLs.

This post briefly outlines the approach and accompanies my recent talk at Functional Conf 2025: *Type-Level Interfaces in Haskell and Rust: Type-Level eDSLs* ([slides](https://docs.google.com/presentation/d/1Pi_p_a8Wvbl1MXNl_VsDMxcsSYq8FUPghZk8seVOvcs/edit?usp=sharing), [code](https://github.com/graninas/Pragmatic-Type-Level-Design/tree/96f16ab0c9d23e407653869801725b8cca130356/First-Edition/functional_conf_2025)). I omit many details here; consider reading the book. You can buy it on Leanpub with today’s special 50% discount using [this link](https://leanpub.com/pragmatic-type-level-design/c/rust_promo_50). The offer expires in three days. If you're unsure, you can first download a substantial 90-page free sample chapter and decide. Its model language is Haskell, but there are special Rosetta Stone chapters with Rust and Scala 3.

I’m also the author of *Functional Design and Architecture* (Manning Publications, 2024), a deep and advanced book that I can’t help but recommend. It’s a great resource for those who love functional programming and want to explore practical approaches structured into comprehensive, well-organized knowledge. My goal was ambitious: to consolidate FP ideas into a methodology that could finally serve as a true counterpart to Object-Oriented Design. I call it *Functional Declarative Design*—the missing link between functional programming and software engineering. You can buy FDaA at [Manning](https://www.manning.com/books/functional-design-and-architecture) or on [Amazon](https://www.amazon.com/Functional-Design-Architecture-Alexander-Granin/dp/1617299618).

Finally, I’m interested in writing an entire book on type-level design in Rust. A sustainable support of the PTLD book will give me a good argument for dedicating another two-three years of my life to this. Advanced books are super expensive to write.

You'll find the code for this post in the same repository.

### **Type-level eDSLs**

In Haskell, there’s *Servant*—a well-known type-level HTTP API. You define your routes entirely at the type level, using types and nothing but types. Below are the three methods of a Tic-Tac-Toe game API.

```haskell
type TicTacToeAPI =
      "start"                     -- POST "/start". Start a new game.
      :> Post '[JSON] Game        -- Returns game id (string)
 :<|> "move"                      -- POST "/move". Make move.
      :> Capture "id" String      -- game id parameter
      :> Capture "sign" String    -- cross or circle
      :> QueryParam "h" Int       -- h and v coordinates of the sign
      :> QueryParam "v" Int       -- (query parameters)
      :> Post '[JSON] String      -- Request the board.
 :<|> "board"                     -- GET "/board". Make move.
      :> Capture "id" String      -- game id parameter
      :> Get '[JSON] Board        -- Returns Board as JSON
```

Servant’s runtime handles this model and connects it to the handlers that perform the actual work. The type-level eDSL itself defines return types for methods, directs URL parsing, and verifies the formats supported by the server. While the library is praised for its type-level approach, it relies on some arbitrary, complex, and tricky Haskell concepts that are difficult to replicate in other languages.

Type-level interfaces solve this problem. My approach is more universal, unified, and structured compared to Servant’s internals. Type-level interfaces work seamlessly in Rust, Haskell, and Scala 3, making the Servant-like style possible across languages. Take a look at Rust’s equivalent:

```rust
type TicTacToeAPI = tl_list![IRoute,
     Route<POST, tl_str!("/start"),
           tl_list![IClause],
           tl_list![IFormat, JSON],
           DataType<Game>>,
     Route<POST, tl_str!("/move"),
           tl_list![IClause,
                    Capture<tl_str!("id"), StringType>,
                    Capture<tl_str!("sign"), StringType>,
                    QueryParam<tl_str!("h"), IntType>,
                    QueryParam<tl_str!("v"), IntType>],
           tl_list![IFormat, JSON],
           StringType>,
     Route<GET, tl_str!("/board"),
           tl_list![IClause,
                    Capture<tl_str!("id"), StringType>],
           tl_list![IFormat, JSON],
           DataType<Board>>];
```

Understandably, Rust is more verbose since it lacks various type-level features. Yet, just three features were enough to introduce type-level interfaces: traits as type classes, associated types, and empty parameterized structs. We’ll explore this in the next sections.

What else can you achieve? Take a look at the following example—it’s a type-level rule for a cellular automaton application. The code describes Conway’s *Game of Life*. Other rules, such as *Seeds* and *Replicator*, can be built using the same construction blocks.

```rust
type A = State<tl_str!("Alive"), 1>;
type D = State<tl_str!("Dead"), 0>;

type Neighbors3  = NeighborsCount<A, tl_i32![3]>;
type Neighbors23 = NeighborsCount<A, tl_i32![2, 3]>;

type GolTransitions = tl_list![IStateTransition,
   StateTransition<D, A, Neighbors3>,
   StateTransition<A, A, Neighbors23>];

type GoLStep = Step<D, GolTransitions>;

type GoLRule = Rule<
   tl_str!("Game of Life"),
   tl_str!("gol"),
   GoLStep>;
```

The Tic-Tac-Toe API, GoL rule, and other eDSLs will have a similar structure and won’t require deep learning, as type-level interfaces serve as a universal design pattern. Now, let’s explore what else makes these eDSLs remarkable.

### **Why type-level eDSLs?**

Traditional, old-school value-level approaches certainly work. Encoding an HTTP API as a collection of ADTs and functions is straightforward, and most web server libraries follow this approach. They provide either an FP-like or OO-like set of tools for declaring APIs. *Tiny-http* in Rust, for example, keeps things simple and doesn’t rely heavily on type-level features. In contrast, *Axum* enforces some level of correctness and type safety through additional typing. However, neither comes close to the fully type-level design of *Servant*.

What can a type-level *Servant* do that *Axum* cannot? Perhaps its biggest advantage is the ability to generate a Swagger/OpenAPI description at compile time. Since the API is defined as a type, the compiler can decompose it and transform it into another representation of your choice.

Type-level eDSLs have several properties that make them particularly useful in certain cases:

* **Expressive and declarative.** You can build a fully type-level domain model that purely and declaratively captures the essence of your domain.
* **Noun-extensible.** New domain concepts—such as a new automaton state or HTTP method—can be introduced independently of existing ones. There's no need to modify your current model; new nouns naturally integrate, thanks to type-level interfaces.
* **Interpreted.** Once you have a type-level model, you can interpret it into something operational. For my showcase HTTP eDSL, I wrote two interpreters: one targeting the *Axum* runtime and another for *tiny-http*.
* **Verb-extensible.** New ways to interpret your type-level eDSL (new verbs) can be introduced independently. For example, you can introspect your HTTP API into OpenAPI, convert it into a string, or generate documentation—without affecting other interpreters. *(We’ll explore this noun-verb extensibility later in the article.)*
* **Type-safe.** Type-level lists accept only elements that conform to a specific type-level interface, ensuring that invalid notions are rejected. The same applies to type-level data fields—if a field is declared as a method (using the corresponding type-level interface), only methods can be assigned to it.
* **Making invalid states unrepresentable.** Design your domain model in a way that statically prevents inconsistent, absurd, or meaningless data.
* **Compile-time correctness checking.** If you need additional guarantees, you can connect your model to type-level validation algorithms that verify invariants at compile time. *(We’ll skip this topic here, but you can find more details in PTLD.)*
* **Compile-friendly.** The type-level interface approach is extremely simple for the compiler to process—no deep type inference is required, and no complex ambiguities need to be resolved. While large type-level models may have other drawbacks, slow compilation is not one of them.

### **Type-level domain modeling**

We typically solve programming problems by encoding domain concepts and scenarios. We know how to do this using ADTs and regular types—we're all familiar with domain modeling at the value level. But what does it take to go fully type-level? How powerful can our models be?

Looking at the examples above, we can see that type-level domain modeling in Rust is possible, though more limited than traditional approaches. With enough effort, almost anything can be expressed at the type level, but you may find yourself engaging in unnecessary code golf.

Currently, type-level domain modeling in Rust includes:

* **Type-level constants:** Integers, booleans, and characters.
* **Type-level collections:** Typically lists and trees. Rust doesn’t support type-level collections natively, so we’ll need to implement a custom list. With macros, usage can be made relatively convenient.
* **Type-level strings:** Rust lacks built-in support for these. My approach is straightforward—type-level strings are simply lists of type-level characters. Some macros provide a more ergonomic syntax.
* **Type-level records:** Empty structs with type parameters as fields.
* **Type-level interfaces:** We’ll use them to group certain user-defined types under an abstract umbrella, thus making them interchangeable. Fields in type-level structs will primarily reference these interfaces rather than concrete types. Additionally, type-level interfaces will integrate with type-level lists to enforce type safety.

Since I’m not aiming to repeat my entire *PTLD* book here, let’s jump straight to type-level interfaces.

### **Domain modeling with type-level interfaces**

Let’s examine the `MoveRouteImpl` user-defined type presented below. It’s based on the type `RouteImpl` that has five fields:

```rust
type MoveRouteImpl = RouteImpl<
   POST,
   tl_str!("/move"),
   MoveClauses,
   SupportedFormats,
   StringType>;
```

These fields are:

* **HTTP method** (the `POST` type)
* **Path** (a type-level string)
* **List of clauses** (such as capture parameters and query parameters)
* **List of supported formats** (for the return value)
* **Return type** (this method will return a string)

The values assigned to these fields are types—but of specific kinds. You can’t place `POST` where the `"/move"` string belongs. You can’t use integers where a string or a user-defined type is expected. You can’t mistakenly swap the list of clauses with the list of formats. These lists are distinct because they contain different kinds of types. This will become clearer when we define the `RouteImpl` type:

```rust
struct RouteImpl<
    Method: IInterface<IMethod>,
    Path: TlStr,
    Clauses: HList<IClause>,
    SupportedFormats: HList<IFormat>,
    ReturnType: IInterface<IType>>
  (PhantomData::<(...)>);  // Don't pay attention to this yet
```

It’s a parameterized struct with five type parameters. These type parameters function as fields, and, much like in OOP, fields reference interfaces. Here, you can see two direct type-level interfaces: `IInterface<IMethod>` and `IInterface<IType>`. Both lists also rely on type-level interfaces internally to determine which types are allowed as items. In this case, the lists can contain either `IClause` or `IFormat` types.

`IMethod`, `IType`, `IFormat`, and `IClause` are domain-specific kinds that I, as the developer, define for my types. *(And for those wondering—yes, this effectively serves as a working kind system in Rust.)* Finally, `TlStr` designates a field for a type-level string.

`IInterface<T>`, `HList<T>`, and `TlStr` are special traits from my type-level library (soon available in crates).

Kinds are just empty structs:

```rust
struct IMethod;
struct IClause;
struct IType;
struct IFormat;
```

The `RouteImpl` type also has its own kind, `IRoute`, which corresponds to the `IInterface<IRoute>` type-level interface. We link `RouteImpl`, `IRoute`, and `IInterface<T>` using the `Wrapper` type:

```rust
struct IRoute;

type Route<Method, Path, Clauses, Formats, ReturnType> =
   Wrapper<IRoute,
           RouteImpl<Method, Path, Clauses, Formats, ReturnType>>;
```

Type aliases we defined earlier would match, too:

```rust
type MoveRoute = Wrapper<IRoute, MoveRouteImpl>;
```

With this wrapper, we encapsulate the implementation type behind the type-level interface. Now, let's apply the same approach to some HTTP methods. In the simplest design, they might just be empty structs:

```rust
struct PostMethodImpl;
struct GetMethodImpl;

type POST = Wrapper<IMethod, PostMethodImpl>;
type GET  = Wrapper<IMethod, GetMethodImpl>;
```

While `PostMethodImpl` and `GetMethodImpl` have nothing in common, `POST` and `GET` share the same type-level interface, `IInterface<IMethod>`. If we need to add the `DELETE` method, we can do so without modifying existing code—`DELETE` will automatically be eligible for `IInterface<IMethod>` fields.

```rust
struct DeleteMethodImpl;

type DELETE = Wrapper<IMethod, DeleteMethodImpl>;


type DeleteGameRouteImpl = RouteImpl<
   DELETE,
   ...
   >
```

Here is a picture of this hierarchy.

\<picture\>

### **Type-level interfacing mechanism**

This interesting trait is a key to the approach:

```rust
trait IInterface<I> {
   type Interface;
}
```

The associated type `Interface` will represent user-defined kinds, such as `IMethod` and `IRoute`. Ideally, we would equate `Interface` to `I` if Rust supported this feature:

```rust
trait IInterface<I> {
   type Interface = I;   // Not supported
}
```

Instead, we use the Wrapper and its IInterface instance for the trick.

```rust
struct Wrapper<I, T> (PhantomData::<(I, T)>);

impl<I, T> IInterface<I> for Wrapper<I, T> {
   type Interface = I;         // Assigning the kind
}
```

Essentially, `Wrapper` is a type-level existential wrapper that encapsulates implementation types, making them appear identical under the same interface. The implementation types (*“nouns”*) may have complex structures, but we can still reference them in fields. Without this unification, building proper type-level domain models would be challenging.

An experienced type-level developer might ask, *“Wait, what’s wrong with using simple traits as kinds? We do this all the time in our libraries.”* Simple traits as kinds represent the following idiom:

```rust
trait BoolKind {}              // kind

struct True;                   // implementation types
struct False;

impl BoolKind for True {}      // tying the two
impl BoolKind for False {}

struct RWPermissions<
   Read: BoolKind,             // using kinds for fields
   Write: BoolKind>
 (PhantomData::<(Read, Write)>);

type MyPermissions = RWPermissions<True, False>;
```

At first glance, this approach seems sufficient for defining kinds. You could have a `MethodKind` trait for `POST` and `GET` implementation types and a `FormatKind` trait for `JSON`, `XML`, and `PlainText`.

```rust
trait FormatKind {}

struct JSON;
struct XML;
struct PlainText;

impl FormatKind for JSON {}
impl FormatKind for XML {}
impl FormatKind for PlainText {}
```

However, this approach is too simplistic. It may work in some cases, but it falls short for advanced domain modeling. One key issue is that traits are not first-class, meaning we can’t parameterize a type-level list with a trait to restrict it to certain types:

```rust
struct NotPossibleWithTraits<
    Formats: HList<FormatKind>,     // List of formats; won't compile
    Flags: HList<BoolKind>>         // List fo flags; won't compile
  (PhantomData::<(Formats)>);
```

In contrast, I use a first-class type-level tag, `IFormat`. I can simply declare a list as `HList<IFormat>` or a field as `IInterface<IFormat>`. This mechanism—combining `IInterface`, `ISomeKind`, and `Wrapper`—proves to be both powerful and more concise.

### **Interpretation and the universal evaluation mechanism**

The type-level model of an HTTP API and the type-level *Game of Life* rule don’t do anything on their own—they are just types. The real work is yet to come. Our models are declarative, interpretable eDSLs, so to make them useful, we need to interpret them.

Interpreting these types means decomposing them layer by layer using traits (type classes). First, let’s explore a naive approach, then move on to a more universal evaluation mechanism.

To convert a type-level `Route` value into a real web server (e.g., *tiny_http*), we might define the following type class:

```rust
trait InterpretRouteToTinyHttp {
    fn interpret() -> ();
}
```

Next, we define instances for each route type and construct a web server. In the following code, notice how we unpack the wrapper to access the implementation type `RouteImpl`:

```rust
impl<Method, Path, Clauses, Formats, ReturnType>
   InterpretRouteToTinyHttp
 for Wrapper<IRoute,        // unpacking the implementation type
             RouteImpl<Method, Path, Clauses, Formats, ReturnType>>
 where
     Method: IInterface<IMethod> + InterpretMethodToTinyHttp,
     Path: TlStr,
     Clauses: HList<IClause> + InterpretClauseListToTinyHttp,
     Formats: HList<IFormat> + InterpretFormatListToTinyHttp,
     ReturnType: IInterface<IType> + InterpretTypeToTinyHttp
{
   fn interpret() -> () {
       todo!()  // do something real, call other type classes
   }
}
```

We’ll need as many type classes as there are domain concepts, and we must define instances for all wrapped implementation types.

Later, we might want to interpret the model not only into *tiny_http* but also into *Axum*. This would double the number of type classes and instances.

```rust
trait InterpretRouteToTinyHttp ...
trait InterpretMethodToTinyHttp ...
trait InterpretFormatToTinyHttp ...
trait InterpretTypeToTinyHttp ...
trait InterpretClauseToTinyHttp ...


trait InterpretRouteToAxum ...
trait InterpretMethodToAxum ...
trait InterpretFormatToAxum ...
trait InterpretTypeToAxum ...
trait InterpretClauseToAxum ...
```

Or we’d want to introspect the API and generate a string description.

```rust
trait DescribeRoute ...
trait DescribeMethod ...
trait DescribeFormat ...
trait DescribeType ...
trait DescribeClause ...
```

To solve this obvious problem and to enable more interesting possibilities, I propose the universal evaluation mechanism.

```rust
trait Eval<Verb, Res>{
   fn eval() -> Res;
}
```

The `Eval` type class will handle everything. It has two parameters: `Res`, which represents the return type specific to the action, and `Verb`, which defines the action itself. Verbs can be simple empty structures or carry additional type-level information if needed by the interpreting code.

```rust
struct TinyBuildRoute;
struct TinyBuildMethod;
struct TinyBuildClauses;
struct TinyBuildFormats;
struct TinyBuildType<T>(PhantomData::<T>);
```

We write instances of Eval similarly to what we did before. We use a verb and instantiate the type class for wrapped implementation types:

```rust
impl<Method, Path, Clauses, Formats, ReturnType>
   Eval<TinyBuildRoute, ()>   // using a verb
 for Wrapper<IRoute,          // unpacking the implementation type
             RouteImpl<Method, Path, Clauses, Formats, ReturnType>>
 where
     Method: IInterface<IMethod> + Eval<TinyBuildMethod, String>,
     Path: TlStr,
     Clauses: HList<IClause> + Eval<TinyBuildClauses, ()>,
     Formats: HList<IFormat> + Eval<TinyBuildFormats, ()>,
     ReturnType: IInterface<IType>
        + Eval<TinyBuildType<ReturnType>, ()>
{
   fn interpret() -> () {
       let method = Method::eval();      // do something real
       Clauses::eval();     // do something real
       Formats::eval();     // do something real
       ReturnType::eval()   // do something real
   }
}
```

The example demonstrates how to invoke interpreters for each field, but having numerous small interpreters isn’t mandatory. Some domain concepts may have dedicated interpreters, while others can be processed together (for example, *Method* and *Path* in the snippet). The way you design your interpreters depends on the eDSL and the final outcome you want to achieve. Each interpreter will perform different tasks, but all will follow a uniform structure.

The `Eval` type class, along with the compiler, will handle your distinct yet consistently interfaced implementation types. The compiler will correctly select the appropriate instance for a wrapped implementation type or stop the compilation if one is missing.

The following two interpreters convert HTTP methods into strings. One of them will be invoked for *Method*, depending on the content of the `IInterface<IMethod>` field:

```rust
impl Eval<TinyBuildMethod, String>
 for Wrapper<IMethod, PostMethodImpl> {
   fn eval() -> String {
     "POST".to_string()
   }
}


impl Eval<TinyBuildMethod, String>
 for Wrapper<IMethod, GetMethodImpl> {
   fn eval() -> String {
     "GET".to_string()
   }
}
```

For more advanced evaluation, use an extended version of the type class that accepts an external context:

```rust
trait EvalCtx<Ctx, Verb, Res>{
   fn eval_ctx(ctx: Ctx) -> Res;
}
```

We no longer need dozens of custom traits. **Discoverability improves**—you might forget the name of a trait or a verb, but you won’t forget `Eval`. Searching for `Eval` instances will immediately lead you to the relevant implementations.

The code is **highly uniform**. Once you learn the pattern, applying it across different cases becomes straightforward. In *PTLD*, I even define this as a design principle: *dumb but uniform*. This is particularly crucial in type-level programming, where multiple approaches exist, making it difficult to keep track of them all. While this issue is less severe in Rust, in Haskell, the lack of uniformity becomes a major pain point due to its vast array of type-level features and countless possible combinations.

Finally, this approach **solves the Expression Problem**. New nouns (*additional domain concepts*) and new verbs (*new operations on those concepts*) can be introduced independently—without recompiling existing code. For instance, after adding the `DELETE` type, wrapping it, and using it in your routes as `IInterface<IMethod>`, the compiler will simply prompt you to provide an `Eval` instance. You can define this instance outside the main implementation, even in a separate package.

This **noun-verb extensibility** may not always be the most convenient, but it comes naturally—*for free*—in any language that supports the type-level interfaces approach.

picture

### **Conclusion**

Type-level programming is not easy. I’ve only scratched the surface, but many important topics remain unexplored:

* **Type-level lists and type-level strings**
* **Type-level data types and collections**
* **Correctness and making invalid states unrepresentable**
* **Type-level domain modeling and eDSL design**
* **Advanced extensibility**
* **Type-level evaluation models**
* **Design principles and type-level design patterns**
* **Application architectures**

All of this forms the foundation of the methodology I call *Pragmatic Type-Level Design*. You can learn it in depth from my book, where these topics are systematically structured and illustrated with fun, well-elaborated examples.

If *PTLD* reaches **1,000 readers** (*which is a lot!*), I’ll add four more chapters and expand the coverage to two additional languages, beyond Haskell, Rust, and Scala 3, that support these techniques.

I’m also considering writing a **dedicated book on type-level programming in Rust**, but I can only do that if there’s clear commercial potential. Writing advanced, rare books is incredibly time-consuming and expensive. If you want to see more work like this, please support my current book—it benefits everyone.

[Pragmatic Type-Level Design (LeanPub, 2024)](https://leanpub.com/pragmatic-type-level-design)
[PTLD's repo](https://github.com/graninas/Pragmatic-Type-Level-Design)
[Functional Design and Architecture: Examples in Haskell (Manning Publications, 2024)](https://www.manning.com/books/functional-design-and-architecture)

**Contact me**

* X: @graninas
* LinkedIn: [linkedin.com/in/graninas](https://www.linkedin.com/in/graninas)
* Email: graninas@gmail.com
