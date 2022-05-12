---
layout: post
title: "Building a Go static analysis tool"
date: 2022-05-11 23:08:02 -0700
categories: golang
---

Go has a lot of very good linters available, and also has great linter
aggregators like [golangci-lint](https://golangci-lint.run/) that let you
run a barrage of linters on your code. I consider it an indispensable part of
a Go developer's toolkit because of how spartan the language is.

I recently wrote [ifacecapture][repo-link] to detect a very specific and
serious problem that we ran into a couple times due to a new interface I rolled
out.

[repo-link]: https://github.com/dgunay/ifacecapture

For background, we use GORM to manage our database schema and queries. Like any
good ORM, GORM supports transactions:

```go
db.Transaction(func(tx *gorm.DB) error {
  tx.Create(&User{Name: "foo"})

  if tx.Error != nil {
    return tx.Error // errors roll back any writes in this callback
  }

  // return nil will commit the whole transaction
  return nil
})
```

Since we use Uncle Bob's Clean Architecture, we have our database queries behind
an interface. I added a method to our interface to allow us to run transactions
like so:

```go
type DatabaseRepo interface {
  Transaction(func(tx DatabaseRepo) error) error
  // ...
}
```

Worked great, we have access to transactions now. However, there's one footgun
with the way this transaction works, stemming from Go's permissive closure
capturing behavior. Here's an example:

```go
func DoQuery(db *SomeImplementationOfDatabaseRepo) error {
  return db.Transaction(func(tx DatabaseRepo) error {
    tx.Create(&User{Name: "foo"})
    db.Create(&User{Name: "bar"}) // Oops
    return nil
  })
}
```

It's very easy for someone to accidentally use the outer-scope variable `db`,
when they should be using the inner-scope variable `tx`. This could result in
important queries making writes that are not meant to be committed.

If you know my philosophy on programming, you know I deeply doubt the ability
of humans to manage the amount of "just be careful" kinds of errors one can make
as a programmer. This maed this a good candidate for a very tightly-scoped
linter, so I set out to write one.

## Creating the linter

Go's tooling and standard library are excellent, so naturally there is a super
convenient framework to [write analysis passes][analysis], and also great sets of primitives
to [navigate parsed and typechecked Go source code][ast].

[analysis]: https://pkg.go.dev/golang.org/x/tools/go/analysis
[ast]: https://pkg.go.dev/go/ast

First I had to get an idea of what Go's Abstract Syntax Tree (AST) looks like.
I wrote an example program and fed it to [GoAst Viewer][ast-viewer]:

[ast-viewer]: https://yuroyoro.github.io/goast-viewer/index.html

{% raw %}

```go
// Example program that erroneously captures the outer variable when it likely
// intends to use the parameter interface.

package main

type MyInterface interface {
	Do()
}

type MyImpl struct{}

var _ MyInterface = (*MyImpl)(nil)

func (m *MyImpl) Do() {}

func doThing(callback func(tx MyInterface)) {
	myImpl := MyImpl{}
	callback(&myImpl)
}

type HasMyImpl struct {
	A MyImpl
}

func (h HasMyImpl) GetMyImpl() *MyImpl {
	return &h.A
}

func main() {
	outer := MyImpl{}
	outer2 := HasMyImpl{A: MyImpl{}}
	outer3 := struct{ B HasMyImpl }{B: HasMyImpl{A: MyImpl{}}}
	outerArr := [2]MyImpl{{}, {}}
	doThing(func(inner MyInterface) {
		outer.Do()              // want "captured variable outer implements interface MyInterface"
		outer2.A.Do()           // want "captured variable outer2.A implements interface MyInterface"
		outer3.B.A.Do()         // want "captured variable outer3.B.A implements interface MyInterface"
		outerArr[0].Do()        // We don't flag this yet because it is a lot of extra work
		outer2.GetMyImpl().Do() // We don't flag this yet because it becomes much harder to analyze where the receiver is coming from
		inner.Do()
	})
}
```

{% endraw %}

This ended up being a usable test case for the linter thanks to the `// want`
directives that the analysis testing package can automatically parse.

There's a lot of stuff in there but the most important things we want to
navigate are:

- `doThing(func(inner MyInterface) {` - since we need to identify calls to functions
  that accept some callback as a parameter, which itself has an interface in its parameters.
- `outer.Do()` - since we need to identify calls on some receiver that implements
  said interface.

### Identifying function calls we want

The first part was easy - `doThing` is a `ast.CallExpr`, which we can filter
on. We can easily tell it has arguments with its `.Args` member. And then
we can filter `.Args` on elements that are castable to `*ast.FuncLit`, to
identify function literals.

### Traversing callback parameter lists

Once we have a function literal, we can traverse its param list:

{% raw %}

```go
func run(pass *analysis.Pass) (any, error) {
	// omitted boilerplate

	inspect := func(node ast.Node) bool {
		// omitted boilerplate
    // callback := /* some *ast.Funclit */

		// Step 4: gather all interface types in the param list
    var paramInterfaceTypes []ParamType
    for _, param := range callback.Type.Params.List {
      if param.Type != nil {
        vars := param.Names

        paramType := pass.TypesInfo.TypeOf(param.Type)
        if _, ok := paramType.Underlying().(*types.Interface); ok {
          // May have to go forward all the way to get the ident
          chain := NewTypeChain()
          if err := chain.ProcessTypeChain(param.Type); err != nil {
            logger.Errorf("Failed to process type chain: %s", err)
            continue
          }

          paramInterfaceTypes = append(paramInterfaceTypes, ParamType{
            InterfaceIdent: chain.Last(),
            InterfaceType:  paramType.Underlying().(*types.Interface),
            Vars:           vars,
          })
        }
      }
    }
  }
}
```

{% endraw %}

The bit about `ProcessTypeChain` is the first part where we need to do actual
AST traversal. In Go, there are many bits of the syntax which are chained
together using `.`, which in the AST is referred to as a `SelectorExpr`.
In the case of something like `mypkg.MyInterface`, the AST looks like:

![](/assets/SelectorExpr-example.svg)

Simple enough, but the wrinkly part comes from the right leaf of that tree
potentially being a chain of `SelectorExpr`s. The `ProcessTypeChain` bit recursively
traverses the AST until it runs into a terminating `Ident`. We store the Ident
of the interface and other relevant information for later.

### Finding captured variables we care about

We then need to find all places where a function is called with some receiver.
This is harder and I ended up cutting some corners. The code:

{% raw %}

```go
// Step 5: gather all captured variables in the body
// Get all CallExprs with receivers
capturedCalls := []CallViaReceiver{}
ast.Inspect(callback.Body, func(node ast.Node) bool {
  switch node.(type) {
  case *ast.CallExpr:
    capturedCall := NewCallViaReceiver(pass.TypesInfo)

    expr := node.(*ast.CallExpr).Fun
    if selExpr, ok := expr.(*ast.SelectorExpr); ok {
      err := capturedCall.ProcessSelExpr(selExpr)
      if err == nil {
        capturedCalls = append(capturedCalls, capturedCall)
      } else {
        logger.Error(err)
      }
    }
  }
  return true
})
```

{% endraw %}

The `SelectorExpr` traversal is encapsulated in my `CallViaReceiver` struct.
It can handle the simple case `a.b.c.Foo()`, but as you may have surmised from
the test code from earlier, it doesn't handle receivers in arrays and
receivers from the return value of other functions. Finding the type of the
receiver was not hard, but I currently don't know of a convenient way of
tracing the receiver's owner and where they are scoped. Maybe this is easy and I
just don't know how to do it yet - that turned out to be the case for the next
problem.

### Figure out if the captured receivers implement any of our interface types

This part took me a while. At first I tried to access the type information
through the AST, since some of the structs have pointers to `ast.Object`, which
has type information in the same package. However, some objects pointed to
things in other packages, and in that case the type information would usually
be `nil` - super unhelpful.

I eventually reached out on the Gophers Slack in the #tools channel and someone
coaxed me into finding out that the `*analysis.Pass` that our run function
receives already has the type information for all of the code. Since we
have access to the type info and gathered it in the last two steps, all we
end up having to do (after removing the looping and configuration boilerplate):

{% raw %}

```go
// Don't check if the receiver is one of the function params
// (code that skips receivers that are function params)

ifaceType := paramType.InterfaceType
logger.Debugf("Checking if %s implements %s", capturedType, paramType.InterfaceIdent.Name)
if types.Implements(capturedType, ifaceType) {
  Report(pass, &capturedCall, paramType)
} else if types.Implements(types.NewPointer(capturedType), ifaceType) {
  // FIXME: it is unclear to me why sometimes it is necessary
  // to convert the type to a pointer before checking if it
  // implements the interface. Haven't yet reproduced the bug.
  Report(pass, &capturedCall, paramType)
}
```

{% endraw %}

## Testing

I wrote a variety of tests using example code, and then used the [`analysistest`][analysistest]
package to run `ifacecapture` and check that the `// want` directives happened
(and that no false positives happened).

[analysistest]: https://pkg.go.dev/golang.org/x/tools/go/analysis/analysistest

## Remarks

Overall I found the process surprisingly accessible. Go's obnoxious simplicity
works to its advantage here greatly. It took me a weekend to get this working
in an okay-ish state; I'd highly recommend that if you maintain a large Go
codebase, you take some time to learn about the `ast` package and writing
analyzers. You could write bespoke tooling to enforce your organization's
code patterns and catch usage mistakes that are very specific to your
application's design.
