## TLP in Scala

### Concepts ###

- Dependent Type
- Abstract Type
- `type` keyword

- Phantom Type, used as type constraints but never instantiated.

- Implicit Parameter/Conversion

``` scala
trait Printer[T] {
  def print(t: T): String
}

implicit val sp: Printer[Int] = new Printer[Int] {
  def print(i :Int) = i.toString
}

def foo[T](t: T)(implicit p: Printer[T]) = p.print(t)
```

- Type Class

``` scala
trait CanFoo[A] {
  def foos(x: A): String
}

case class Wrapper(wrapped: String)

object WrapperCanFoo extends CanFoo[Wrapper] {
  def foos(x: Wrapper) = x.wrapped
}
```
Idea: Provide evidence(`WrapperCanFoo`) that a class(`Wrapper`) satisfies an interface(`CanFoo`).

add implicit to turn into:

``` scala
implicit object WrapperCanFoo extends CanFoo[Wrapper] {
  def foos(x: Wrapper) = x.wrapped
}
def foo[A](thing: A)(implicit evidence: CanFoo[A]) = evidence.foos(thing)
foo(Wrapper("hi"))
```

- `implicitly` and `=:=`

- Aux Pattern


### Reflection ###

#### Symbol ####

TypeSymbol: type, class, trait declarations, type parameters.
  - ClassSymbol
TermSymbol: val, var, def, object declarations, packages, value parameters.
  - MethodSymbol: def
  - ModuleSymbol: object declaration

转换到更具化的符号。

`asClass`,
`asMethod`

``` scala
typeOf[String].member(TermName("length")).asMethod
```

#### Type ####

``` scala

universe.typeOf
universe.weakTypeOf

```

- check subtyping of two types: `<:<`, `weak_<:<`
- check equality of two types: `=:=`(not `==`)
- query for certain members or inner types: `typeOf[T].members`, `typeOf[T].declarations`


