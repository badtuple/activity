# streams

Please read the `README.md` in the `go-fed/activity/vocab` package first. This
library is a convenience layer on top of the `go-fed/activity/vocab` library, so
this README builds off of that one.

This library is entirely code-generated by the
`go-fed/activity/tools/streams/gen` library and `go-fed/activity/tools/streams`
tool. Run `go generate` to refresh the library, which requires `$GOPATH/bin` to
be on your `$PATH`.

## What it does

This library provides a `Resolver`, which is simply a collection of callbacks
that clients can specify to handle specific ActivtyStream data types. The
`Resolver.Deserialize` method turns a JSON-decoded `map[string]interface{}`
into its proper type, passed to the corresponding callback.

For example, given the data:

```
{
  "@context": "https://www.w3.org/ns/activitystreams",
  "type": "Note",
  "name": "Equivalent Exchange",
  "content": "I'll give half of my life to you and you give half of yours to me!",
  "attachment": "https://example.com/attachment"
}
```

in `b []byte` one can do the following:

```
var m map[string]interface{}
if err := json.Unmarshal(b, &m); err != nil {
	return err
}
r := &Resolver {
	NoteCallback: func(n *Note) error {
		// 1) Use the Note concrete type here
		// 2) Errors are propagated transparently
	},
}
if handled, err := r.Deserialize(m); err != nil {
	// 3) Any errors from #2 can be handled, or the payload is an unknown type.
	return err
} else if !handled {
	// 4) The callback to handle the concrete type was not set.
}
```

Only set the callbacks that are interesting. There is no need to set every
callback, unless your application requires it.

## Using concrete types

The convenience layer provides easy access to properties with specific types.
However, because ActivityStreams is fundamentally built off of JSON-LD and
still permits large degree of freedom when it comes to obtaining a concrete type
for a property, the convenience API is built to give clients the freedom to
choose how best to federate.

For every type in this package (except `Resolver`), there is an equivalent type
in the `activity/vocab` package. It takes only a call to `Raw` to go from this
convenience API to the full API:

```
r := &Resolver {
	NoteCallback: func(n *Note) error {
		// Raw is available for all ActivityStream types
		vocabNote := n.Raw()
	},
}
```

To determine whether the call to `Raw` is needed, the "get" and "has" methods
use `Resolution` and `Presence` types to inform client code. The client is free
to support as many types as is feasible within the specific application.

Reusing the `Note` example above that has an `attachment`, the following is 
client code that tries to handle every possible type that `attachment` can
take. **The W3C does not require client applications to support all of these
use cases.** 

```
r := &Resolver {}
r.NoteCallback = func(n *Note) error {
	if n.LenAttachment() == 1 {
		if presence := n.HasAttachment(0); p == ConvenientPresence {
			// A new or existing Resolver can be used. This is the convenient getter.
			if resolution, err := n.ResolveAttachment(r, 0); err != nil {
				return err
			} else if resolution == RawResolutionNeeded {
				vocabNote := n.Raw()
				// Use the full API
				if vocabNote.IsAttachmentIRI(0) {
					...
				} else ...
			}
		} else if p == RawPresence {
			vocabNote := n.Raw()
			// Use the full API
			if vocabNote.IsAttachmentIRI(0) {
				...
			} else ...
		}
	}
}
```

## Serializing data

Creating a raw type and serializing it is straightforward:

```
n := &Note{}
n.AddName("I'll see you again")
n.AddContent("You don't have to be alone when I leave")
// The "type" property is automatically handled...
m, err := n.Serialize()
if err != nil {
	return err
}
// ...but "@context" is not.
m["@context"] = "https://www.w3.org/ns/activitystreams"
b, err := json.Marshal(m)
```

The only caveat is that clients must set `"@context"` manually at this time.

## What it doesn't do

Please see the same section in the `go-fed/activity/vocab` package.

## Other considerations

This library is entirely code-generated. Please see the same section in the
`go-fed/activity/vocab` package for more details.
