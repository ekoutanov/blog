OData evaluation
===
Open Data Protocol (OData) 4.0 is a mature-ish open protocol initially championed by Microsoft in 2007 and is now an OASIS standard. It can do lots of things well. No point promoting stuff that has been touted elsewhere; this post discusses some things that it doesn't do so well, and how it could be fixed.

#1. Navigation is not explicit
Assume the following uber-simple composition: A `Dog` class has an optional `Tail` class. Suppose a tail may potentially encompass a lot of data such that returning a tail by default is not desirable.

In OData:  

	GET serviceRoot/Dogs('1234')

Response:  

	{
	  "@odata.context": "serviceRoot/$metadata#Domain.Dog",  
	  "@odata.id": "serviceRoot/Dogs('1234')",  
	  "id": 1234,  
	  "name": "Bobby"  
	}


To get both the dog and the tail:

	GET serviceRoot/Dogs('1234')?$expand=tail

Response:

	{
	  "@odata.context": "serviceRoot/$metadata#Domain.Dog",
	  "@odata.id": "serviceRoot/Dogs('1234')",
	  "id": 1234,
	  "name": "Bobby",
	  "tail": {
	    "@odata.context": "serviceRoot/$metadata#Domain.Tail",
	    "length": 20,
	    "description": "Some long description"
	  }
	}

Going back to the first snippet, the requester has no way of determining whether this particular dog has a tail. The tail property is not marshalled and no navigation hint is provided. Assuming the cardinality of a tail is 0..1, then without additional "hacky" fields, it is not possible for a client to determine whether this dog has a tail without explicitly requesting for one. (Before you ask, some rare breeds have no tails.)

One could employ an explicit null to indicate that no object exists (vs omitting the property which would indicate that a tail exists). Stated formally, marshal a tail property by default (without an explicit `$expand=tail` directive) if and only if `tail != null`:

	GET serviceRoot/Dogs('1234')
Response:

	{
	  "@odata.context": "serviceRoot/$metadata#Domain.Dog",
	  "@odata.id": "serviceRoot/Dogs('1234')",
	  "id": 1234,
	  "name": "Bobby",
	  "tail": null
	}

This approach introduces an antipattern of differentiating optionally hidden (non-marshalled) properties from nulls which is naturally ambigiuos. I.e. a 'missing property' != 'null property', which is not natively supported by most JSON serialization frameworks (without reverting to custom serializers). (I'm not aware of any schema or IDL that supports this notion, and rightly so.)

Thus OData is useful for piece-wise retrieval of data in a strict composition (cardinality of 1:1) where a nested object is assumed to exist _a priori_, but is awkward in a loose composition (1:0..M) or aggregation (0..N:0..M) scenarios where hiding a property means nothing to the consumer. Additionally _a priori_ assumptions introduce a form of undesired coupling where semantic knowledge of the underlying data may be required and cannot be communicated by the interface contract alone.

Fix:
Option 1. Write additional read-only "hints" in the parent object. E.g.

    {
      "@odata.context": "serviceRoot/$metadata#Domain.Dog",
      "@odata.id": "serviceRoot/Dogs('1234')",
      "id": 1234,
      "name": "Bobby",
      "tailLink": "serviceRoot/Dogs('1234')/tail"
    }

Subsequent invocation with `?$expand=tail` will introduce the `tail` attribute. The client can now safely assume that a tail exits because `tailLink` is not null.

Option 2: Introduce wrapper objects such that the absence of a wrapper indicates a `null`. E.g.
If the tail exists:


    {
      "@odata.context": "serviceRoot/$metadata#Domain.Dog",
      "@odata.id": "serviceRoot/Dogs('1234')",
      "id": 1234,
      "name": "Bobby",
      "tail": {
        "@odata.context": "serviceRoot/$metadata#Common/Reference",
        "link": "serviceRoot/Dogs('1234')/tail/value",
        "value": null // would normally be hidden as per OData conventions
      }
    }

Subsequent invocation with `?$expand=tail/value` will introduce the `value` property of the wrapper.

Alternatively, if the tail was truly missing:

    {
      "@odata.context": "serviceRoot/$metadata#Domain.Dog",
      "@odata.id": "serviceRoot/Dogs('1234')",
      "id": 1234,
      "name": "Bobby",
      "tail": null
    }


#2. Broken PUT semantics

PUT is an idemptotent operation that accepts a complete resource object (sans the read-only/derived fields, which would otherwise be ignored/rejected). Partial updates are not strictly PUT and should be performed using either POST or the newer RFC5789 PATCH verb.

Consider the following retrieval operation on a simple social network:

    GET serviceRoot/People('nigel')
    
Response:
    
    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "single",
      "friends": null // shown here but normally wouldn't be marshalled 
    }

Does Nigel have no friends? Stereotypes aside, assume that friends can be retrieved with:

    GET serviceRoot/People('nigel')?$expand=friends
    
Response:
    
    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "single",
      "friends": [
        {
          "@odata.context": "serviceRoot/$metadata#Person",
          "@odata.id": "serviceRoot/People('alice')",
          "id": "alice",
          "name": "Alice",
          "status": "single",
        },
        {
          "@odata.context": "serviceRoot/$metadata#Person",
          "@odata.id": "serviceRoot/People('bob')",
          "id": "bob",
          "name": "Bob",
          "status": "single",
        },    
      ]
    }

Nigel got married!

    PUT serviceRoot/People('nigel')

What should the request payload be?

Is it (#1):

    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "married",
    }
    
No, because this is not a complete object. The above is a partial update and should be done with a PATCH.

Is it (#2):

    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "married",
      "friends": null
    }
    
No; while the object is complete, it is not accurate. Nigel does have friends.

Is it (#3):

    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "married",
      "friends": [
        {
          "@odata.context": "serviceRoot/$metadata#Person",
          "@odata.id": "serviceRoot/People('alice')",
          "id": "alice",
          "name": "Alice",
          "status": "single",
        },
        {
          "@odata.context": "serviceRoot/$metadata#Person",
          "@odata.id": "serviceRoot/People('bob')",
          "id": "bob",
          "name": "Bob",
          "status": "single",
        }    
      ]
    }
    
Very dangerous, as this updates the complete graph with all the friends (which could have changed since the last request). It's also impractical as friends have other friends and contain circular references. If only OData supported the notion of first-class links...

Fix:
Again, using a wrapper. 

The "lean" GET becomes:

    GET serviceRoot/People('nigel')
    
Response:

    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "single",
      "friends": {
        "@odata.context": "serviceRoot/$metadata#Common/Reference",
        "link": "serviceRoot/People('nigel')/friends/value",
        "value": null // would normally be hidden in OData
      }
    }

The "full" GET response becomes:

    GET serviceRoot/People('nigel')?$expand=friends/value
    
Response:

    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "single",
      "friends": {
        "@odata.context": "serviceRoot/$metadata#Common/Reference",
        "link": "serviceRoot/People('nigel')/friends/value",
        "value": [
          {
            "@odata.context": "serviceRoot/$metadata#Person",
            "@odata.id": "serviceRoot/People('alice')",
            "id": "alice",
            "name": "Alice",
            "status": "single",
            "friends": {
              "@odata.context": "serviceRoot/$metadata#Common/Reference",
              "link": "serviceRoot/People('alice')/friends/value",
            }
          },
          {
            "@odata.context": "serviceRoot/$metadata#Person",
            "@odata.id": "serviceRoot/People('bob')",
            "id": "bob",
            "name": "Bob",
            "status": "single",
            "friends": {
              "@odata.context": "serviceRoot/$metadata#Common/Reference",
              "link": "serviceRoot/People('bob')/friends/value",
            }
          }
        ]
      }
    }

The PUT request becomes:

    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "married",
      "friends": {
        "@odata.type": "serviceRoot/$metadata#Common/Reference",
        "link": "serviceRoot/People('nigel')/friends/value",
        "value": null // would normally be hidden
      }
    }
    
Because the link to `serviceRoot/People('nigel')/friends/value` hasn't changed, sending back the same link value in the PUT request does not break PUT semantics.

Charlie is now friends with Nigel. How can we add Charlie to Nigel's array of friends? The current model allows for one level of indirection, but ideally both the array and the element(s) within should be independently expandable. The fix? Introduce a yet another wrapper and nest expansion directives.

So the "full" GET now becomes:

    GET serviceRoot/People('nigel')?$expand=friends/value($expand=value) 

or alternatively

    GET serviceRoot/People('nigel')?$expand=friends/value($levels=1) 

Response:

    {
      "@odata.context": "serviceRoot/$metadata#Person",
      "@odata.id": "serviceRoot/People('nigel')",
      "id": "nigel",
      "name": "Nigel",
      "status": "single",
      "friends": {
        "@odata.context": "serviceRoot/$metadata#Common/Reference",
        "link": "serviceRoot/People('nigel')/friends/value",
        "value": [
          {
            "@odata.context": "serviceRoot/$metadata#Common/Reference",
            "link": "serviceRoot/People('alice')",
            "value": {
              "@odata.context": "serviceRoot/$metadata#Person",
              "@odata.id": "serviceRoot/People('alice')",
              "id": "alice",
              "name": "Alice",
              "status": "single",
              "friends": {
                "@odata.context": "serviceRoot/$metadata#Common/Reference",
                "link": "serviceRoot/People('alice')/friends/value",
              }
            }
          },
          {
            "@odata.context": "serviceRoot/$metadata#Common/Reference",
            "link": "serviceRoot/People('bob')",
            "value": {
              "@odata.context": "serviceRoot/$metadata#Person",
              "@odata.id": "serviceRoot/People('bob')",
              "id": "bob",
              "name": "Bob",
              "status": "bob",
              "friends": {
                "@odata.context": "serviceRoot/$metadata#Common/Reference",
                "link": "serviceRoot/People('bob')/friends/value",
              }
            }
          }
        ]
      }
    }

Assuming Charlie's already registered with the social network, we add Charlie by PUTting the friends array.

    PUT serviceRoot/People('nigel')/friends

Request:

    {
      "value": [
        {
          "@odata.context": "serviceRoot/$metadata#Common/Reference",
          "link": "serviceRoot/People('alice')",
          "value": null // not needed
        },
        {
          "@odata.context": "serviceRoot/$metadata#Common/Reference",
          "link": "serviceRoot/People('bob')",
          "value": null // not needed
        },
        {
          "@odata.context": "serviceRoot/$metadata#Common/Reference",
          "link": "serviceRoot/People('charlie')",
          "value": null // not needed
        }
      ]  
    }


# Conclusion:
ODdata offers some powerful OOTB capabilities for slicing and dicing graph data, but does not cater well to linked entities, is loosely typed and breaks REST semantics. With additional effort (and introduction of some ugly wrappers(wrappers(wrappers))) it can be used as a compromise to an intrinsically link-aware REST framework. 