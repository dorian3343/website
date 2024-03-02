---
title: FAQ
weight: 7
---

Welcome to the FAQ page! Here, we've compiled all the questions that you might not find answers to on other documentation pages. This section was primarily created as a reference for ourselves - when developing something as complex as a programming language, it's easy to forget the reasons behind certain decisions. So, we thought, why not document at least those related to the language's external aspects, right here on the website!

## Is Neva "classical FBP"?

No. But it inherits so many ideas from it that it would be better to use word "FBP" than anything else. There's a [great article](https://jpaulm.github.io/fbp/fbp-inspired-vs-real-fbp.html) by mr. J. Paul Morrison (inventor of FBP) on this topic.

Now here's what makes Neva different from classical FBP:

- C-like syntax as a textual representation. Classical FBP syntax is somewhat esoteric. Nevalang programs are 100% declarative though.
- Nevalang assumes you don't have to program in lower level language in normal cases while FBP assumes your elementary components could be written in some control-flow language by you.
- Neva introduces builtin observability via event listener and messages tracing, This is out of the scope of classical FBP
- Existing FBP implementations are essentially interpreters. Neva has both compiler.
- Neva is statically typed while FBP isn't. FBP's idea is that you write code by hand in statically typed langauge and then use it in a non-typed FBP program, introducing runtime type-checks where needed
- Neva have _runtime functions_. Mr. Morrison did not approved the idea of having builtin "atomic" components like `Adder`
- Nevalang has hierarchical structure of program entities with visibility scope and package management.
- Neva leverages existing Go's GC, FBP on the other hand introduces IP's life-cycle on top of that.
- Neva's concurrency model runs on top of Go's scheduler which means it uses CSP as a lower-level fundament. FBP implementations on the other hand use shared state internally.
- Different abstraction for "bound inports" - _Const/literal senders_ instead of FBP's IIPs.
- Different naming for a lot of things: messages instead of IPs, nodes instead of processes, etc.

## Why need array-ports?

Every time we need to somehow combine/accumulate/reduce several sources of data into one e.g.

- create list of 3 elements based on outputs of 3 outports
- create structure with field-values from several outports
- substract values from left to right

Ok but can't we substract values and do other stuff like that by simply passing lists around? Well, we have to create that list right somehow? It's fine if you already have it (let's say from JSON file you got from server) but what if you need to build it?

## Why component can't read from it's own array-inports by index?

Imagine you do stuff like:

```neva
in.foo[0] -> ...
in.foo[1] -> ...
```

Now what will happen if parent node will only use `0` slot of your `foo` array-inport? Should it block forever? Or maybe should the program crash? Sounds not too good.

The other way we could handle this is by making analyzer ensure that parent of your component uses your `foo` array-inport with exactly `0` and `1` slots. The problem is that makes array-ports useless. Why even have them then? The whole point of array-ports is that you don't know how many slots are going to be used. And that makes your component flexible. It allows you to create components that can do stuff like "sum all numbers from all inports no matter how many of them are present".

Besides, you can already say "use my component with exactly two values" already and you don't need any array-ports for that at all! All you need in that case is simply create two inports.

Having that said we must admit that it's impossible to allow component read form it's own array-inports by index and still having type-safety.

Also think about _variadic arguments_ in Go. It's not safe to refer to `...args` by index (even though it's possible because Go compiler won't protect you).

## Why component can read from sub-node's array-outports by index?

Isn't it unsafe to read from array-outports by index? We've restricted that for component itself by banning the ability to read form self outports by index. Why allow read from sub-node outports by index then?

Well, it turns out there are critical cases where we need that. One of them is "routing" - where you have some data on the input and you need to figure out, based on some predicate, where to send it further. Like if you have a web-server and you received a request, you need to route it to specific handler based on the path that this request contains.

It's possible to do that with sequence of if-else though but that would be really tedious and error-prone. That also would make your network more nested and less straightforward.

### Can't we implement syntax sugar for that?

It's possible to introduce some sort of syntax sugar where user interacts with array ports and under the hood it's just a bunch of if-elses. But that actually makes no sense. If we have array-outports as a part of the language interface, we have them anyway. We also have use-cases for array-inports which means there are other reasons why have array ports. And finally it would be better for performance to have one low-level control-flow component implemented in implementation langauge and not Nevalang instead of compiling one high-level component to another big high-level component. One might ask - but we did that for Lock, what's the difference? The thing is with lock we are not replacing one component usage with the another, like we would in case of replacing some kind of "router" with bunch if if-elses. We simply insert implicit code, that is assumed by the higher level constructs like only exist at the level of the source code and not the real machinery.

## Why outports usage is optional and inport usage is required?

Indeed when component `A` uses `B` as it's sub-component (when it instantiates a _node_ with it) in it's _network_ it's _enforced_ to use _all_ the inports of `B` and it's _at least one_ outport. It doesn't have to use all the outports though.

This is because inports are requirements - they are needed to receive the data that component _needs_ to produce result. Outports on the other hands are options. They are results that parent network might need to a sertain degree. For instance if `B` have outports `foo` and `bar`, it's completely possible that `A` only needs `foo` and have nothing to do with `bar`.

This leads us to the need of the `Void` (builtin) component. This is the only component that doesn't have outports. It is used for discarding the unwanted data. If there would be no syntactic sugar for that, then we would have to explicitly create `void` nodes and use it in places like this:

```neva
nodes {
    b B
    void Void
}
net {
    // ...
    b.bar -> void.v // discard all messages from `bar` outport
}
```

It's not the problem that it's tedious (even though it is, imagine having 10 unwanted outports in your network which is completely possible). The real problem is that by discarding some outports user is in danger of programming the dataflow in the wrong way.

Imagine that `B` has outports `v` (for valid results) and `err` (for error messages). It fires either `v` or `err` and never both at the same time. And we want out program to terminate if there's nothing to do left. Consider this code:

```neva
component Main(start any) (stop any) {
    nodes {
        b B
        destructor Destructor
        print Print
    }
    net {
        :start -> b:sig
        b:err -> destructor:msg // ignore the `err` outport, only handle happy path
        b:v -> print:v
        print:v -> :exit
    }
}
```

We print the success result and then terminate. If there is no success result and only error we well... do nothing. And that's bad. What we should do instead is this:

```neva
// ...
net {
    in.start -> b.sig

    // print both result and error
    b.err -> print.v
    b.v -> print.v

    // and then exit
    print.v -> out.exit
}
```

As you can see it's easy to get in trouble by ignoring some outports (especially the error ones). If user wouldn't have the ability to do so he would have to do _something_ with `err` message. Anyway there would still be two problems...

1. Even then user still _can_ send the data in the wrong way. E.g. send the `err` message back to `b.sig` or `print` it but then send the `print.v` back to the `print` forming an endless loop. This kind of _logical_ mistakes are hard to catch. Making the language _that_ safe would also make it much more complicated (think of Haskell or Rust (where we still have such kinds of problems!)).
2. Sometimes we have _nothing to do_ with unwanted data. We don't wanna print it or even send downstream (because that would simply delay the question what to do with unwanted data). This is the reason why `Void` doesn't have outports. Otherwise a paradox arises.

This leads us to a conclusions:

- There must be a way to omit unwanted data, whether it's explicit (`Void`) or implicit sugar
- It's impossible to make langauge 100% safe without sacrificing the simplicity of use

As we saw explicit Void doesn't solve these problems so why not introduce sugar? Let's allow user to simply omit unwanted data and let the compiler implicitly insert `Void` under the hood. The logical mistakes? Well... They are "unsolvable" anyway.

## Why there's no int32, float32, etc?

Because that's a simple language. Lack of ability to configure bit-size of the number but still being able to choose between integers and floats is the compromise that seems to be reasonable. Probably Python is a good example of that too.

## Why have integers and floats and not just numbers?

1. Overflow issues: if you only have `number`, probably represented as a `float64` in memory, your maximum safe number is bigest float64. Integer can store "bigger" values because it doesn't have to store (floating) precision. This is especially important when you work with big numbers.

2. Performance Overhead: Floating-point operations are generally slower than integer operations. In a system where all numbers are floating-point, operations that could have been more efficient with integers suffer a performance penalty.

3. Predictability in Comparisons: Floating-point arithmetic can lead to non-intuitive results due to precision errors, making comparisons and certain calculations (like summing a large list of numbers) less predictable.

4. Lack of Type Safety: The absence of distinct types can lead to bugs that are hard to detect, as the language won't provide errors or warnings when performing potentially erroneous operations between different kinds of numeric values.

## What is the motivation behind putting exactly these entities under builtin package?

1. Frequently used
2. Used internally by compiler (desugarer)

## Why Emitter implemented like an infinite loop?

Const nodes are implemented like infinite loops that constantly sends messags across their receivers. This covers all the usecases but also requires locks because we usually want control when we send constant messages.

Alternative to that design would be "trigger" semantics where we have some kind of `sig` inport for const nodes. Instead of starting at the program startup such trigger component would wait for external signal and then do the usual const stuff (infinite loop with sending messages).

**The problem #1 with this approach - we still needs locks**. It's not enough to trigger infinite loop. E.g. in "hello world" example nothing would stop `msg` const node to send more than 1 `hello world` message to `print`.

**Possible solution for that would be to change semantics and remove infinite loop logic**. Make const node send signal only after we trigger it via sig port. The problem with this approach is that there is many usecases where we want infinite loop behavior. Think of initial inports - e.g. `requestSender` component with `data` and `url` inports where `data` is dynamic and `url` is static. It's not enough to send static url value just once (`requestSender` must remember it, we don't go that way because that leads to bad design where components know where they get data from - this is huge violation of transport vs logic separation).

This problem by itself is fixable with using external sources like signals. When we have some static inport we usually have some kind of dynamic data that must be used in combination with it. Even though it would lead to making networks more complicated (locks do this too though), it's possible solution. But we have another problem:

**Problem #2** - `$` syntax sugar.

Another problem with previous solution (const nodes have sig inport and they send one message per one signal) is how use `$` syntax sugar.

Currently it's possible to _refer constants_ in network like this:`$msg -> ...`

This won't be the thing because we have to have not just entity reference but regular ports like `$msg.sig` and `$msg.v`. This is not a disaster but rather strange API.

Where this `$msg` node come from? Is it clear to me as a user that there are implicit nodes $ prefix for every constant that I can refer? Why these `sig` and `v` inports? Because this is how `std/builtin.Const` works? Why do I have to know this? Why do I have to know how syntax sugar is implemented under the hood in compiler?

Finally another possible solution to that could be `Trigger` components in combination with `Const` components. The difference would be that const behaves like infinite loops that requires locks and triggers behaves like single sending triggers (no lock required).

Problems with this solution are:

1. Now we have 2 ways to do the same thing. Do I need to use const? Or trigger? When to use what?
2. Trigger either must be used in combination with `#runtime_func_msg` directive (this violates principle that user must be able to program without directvies) or there must be sugar for triggers.

It's possible in theory to create sugar for triggers but the language could be too complicated with so many syntax features. This conclusion by itself is questionable but in combination with the first problem - having 2 ways to use static values. Looks like it's better not to have triggers.

All this leads to a conclusion that the only good semantic for constants is the current ones - infinite loops that requires locks. The need to have locks is not a fancy thing by itself, but "then connections" sugar made this pretty simple.

## Why have sub-streams?

In programming we range over collections all the time. We do that either via _loops_ or higher order functions (recursion). There's a problem with both of these in FBP:

We don't have _code as data_ (we can't pass components around like we pass functions in conventional languages)

To implement loop we need:

1. Mutable state. (We don't have one! We can simulate one using network loops tho). After we simulate mutable
2. Condition to check whether current cursor is still less than the length of the list
3. Length of the list

It would be very verbose to do such things all the time so we can imagine some kind of generic `for` component that takes `list<T>` and sends single values `T`.

The biggest problem is - how do we know that list ended? How do we know that the previous element was the last one and the current element is the first element of the new list that just arrived? Without knowing this we loos information about the list boundary. And that is huge problem. In conventional programming we always know that. Without this simplest iteration patterns like `map` are impossible to implement.

Possible solution to this (without introducing sub-streams) would be adding some kind of _signal_ that "the list just ended". One might think that `For` component could simply have two outports `v` and `sig`. No, it can't. In this case `sig` cannot be separate port because it needs to be in the exact same _steam_ as the elements themselves. Otherwise it's unclear how to be sure that we synchronized both streams (flows) together. Streams are concurrent and the order of messages across different streams is often unpredictable.

That leads us to conclusion - such `For` component must have one outport (or at least it must not have separate `sig` outport). It instead must send not just `T` values (single elements of the colletion), but instead it must send some kind of structures. The shape must be something like this

```neva
types {
    Element<T> {
        v T
        isLast bool
    }
}
```

Congratulations! You just discovered sub-streams.

## Why sub-streams are not like in classical FBP?

John Paul Morrison, creator of Flow-Based Programming created _Sub streams_ as a part of the FBP. The problem that he was solving wasn't just iteration over collections. It was work with structured data. Sub-streams are the way we transfer structured objects in his paradigm.

In Nevalang we have `struct`, `map` and `list` for that. We don't need to create "flat nested" sub-stream like this `( (1 2 3) (4 5 6) ) ( (7 8 9) )` to move two lists, we can simply move them like regular messages across the stream `-> l1 l2 ->`. The downstream component that receives `l1` and `l2` can then unpack them into sub-streams and process their individual elements.

At the end it must pack them pack tho. This is _maybe_ where classical FBP outperforms Nevalang. We have to spend time on destructuring and structuring back. However, the **data in the outside world is structured**. We usually work with some kind of relational data, JSON, Protobuf, etc.

## Isn't it the problem that component that works with type `T` cannot operate on `SubStreamItem<T>`

This is just how type-system works. We don't want to have a lot of special cases here and there. It's not a big deal also.

If you have a component `C1` that takes `T` and you want to operate on `SubStreamItem<T>` all you need is to create a wrapper. That wrapper will receive `SubStreamItem<T>` and use `C1` inside of it with `.v` struct selection.

If you need to continue sub-stream you simply send `SubStreamItem<T>` from you wrapper component downstream. Or `SubStreamItem<WhateverYouWant>` (probably preserving `isLast` value).

It's either you continue sub-stream or you do not. Depending on what your're doing (maybe you're counting sub-stream items so you just sends `int` eachtime sub-stream ends).

## Why `:stop` of the `Main` is't `int`?

This is the question about why `:stop` isn't interpreted as exit code.

The things is - you don't always have `int` as your exit condition. That's why it's `any`.

Ok, but why then we don't check if that `any` is actually `int` under the hood and then interpret it as an exit code?

Well, we can do that. But that would lead to situations where you accidentally have `int` like your exit condition but don't actually want it to be your exit code. Such cases are non obvious and will require you to somehow check that you send exit code you want.

This problem gets bigger when you have `any` or _union_ `... | int` outport that is directed to `:stop` - you'll have to check whether value isn't an `int`. Otherwise you're at risk of terminating with wrong code.

**Exit codes are important**. Shell scripts and CI/CD depends on that. Most of the time you want your exit code to be `zero`. Non-zero exit code is not happypath, it's more rare. Having corner case like a base design decision is not what we want.

## Why structural subtyping?

1. It allowes write less code, especially mappings between records, vectors and maps of records
2. Nominal subtyping doesn't protect from mistake like passing wrong value to type-cast

## Why have `any`?

First of all it's more like Go's `any`, not like TS's `any`. It's similar to TS's `unknown`. It means you can't do anything with `any` except _receive_, _send_ or _store_ it. There are some [critical cases](https://github.com/nevalang/neva/issues/224) where you either make your type-system super complicated or simply introduce any. Keep in mind that unlike Go where generics were introduced almost after 10 years of language release, Neva has type parameters from the beggining. Which means in 90% of cases you can avoid using of `any` and panicking.

## Why only primitive messages can be used as literal network senders?

1. Their type is relatively easy to infer by the compiler. To cover structs, lists, maps and other stuff we gonna either use verbose syntax like `|User {age: 32}| -> ...|` or implement real type inference which is very hard and won't be consistent with other language (no inference in other places)
2. Allowing to use complex messages as senders would lead to very hard to read networks. Imagine e.g. list of structs with maps.

## Why there's no syntax sugar for `list<T>`?

Many languages provide syntax like `[]T` or `T[]`. The problem is exactly that they do so differently. Anyone who worked with Go and TS in single project knows this problem when brain always confused with which syntax to use. Graphql has better syntax `[T]` but it's reserved in Nevalang for array-ports.

As a side bonus of current solution - `list<T>` is consistent with the rest syntax, no need for new syntax rules and (this is probably most important) you don't have 2 possible options for how to express your list. If you have `pub list<T>` in stdlib then you should be able to use it like any normal type isn't it?

## Why there's no syntax sugar for `maybe<T>`?

The reason is exactly the same as for `list<T>`. Sometimes it's `?T` and sometimes it's `T?`. Also having sugar for maybe but not for list could look like a strange corner-case.

## Why there's inconsistent naming in stdlib?

A few components in stdlib don't follow style-guide in their naming. Examples are `Len`, `Eq` and so on. The thing is - they do not follow naming in conventional languages too. E.g. in almost any controlflow language we name functions like verbs but `len` function in Go, Odin, etc is not `getLen()`. So it's just simpler to say `Len` than `LenGetter`. Also should it be "getter" or "reader" or whatever? However this relates to most basic stuff and most things in stdlib are consistent with the style guide.

## What is the reasoning behind naming in Nevalang?

So it's not classic FBP with processes, IPs, IIPS, etc. The reasoning was pretty much simple - stuff should look familiar for most of the programmers. Kinda like in "Interslavic" language if that makes sense. They have to shift paradigm so let's help them at least for a little.

## Why struct and map literals require `:` and `,` and `struct` declaration not?

Indeed you define structures like this:

```neva
type struct Point {
    x int
    y int
}
```

And not like this

```neva
type struct Point {
    x: int,
    y: int,
}
```

So why you define struct/map literals with colons and commas? Well, the answer is that this way it almost valid JSON just like in Python in JS. Except (optional) trailing comma (afterall JSON compatibility is not our goal). This also good to distinguish easily between type and const expressions. Finally this is how we do in JS/TS, Python and Go.

## Can't we have shorter syntax for connections?

There was an idea to have sugar for connections where sender and receiver has same ports

```
:sig -> scanner:sig
scanner:data -> logger:data
logger:sig -> :sig
```

To look like this

```neva
in -> scanner
scanner -> logger
logger -> out
```

The problem is that this

```neva
foo:x -> bar:x
foo:y -> bar:y
```

becomes this

```neva
foo -> bar
foo -> bar
```

So it's impossible to tell what port we mean. This could make sense only for components with single in/out port but there's not a lot of such components.

We could in theory limit this syntax to a form where you can have only one `foo ->` connection but that would lead to inconsistency

```
foo -> bar
foo:y -> bar:y
```

And the last but not least. How am I suppose to figure out what port `foo -> bar` these two use? I have to look into their interfaces which is not cool.
