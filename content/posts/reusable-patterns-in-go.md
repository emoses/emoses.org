+++
title = 'Go Generics: Use Structs for Generic Arguments Lists'
date = 2024-04-30
categories = ['blog']
tags = ['go', 'programming', 'patterns']
description = """ I wanted to build reusable code for a pattern in Go,
I had to fight the type system a bit but I won in the end
We can pack argument lists into structs to make the pattern generic over different sets of arguments to functions"""
+++

{{< tldr >}}
* I wanted to build reusable code for a pattern in Go
* I had to fight the type system a bit but I won in the end
* We can pack argument lists into structs to make the pattern generic over different sets of arguments to functions
* Skip to the [end of the post](#solution) for spoilers
{{< /tldr >}}

I have a preference for functional-style programming. I tend to model algorithms in my head as data and transforms of
that data (versus, say, objects with behaviors).  When I see a pattern repeat itself in my code my first instinct is to
reach for a higher-order function to abstract that pattern.

However, my day job is coding in Go.  I have mixed feelings about the language and I'm not gonna start a Go flame war
here, but I don't think it's controversial to say that Go does not lend itself to functional patterns.

## The pattern

I implemented a new, faster algorithm with to make some security calculations for our system; the details are
unimportant, but if the result differed between the old code and my new algorithm, it would represent a serious problem
for our customers.  To make sure my new algorithm worked correctly, my plan was to calculate it the new way *and* the
old way, compare the two values, and return the old value, logging an error if there were differences.  I added two
feature flags: `UseNewAlgorithmDark` enabled the new calculation with comparison and logging, and `UseNewAlgorithmOnly`
stopped using the old calculation and just returned the result of the new one.

There are actually a few different places in the code that would be using the new code, and they did different
calculations with it.  So I now had multiple places in code that looked something like this:

```golang
func (c *Controller) GetImportantData(
    ctx context.Context,
    tenant *models.Tenant,
    userId string,
    filter *ImportantDataFilter,
) (*ImportantData, error) {
    useNewWay := c.HasFeature(ctx, tenant, features.UseNewAlgorithmDark)

    if useNewWay {
        newWayLaunched := c.HasFeature(ctx, tenant, features.UseNewAlgorithmOnly)

        newWayResult, err := c.getImportantDataNewWay(ctx, tenant, userId, filter)
        if err != nil {
            c.Log().Error("Error getting important data new way, falling back")
            return c.getImportantDataOldWay(ctx, tenant, userId, filter)
        }

        // If the Launched flag is on, just return the new result
        if newWayLaunched {
            return newWayResult, nil
        }

        // otherwise get the result the old way...
        oldWayResult, err := c.getImportantDataOldWay(ctx, tenant, userId, filter)
        if err != nil {
            return nil, err
        }

        // ...and compare it against the result from the new way.
        if !dataIsEqual(newWayResult, oldWayResult) {
            // If there's a mismatch, log the error, but continue
            c.Log().Error("Mismatch between old way and new way for ImportantData",
                zap.String("userId", user_id),
                zap.Any("filter", filter),
            )
        }

        // and always return the old result.
        return oldWayResult, nil
    }

    return c.getImportantDataOldWay(ctx, tenant, userId, filter)
}
```

Importantly, not all the controller functions that used the launch feature took the same sort of arguments.  There was
another that was *almost* identical, but the signature looked like this:

```golang
func (c *Controller) GetDifferentData(ctx context.Context, tenant *models.Tenant,
    relatedThingId string) (*DifferentData, error) { ... }
```

## The abstraction

Beyond wanting to keep my code DRY, I wanted to codify the Dark Launch pattern so other folks on our team could reuse it.
So I started with something like this:

```golang
type DarkLaunch[T any] struct {
    OperationName string
    DarkFlag features.FeatureFlag
    LaunchFlag features.FeatureFlag
    NewWay func(context.Context, *models.Tenant) (T, error)
    OldWay func(context.Context, *models.Tenant) (T, error)
    Cmp func(T, T) bool
    Logger *zap.Logger
    //features.FeatureManager is an existing interface that has HasFeature
    FeatureManager features.FeatureManager
}

func (dl DarkLaunch[T]) Execute(ctx context.Context, tenant *models.Tenant) (T, error) {
    useNewWay := dl.FeatureManager.HasFeature(ctx, tenant, dl.DarkFlag)

    if useNewWay {
        newWayLaunched := dl.FeatureManager.HasFeature(ctx, tenant, dl.LaunchFlag)

        newWayResult, err := dl.NewWay(ctx, tenant)
        if err != nil {
            dl.Logger.Error(fmt.Sprintf("Error getting data new way for %s, falling back", dl.OperationName))
            return dl.OldWay(ctx, tenant)
        }

        if newWayLaunched {
            return newWayResult, nil
        }

        oldWayResult, err := dl.OldWay(ctx, tenant)
        if err != nil {
            var zero T
            return zero, err
        }

        if !dl.Cmp(newWayResult, oldWayResult) {
            dl.Logger.Error(fmt.Sprintf("Mismatch between old way and new way for %s", dl.OperationName))
                // Hmmm, I'm missing context data here for logging here, aren't I 🤔
            )
        }

        return oldWayResult, nil
    }

    return c.OldWay(ctx, tenant)
}
```

And it could be consumed like so:

```golang
func (c *Controller) GetImportantData(
    ctx context.Context,
    tenant *models.Tenant,
    userId string,
    filter *ImportantDataFilter,
) (*ImportantData, error) {
    dl := DarkLaunch[*ImportantData]{
        OperationName: "GetImportantData",
        DarkFlag: features.UseNewAlgorithmDark
        LaunchFlag: features.UseNewAlgorithmOnly
        NewWay: func(ctx context.Context, tenant *models.Tenant) (*ImportantData, error) {
            // close over extra arguments here
            return c.getImportantDataNewWay(ctx, tenant, userId, filter)
        }
        OldWay: func(ctx context.Context, tenant *models.Tenant) (*ImportantData, error) {
            // close over extra arguments here
            return c.getImportantDataOldWay(ctx, tenant, userId, filter)
        },
        Cmp: dataIsEqual,
        Logger: c.Log(),
        FeatureManager: c.FeatureManager
    }
    return dl.Execute(ctx, tenant)
}
```

This worked, but it was inelegant.  For one thing, the only way to consume it is to make a fresh closure
over the arguments each time you want to call it, which means you can't just define a `DarkLaunch` somewhere and
reference it.  You also need to pass in related machinery (the logger and the feature manager).  Finally, the generic
`Execute` method doesn't have access to the arguments of the `NewWay`/`OldWay` functions, so it can't log them if there
are errors; we'd probably have to insert extra logging in the closures to log properly.

### Go issues, and other languages' solutions

The problem is that DarkLaunch isn't generic over the arguments of the NewWay/OldWay functions.  In more dynamic languages,
you'd be able to just `apply` a function to an arbitrary set of arguments, for instance in Clojure:

```clojure
(defn dark-launch [{:keys [get-feature dark-flag launch-flag old-way new-way cmp]}
                    tenant & args]
  (if-not (get-feature dark-flag)
    (apply old-way tenant args) ;; This is the secret sauce
    (let [launched (get-feature launch-flag)
          new-way-result (apply new-way tenant args)] ;; Here it is again
      (if launched
        new-way-result
        (let [old-way-result (apply old-way tenant args)] ;; And here
          (when-not (cmp new-way-result old-way-result)
                  (errorf "Mismatch, args: %s" args)))
          old-way-result)))))

(def get-important-data-dl
   {
     :get-feature feature-getter
     :dark-flag :features/use-new-algorithm-dark
     :launch-flag :features/use-new-algorithm-only
     :old-way get-important-data-old-way
     :new-way get-important-data-new-way
     :cmp data-matches
   })

;; It's tempting to fancier with macros and have something like defdarklaunch
;; but this is a simple port of the Go code and macros would be overcomplicated here anyway.
(defn get-important-data [tenant userId filter]
   (dark-launch get-important-data-dl tenant userId filter))
```

A language with a sophisticated type system can express the same idea but with strong typing.  I'm sure it can be done
in Haskell [^1], but I know enough TypeScript better, so taking advantage of [variadic tuple
types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-0.html):

[^1]: I'm not a Haskeller, suggestions welcome.  Contact info in sidebar.

```typescript
interface DarkLaunch<T, A extends any[]> {
  getFeature: (f: Feature) => boolean;
  darkFeature: Feature;
  launchFeature: Feature;
  oldWay: (ctx: Context, tenant: Tenant, ...args: [...A]) => T | undefined;
  newWay: (ctx: Context, tenant: Tenant, ...args: [...A]) => T | undefined;
  cmp: (a: T, b: T) => boolean;
}
```

The TypeScript compiler will enforce that `oldWay` and `newWay` have the same number and type of arguments, which is
what we want.  And that gives us a clue to how we can solve this in Go.

## A Go Solution {#solution}

We don't have a anything like Variadic Tuple Types [^2] in Go, but we can express the idea of a set of types
bundled together: it's a simple struct. This requires re-writing the `oldWay`/`newWay` functions slightly to take their
arguments as a limited-use struct, but that's not really a big deal.

[^2]: That's fun to say

The other improvement I made was to bundle the "auxiliary" members of the struct like the logger and the FeatureManager
that are only used to actually execute the code into an `Executor` interface.  `Controller` already satisfied
`Executor`, so now we could bundle the whole thing into its own package.  Here's the final code:

```golang
package dark_launch

type Executor interface {
	Log() *zap.Logger
	Features() features.FeatureManager
}

type DarkLaunch[T any, A any] struct {
	OperationName string
	DarkFlag      features.FeatureFlag
	LaunchFlag    features.FeatureFlag
	NewWay        func(context.Context, *models.Tenant, A) (T, error)
	OldWay        func(context.Context, *models.Tenant, A) (T, error)
	Cmp           func(T, T) bool
}

func (dl DarkLaunch[T, A]) Execute(
    ctx context.Context,
    x Executor,
    tenant *models.Tenant,
    args A,
) (T, error) {
    useNewWay := x.Features().HasFeature(ctx, tenant, dl.DarkFlag)

    if useNewWay {
        newWayLaunched := x.Features().HasFeature(ctx, tenant, dl.LaunchFlag)

        newWayResult, err := dl.NewWay(ctx, tenant, args)
        if err != nil {
            x.Log().Error(fmt.Sprintf("Error getting data new way for %s, falling back", dl.OperationName))
            return dl.OldWay(ctx, tenant, args)
        }

        if newWayLaunched {
            return newWayResult, nil
        }

        oldWayResult, err := dl.OldWay(ctx, tenant, args)
        if err != nil {
            var zero T
            return zero, err
        }

        if !dl.Cmp(newWayResult, oldWayResult) {
            x.Log().Error(fmt.Sprintf("Mismatch between old way and new way for ImportantData"
                zap.String("tenantId", tenant.Id),
                zap.Any("args", args),
            )
        }

        return oldWayResult, nil
    }

    return dl.oldWay(ctx, tenant, args)
}

// ----- elsewhere -----

type importantDataArgs struct {
    userId string
    filter *ImportantDataFilter
}

func (c *Controller) GetImportantData(
    ctx context.Context,
    tenant *models.Tenant,
    userId string,
    filter *ImportantDataFilter,
) (*ImportantData, error) {
    // This could be defined outside this function without much extra work, but it would
    // only be a teensy efficiency gain and wouldn't add much readability.  I'd do it if
    // the same args struct and getImporatantData* were re-used in a few different places.
    dl := DarkLaunch[*ImportantData, importantDataArgs]{
        OperationName: "GetImportantData",
        DarkFlag: features.UseNewAlgorithmDark,
        LaunchFlag: features.UseNewAlgorithmOnly,
        NewWay: c.getImportantDataNewWay,
        OldWay: c.getImportantDataOldWay,
        Cmp: dataIsEqual,
    }

    return dl.Execute(ctx, c, tenant, importantDataArgs{userId: userId, filter: filter})
}

// getImporantDataNewWay and getImportantDataOldWay must be re-written to take in the args struct
func (c *Controller) getImportantDataNewWay(
    ctx context.Context,
    tenant *models.Tenant,
    args importantDataArgs,
) (*ImportantData, error) { ... }


// GetDifferentData works the same, but since it only takes a single argument beyond the common args,
// we don't even need a limited-use struct
func (c *Controller) GetDifferentData(ctx context.Context, tenant *models.Tenant,
    relatedThingId string) (*DifferentData, error) {
    dl := DarkLaunch[*DifferentData, string]{
        OperationName: "GetDifferentData",
        DarkFlag: features.UseNewAlgorithmDark,
        LaunchFlag: features.UseNewAlgorithmOnly,
        NewWay: c.getDifferentDataNewWay,
        OldWay: c.getDifferentDataOldWay,
        Cmp: dataIsEqual,
    }

    return dl.Execute(ctx, c, tenant, relatedThingId)
}
```

Much prettier, huh? Go's type system isn't powerful enough to have a type argument that expresses an arbitrary-length
but finite list of types *as parameters to a function*; however we can use a product type (aka a struct) to represent
the finite list [^3].

[^3]: My category theory knowledge comes almost exclusively from reading intro-to-Haskell blog posts linked from the
    orange site.  Please be pedantic at me if I got my terminology wrong.
