ocaml-optics
------------

*Status: WIP & Experimental*

Optics using [existentials](https://www.tweag.io/blog/2022-05-05-existential-optics/).

```ocaml
# #require "optics";;
# open Optics;;
```

- [Usage](#usage)
    - [Lenses](#lenses)
        - [Constructing Lenses](#constructing-lenses)
        - [Composing Lenses](#composing-lenses)
        - [Getting and Setting Values](#getting-and-setting-values)
    - [Prisms](#prisms)
        - [Constructing Prisms](#constructing-prisms)
        - [Composing Prisms](#composing-prisms)
        - [Getting and Setting Values](#getting-and-setting-values)
    - [Optionals](#optionals)
        - [Deeper Composition with Infix Operators](#deeper-composition-with-infix-operators)
    - [Optionals from Lenses and Prisms](#optionals-from-lenses-and-prisms)


## Usage

```ocaml
type t = {
    point : point;
    props : prop list;
}

and point = Point2D of point2d | Point3D of point3d

and point2d = { x: float; y: float }

and point3d = { x : float; y: float; z: float}

and prop = { key : string; value : prop_value }

and prop_value  = String of string | Int of int
```

and we'll make a quick value too.

```ocaml
# let example = 
  {
    point = Point2D { x = 1.0; y = 2.0 };
    props = [ { key = "Hello"; value = String "World" }; { key = "Bonjour"; value = String "Monde" } ]
  };;
val example : t =
  {point = Point2D {x = 1.; y = 2.};
   props =
    [{key = "Hello"; value = String "World"};
     {key = "Bonjour"; value = String "Monde"}]}
```

### Lenses

A lens allows you to get and set fields of a record. Defining one requires to provide two functions that break apart a record into a field and the rest of the record, and another function which builds the record back together.

#### Constructing Lenses

```ocaml
# let props = Lens.V ((fun t -> (t.props, t)), (fun (props, t) -> { t with props }));;
val props : (t, prop list) Lens.t = Optics.Lens.V (<fun>, <fun>)
# let point = Lens.V ((fun t -> (t.point, t)), (fun (point, t) -> { t with point }));;
val point : (t, point) Lens.t = Optics.Lens.V (<fun>, <fun>)
# let key = Lens.V ((fun t -> (t.key, t)), (fun (key, t) -> { t with key }));;
val key : (prop, string) Lens.t = Optics.Lens.V (<fun>, <fun>)
# let value = Lens.V ((fun t -> (t.value, t)), (fun (value, t) -> { t with value }));;
val value : (prop, prop_value) Lens.t = Optics.Lens.V (<fun>, <fun>)
```

#### Composing Lenses

Lenses compose nicely in the way you might expect. Given a `('a, 'b) Lens.t` and a `('b, 'c) Lens.t` we compose the two to get a `('a, 'c) Lens.t`.

```ocaml
# let key_at n = Lens.(props >> nth n >> key);;
val key_at : int -> (t, string) Lens.t = <fun>
# let value_at n = Lens.(props >> nth n >> value);;
val value_at : int -> (t, prop_value) Lens.t = <fun>
```

#### Getting and Setting Values

```ocaml
# Lens.(get (key_at 0) example), Lens.(get (value_at 0) example);;
- : string * prop_value = ("Hello", String "World")
```

```ocaml
# Lens.set (key_at 0) example "Salut" |> Lens.get (key_at 0);;
- : string = "Salut"
```


### Prisms

Prisms are to sum-types (variants) what lenses are to product types (records). The difference is we need to encode the idea that a variant could be the constructor we want or something else entirely. We do this with the `('a, 'b) result` type.

#### Constructing Prisms

```ocaml
# let point2d = 
  let into = function
    | Point2D f -> Ok f
    | v -> Error v
  in
  let out_of = function
    | Ok f -> Point2D f
    | Error v -> v
  in
  Optics.Prism.V (into, out_of);;
val point2d : (point, point2d) Prism.t = Optics.Prism.V (<fun>, <fun>)
# let point3d = 
  let into = function
    | Point3D f -> Ok f
    | v -> Error v
  in
  let out_of = function
    | Ok f -> Point3D f
    | Error v -> v
  in
  Optics.Prism.V (into, out_of);;
val point3d : (point, point3d) Prism.t = Optics.Prism.V (<fun>, <fun>)
```


And for property values (only `string` shown for brevity).

```ocaml
# let string = 
  let into = function
    | String s -> Ok s
    | v -> Error v
  in
  let out_of = function
    | Ok f -> String f
    | Error v -> v
  in
  Optics.Prism.V (into, out_of);;
val string : (prop_value, string) Prism.t = Optics.Prism.V (<fun>, <fun>)
```

#### Composing Prisms

Prisms compose just like [lenses](#composing-lenses).

```ocaml
# Prism.(>>);;
- : ('a, 'b) Prism.t -> ('b, 'c) Prism.t -> ('a, 'c) Prism.t = <fun>
```

#### Getting and Setting Values

Getting and setting values works much the same way as lenses except getting values can return `None` if the you are trying to get a different variant constructor.

```ocaml
# let p = example.point;;
val p : point = Point2D {x = 1.; y = 2.}
# Prism.get point3d p;;
- : point3d option = None
# Prism.get point2d p;;
- : point2d option = Some {x = 1.; y = 2.}
# Prism.set point2d {x = 1.; y = 2.} ;;
- : point = Point2D {x = 1.; y = 2.}
```

### Optionals

Optionals are lenses but with optional values for the type under focus.

```ocaml
# #show_type Optional.t;;
type nonrec ('s, 'a) t = ('s, 'a option) Lens.t
```

Optionals are actually a middle-ground between prisms and lenses that allow us to compose a lens and prism.

```ocaml
# let t_to_point2d = Optional.(point >& point2d);;
val t_to_point2d : (t, point2d) Optional.t = Optics.Lens.V (<fun>, <fun>)
# Lens.get t_to_point2d example;;
- : point2d option = Some {x = 1.; y = 2.}
```

#### Deeper Composition with Infix Operators

The library comes with a `Optics.Infix` set of operators that can help with deeply nested composition. For example getting the string value of the `nth` property in the `example` value.


```ocaml
# open Infix;;
# let t_to_prop_value_string n = props & Lens.nth n & value >& string;;
val t_to_prop_value_string : int -> (t, string option) Lens.t = <fun>
# Lens.get (t_to_prop_value_string 0) example;;
- : string option = Some "World"
```

Within the `Infix` operator the rules are:

 - If the operator starts with `>` then it produces an `Optional.t`
 - If the operator contains `&`, it is closely tied to `Lens.t`. Either it composes lenses or the LHS should be a lens.
   + `&>` composes an optional followed by a lens returning an optional
   + `&` is `Lens.(>>)`
   + `>&` composes a lens with a prism and returns an optional
 - If the operator contains `$`, it is closely tied to `Prism.t`.
   + `$>` composes an optional followed by a prism returning an optional
   + `$` is `Prism.(>>)`
   + `>$` composes a prism with a lens and returns an optional

```ocaml
# #show_module Infix;;
module Infix :
  sig
    val ( >> ) :
      ('a, 'b) Optional.t -> ('b, 'c) Optional.t -> ('a, 'c) Optional.t
    val ( &> ) :
      ('a, 'b) Optional.t -> ('b, 'c) Lens.t -> ('a, 'c) Optional.t
    val ( $> ) :
      ('a, 'b) Optional.t -> ('b, 'c) Prism.t -> ('a, 'c) Optional.t
    val ( >& ) : ('a, 'b) Lens.t -> ('b, 'c) Prism.t -> ('a, 'c) Optional.t
    val ( >$ ) : ('a, 'b) Prism.t -> ('b, 'c) Lens.t -> ('a, 'c) Optional.t
    val ( & ) : ('a, 'b) Lens.t -> ('b, 'c) Lens.t -> ('a, 'c) Lens.t
    val ( $ ) : ('a, 'b) Prism.t -> ('b, 'c) Prism.t -> ('a, 'c) Prism.t
  end
```

### Optionals from Lenses and Prisms

You can always create an `Optional.t` from a `Prism.t` or a `Lens.t`.

```ocaml
# Optional.prism point2d;;
- : (point, point2d) Optional.t = Optics.Lens.V (<fun>, <fun>)
# Optional.lens props;;
- : (t, prop list) Optional.t = Optics.Lens.V (<fun>, <fun>)
```
