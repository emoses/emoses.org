+++
title = 'Representing State as interfaces in Go'
date = 2024-03-26
categories = ['blog']
tags = ['go', 'patterns', 'programming']
+++

I made up a neat little pattern in Go the other day. It's a way to represent a state change in a system by exposing
different APIs for different states, while only holding state in a single underlying struct. I'm sure I'm not the first
person to invent this, and it may already a name, so please let me know if you know of one *[Update:
[apg](https://lobste.rs/~apg) on Lobsters [pointed out the
name](https://lobste.rs/s/tzgizl/representing_state_as_interfaces_go#c_cvlm2d) "typestate", which I like]*.  I'm going
to show an instance of the pattern first and the motivation after.

## The Pattern

```go
// You start with a Resolver, and incrementally feed it data to be looked up
type Resolver interface {
    // Collect collects some piece of data and maybe extra information about it that you
    // need to do resolution
    Collect(someId string, someData string)
    // Maybe you also need some global contextual data to do the resolve
    AddContextualData(data SomeContext)
    // When you're done, the resolve can send the bulk query to the executor to perform
    // the lookup that you want to do
    Execute(context.Context, Executor) (Resolved, error)
}

// Resolved is what you get back after you Execute, and it lets you access the resolved data
type Resolved interface {
    // Resolve returns the result. If the result doesn't have that id, either because it
    // wasn't looked up successfully during execution or because you never [Collect]ed it,
    // it will return ("", false)
    Resolve(id string) (string, bool)
}

// Executor is capable of doing the db lookup/cache lookup/service request/http request
// that actually gets the data you need to get
type Executor interface {
    DoTheThing(
        ctx context.Context,
        idsToResolve map[string][]string,
        contextualData SomeContext,
    ) ([]ResolveResult, error)
}

```

So far, so boring.  You've got a `Resolver` that batches up query data, an `Executor` that, well, executes the query,
 and a `Resolve` that lets you access the result.  What's neat about this is that `Resolver` and `Resolved` can be
 implemented by the same struct:

```go
type idResolver struct {
	collected map[string][]string
	contextData SomeContext
	resolved map[string]string
}

func (r *idResolver) Collect(someId string, someCategory string) {
	r.collected[someCategory] = append(r.collected[someCategory], someId)
}

func (r *idResolver) AddContextualData(data SomeContext) {
	r.contextData = data
}

func (r *idResolver) Execute(ctx context.Context, executor Executor) (Resolved, error) {
	result, err := executor.DoTheThing(ctx, r.collected, r.contextData)
	if err != nil {
		return nil, err
	}

	r.resolved = transformResult(result)

	/***
	 * 🪄 THE MAGIC 🪄
	 * Ooh look I'm just returning this struct as a Resolved!
	 */
	return r, nil
}

func (r *idResolver) Resolve(id string) (string, bool) {
	res, ok := r.resolved[id]
	return res, ok
}
```

## Ok but why?

I was building a system to bulk-lookup names in my system by ID.  But if you called `Resolve` before it had executed,
you'd have an invalid result.  And if you `Collected` after you executed, you'd never look up the id.  So I added a
boolean `hasExecuted` to `IdResolver` so that if you called `Resolve` before you had called `Execute` or `Collect`
after, it would check that flag and panic.

```go
func (r *StatefulIdResolver) Collect(someId string, someCategory string) {
    if r.hasExecuted {
        panic("Collect called after Execute")
    }
	r.collected[someCategory] = append(r.collected[someCategory], someId)
}

// Execute executes and changes state but doesn't return anything new
func (r *StatefulIdResolver) Execute(ctx context.Context, executor Executor) error {
    if r.hasExecuted {
        panic("Execute called twice")
    }
    ...
    r.hasExecuted = true
    ...
}

func (r *StatefulIdResolver) Resolve(id string) (string, bool) {
    if !r.hasExecuted {
        panic("Resolve called before execute")
    }
	res, ok := r.resolved[id]
	return res, ok
}
```

This is kind of a mess, harder to read and maintain, and much easier to mess up the logic in.

## When to use it

Of course the `Resolver` and the `Resolved` *could* be represented by different structs with their own methods.  But in
this case it felt like we were really working with a single object:  it collects data, operates on it, and manages the
result.

The crux of the matter is that the object has different operations that are valid in different states, and Go interfaces
are a perfect way to expose that.

## Update: Is this just a `Builder`?

A few folks on the internet have pointed out that this is very similar to the Builder pattern that you see pretty
often in Java/C# (and to a lesser extent in Go), and especially a multiphase [Step Builder](https://www.svlada.com/step-builder-pattern/).

I don't think there's a fundamental difference, but Builders that I'm familar with generally have a state-change/execute
  step (`Build()`) that produces an immutable object for use somewhere else, rather than doing any sort of effectful
  execution.  I think this pattern is both more general (you could certainly use it as a Builder) and has a different
  purpose.

{{< discussion >}}
* [Lobsters](https://lobste.rs/s/tzgizl/representing_state_as_interfaces_go)
* [Hacker News](https://news.ycombinator.com/item?id=39848164)
{{< /discussion >}}




<!--  LocalWords:  someData AddContextualData SomeContext
 -->
