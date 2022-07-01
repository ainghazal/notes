# Dependency Injection In Go

_A short glossary of techniques._

These are some notes I took about [Hands-On Dependency Injection in Go](https://www.goodreads.com/book/show/43268781-hands-on-dependency-injection-in-go)

The author of the book explains 6 different techniques of Dependency Injection.

As many things in life, all is a trade-off: complexity, decoupling, test-induced damage. YMMV.

## 1. Monkey-Patching

Good old "replace a global variable" during testing.

Very simple, but need to be careful with data races.

```Go
var foo // the global

func TestFoo(t *testing.T) {
        defer func(original) {
                foo = original
        }(foo)
        foo = bar
}
```

## 2. Constructor Injection

We _inject_ the dependency via the constructor. So this:

```Go
func NewFoo(t *Thing) (*Bar, error) {
    // ...
    return &Bar{
        thing: t
    }, nil
```

becomes this:


```Go
func NewFoo(t Thinger) *Bar {
    return &Bar{
        thing: t
    }
```

Where the interface has been factored out from the struct, for the methods that are used only (**SRP**).

This also satisfies the _"accept an interface, return a struct"_ mantra.


## 3. Method injection

Quite trivial, why is this even a thing? We pass dependencies as _arguments_ to the call.

Works well with pure top-level functions.


```Go
fmt.Fprintf(os.Stdout, "hello complexity")

// is really

func Fprintf(w io.Writer, a ...interface{})
```

Passing a `context` object is a form of Dependency Injection.

_(I do think this section of the book is not very well structured.)_


## 4. Config injection

A *specific* implementation of _Method_ and _Parameter_ injection. We combine multiple Dependency and System-Level configs, and merge them into a *Config interface*.

```Go
func NewLongConstructor(foo, bar, too, many, dull, things...) *Struct {}

// becomes...

func NewConfigConstructor(cfg Config, foo) *Struct{}
```

* We hide the "concerns" (e.g., the clutter if we have too many arguments)
* Config is an interface.


## 5. JIT injection

* Just inject what we need by assigning to the private variable.
* All uses in the code replace refs to the field by a getter that will do an instantiation.
* Better to replace `nil` by sensible defaults (in the getter).
* It is best used when we need it just for testing.

```Go
type Thing struct {
  data DataSourceJIT
}

func (t *Thing) DoStuff(id int) (Foo, error) {
  return m.getData().Load(id)
}

func (t Thing) getData() DataSourceJIT {
  if t.data == nil {
    t.data = NewDataSourceJIT()
  }
  return t.data
}
```

This method perhaps has a too-heavy object-orientation smell, but can come handy if we're injecting only for testability.

## 6. Off-the-shelf Injectors

Leave all the magic to a well-maintained (and hopefully thought) library. The book mentions several (uber, etc.) but goes with [Google's Wire](https://github.com/google/wire).

* Two basic concepts: _providers_ and _injectors_.
  * The `provider` returns an instance of a dependency.
  * The `injector` is a function that `Wire` uses for code generation.
* No extra deps, uses Go's code generation.


The example in the tutorial uses a chain of objects that need to be initialized. In the simple case, they depend in each other linearly.


```Go
// wire.go

// the injector
func InitializeEvent() Event {
    wire.Build(NewEvent, NewGreeter, NewMessage)
    return Event{}
}
```

The `injector` _provides information about which providers to use_ to construct our leaf object. The providers in this case are simple types that need one other to get initialized.

A build constrain is used to avoid including the injector in the compiled binary.

The entrypoint in main is then written like:

```Go
func main() {
    e := InitializeEvent()
    e.Start()
}
```

...that generates this code:

```Go
func main() {
    message := NewMessage()
    greeter := NewGreeter(message)
    event := NewEvent(greeter)

    event.Start()
}
```

* Question: _how's this implemented?_ Does it resolve the dependency DAG?
* Question: is this useful? _Too much_ magic?






