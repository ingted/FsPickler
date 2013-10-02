# FsPickler

A fast, general-purpose binary serializer for .NET written in F# 
that doubles as a pickler combinator library.

* Based on the notion of pickler combinators.
* Provides an automated, strongly typed, pickler generation framework.
* Full support for the .NET type system, including open hierarchies.
* Supports all core serializable types, including types that implement the ``ISerializable`` interface.
* Highly optimized for the F# core types.
* Performance about 5-20x faster than the default .NET serializers.

### Basic Usage

The following snippet presents the basic serialization/deserialization API for FsPickler:

```fsharp
open FsPickler

let fsp = new FsPickler()

// serialize without explicit pickler use
fsp.Serialize<int list option>(stream, Some [1;2;3])
fsp.Deserialize<int list option>(stream)

// serialize with explicit pickler use
let pickler : Pickler<int list option> = fsp.GeneratePickler<int list option> ()

fsp.Serialize(pickler, stream, Some [1; 2; 3])
fsp.Deserialize(pickler, stream) : int list option
```

All generated picklers are strongly typed; pickling is performed efficiently
without intermediate boxings. Picklers are generated at runtime using reflection
and expression trees that are aggressively cached for future use.

### Pickler Combinators

FsPickler offers experimental support for generating user-defined picklers using combinators:

```fsharp
open FsPickler
open FsPickler.Combinators

let p : Pickler<int * string option> = 
    Pickler.string 
    |> Pickler.option 
    |> Pickler.pair Pickler.int
    
let data : byte [] = pickle p (42, Some "")
unpickle p data
```

The combinator library includes all primitives as described in Andrew Kennedy's 
[Pickler Combinators](http://research.microsoft.com/en-us/um/people/akenn/fun/picklercombinators.pdf)
such as ``wrap`` and ``alt``. Fixpoint combinators for declaring recursive picklers are also available:
```fsharp
val Pickler.fix : (Pickler<'T> -> Pickler<'T>) -> Pickler<'T>

type Peano = Zero | Succ of Peano

let pp : Pickler<Peano> =
    Pickler.fix(fun peano ->
        peano
        |> Pickler.option
        |> Pickler.wrap (function None -> Zero | Some p -> Succ p)
                        (function Zero -> None | Succ p -> Some p))
                        
Succ (Succ Zero) |> pickle pp |> unpickle pp
```
The library comes with the ``array``, ``array2D``, ``list``, ``seq``, ``set`` and ``map`` 
combinators that are used to build picklers for the corresponding generic types. 
It should be noted that ``Pickler.seq`` serializes sequences using eager evaluation.

When it comes to generic types, picklers can be defined using user-defined combinators:

```fsharp
type BinTree<'T> = Leaf | Node of 'T * BinTree<'T> list

let binTree (ep : Pickler<'T>) =
    Pickler.fix(fun tree ->
        tree
        |> Pickler.list
        |> Pickler.pair ep
        |> Pickler.option
        |> Pickler.wrap (function None -> Leaf | Some (t,c) -> Node(t,c))
                        (function Leaf -> None | Node (t,c) -> Some(t,c)))
                        
Node(2,[]) |> pickle (binTree Pickler.int)
```
or it could be done using automatic resolution of type parameters:

```fsharp
let binTree<'T> =
    Pickler.fix(fun tree ->
        tree
        |> Pickler.list
        |> Pickler.pair Pickler.auto<'T>
        |> Pickler.option
        |> Pickler.wrap (function None -> Leaf | Some (t,c) -> Node(t,c))
                        (function Leaf -> None | Node (t,c) -> Some(t,c)))


Node([1],[Leaf ; Leaf]) |> pickle binTree
```

### Custom Pickler Declarations

Since F# lacks mechanisms such as type classes or implicits, 
pickler resolution can only be performed by means of reflection.
This is done using the following design pattern:

```fsharp
[<CustomPickler>]
type CustomClass<'T> (x : 'T) =

    member __.Value = x

    static member CreatePickler (resolver : IPicklerResolver) =
        let ep = resolver.Resolve<'T> ()
        ep |> Pickler.wrap (fun x -> CustomClass(x)) (fun c -> c.Value)
```
This tells the pickler generator to yield a pickler for the given type
using that particular factory method. The ``IPicklerResolver`` argument provides
a handle to the pickler generator and can be used for recursive types:
```fsharp
[<CustomPickler>]
type RecursiveClass(?nested : RecursiveClass) =

    member __.Value = nested

    static member CreatePickler (resolver : IPicklerResolver) =
        let self = resolver.Resolve<RecursiveClass> ()
        self 
        |> Pickler.option 
        |> Pickler.wrap (fun x -> RecursiveClass(?nested = x)) (fun rc -> rc.Value)


let p = Pickler.auto<RecursiveClass>

RecursiveClass(RecursiveClass()) |> pickle p |> unpickle p
```
