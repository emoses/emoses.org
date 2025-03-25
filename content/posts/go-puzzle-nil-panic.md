+++
title = 'A Go Puzzle: How can this panic?'
date = 2025-03-21
categories = ['blog']
tags = ['go', 'golang']
draft = false
+++

Here's a puzzle for Go developers.  How can the following code possibly panic with a nil pointer dereference?

```golang
var someThing *SomeStruct

// various things happen, including possibly assigning to someThing

if someThing != nil && someThing.Body != nil {
    fmt.Printf("%s", something.Body)
}
```

Think about it for a minute.  I'll wait.

OK are you done? I don't want to spoil it for you.

## The Answer

Here's the definition of `SomeStruct`,

```golang
type SomeStruct struct {
    *http.Response
    OtherField string
}
```

Now we can see the problem: we checked if `someThing` was nil but we implicitly referenced a field on the embedded
`http.Response` without checking if `Response` was nil.  If `Response` is nil is the code will panic.

## Think really hard before you embed a pointer

I came across this struct in a API client library that I won't name.  I think it was probably a bad design idea. The
struct represented the response from that API.  So it looked something like this:

```golang
type APIResponse struct {
    *http.Response
    // non-exported fields
}

func (*APIResponse) NextPage() (*APIResponse, error) {...}
func (*APIResponse) HasNextPage() bool {...}
```

So: it wraps the actual API response, plus some extra methods for dealing with pagination. I can see why
the author thought embedding the response was a good idea: it lets the consumer treat it like a normal http response
with some extra features.

However, it becomes a problem when you can get a non-nil `APIResponse` with a nil `Response`. If you're going to embed a
pointer in a Go struct, you should do you best to make sure you never leave that pointer nil.

All in all I think this API would have been better served by just making the http response a normal struct field, and
possibly copying important fields like Status into the struct[^1]:

[^1]: Of course, if the response is nil, I suppose you leave `Status` as the zero value and let consumers figure out
    what do do with http response code 0.

```golang
type APIResponse struct {
    Response *http.Response
    // And if it's really important to you that people type APIResponse.Status
    // instead of APIResponse.Response.Status:
    Status int
    // non-exported fields
}
```
