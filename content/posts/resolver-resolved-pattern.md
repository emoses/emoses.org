+++
title = 'Resolver/Resolved Batching Pattern in Go'
date = 2024-03-26
categories = ['blog']
tags = ['go', 'patterns', 'programming']
draft = true
+++

I made up a neat little pattern in Go the other day I'm calling Resolver/Resolved.  I'm sure I'm not the first person to
invent this, and it may already a name, so please let me know if you know of one.  I'll show you the pattern first and
talk about the motivation later.

## The Pattern

```go
// You start with a Resolver, and incrementally feed it data to be looked up
type Resolver interface {
    // Collect collects some piece of data and maybe extra information about it that you need to do resolution
    Collect(someId string, someData string)
    // Maybe you also need some global contextual data to do the resolve
    AddContextualData(data SomeContext)
    // When you're done, the resolve can send the bulk query to the executor to perform the lookup
    Execute(context.Context, Executor) (Resolved, error)
}

// Resolved is what you get back after you Execute, and it lets you access the resolved data
type Resolved interface {
    // Resolve returns the result.  If the result doesn't have that id, either because it wasn't looked up successfully
    // during execution or because you never [Collect]ed it, it will return ("", false)
    Resolve(id string) (string, bool)
}

// Executor is capable of doing the db lookup/cache lookup/service request/http request that actually gets the data you
// need to get
type Exectuor interface {
    DoTheThing(ctx context.Context, idsToResolve map[string][]string, contextualData SomeContext) ([]ResolveResult,
    error)
}

```

So far, so boring.  You've got a `Resolver` that batches up query data, an `Executor` that queries it, and a `Resolve`
that lets you access the result.  What's neat about this is that `Resolver` and `Resolved` can be implemented by the
same struct:

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

    // Ooh look I'm just returning this struct as a Resolver!
	return r, nil
}

func (r *idResolver) Resolve(id string) (string, bool) {
	res, ok := r.resolved[id]
	return res, ok
}
```

## Ok but why?

In the case in question, I was building a system to bulk-lookup names in my system by ID.  I built something very
similar to the above, but `IdResolver` was the only struct.  I added a boolean `hasExecuted` to `IdResolver` so that if
you called `Resolve` before you had called `Execute`, it would check that flag and if it hadn't been `Executed` yet it
would panic; similarlly if you tried to `Collect` another ID after you'd called `Execute` it would panic.

That made the whole thing messier and
