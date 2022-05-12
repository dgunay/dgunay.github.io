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
potentially being another `SelectorExpr`. The `ProcessTypeChain` bit recursively
traverses the AST until it runs into a terminating `Ident`. We store the Ident
of the interface and other relevant information for later.
