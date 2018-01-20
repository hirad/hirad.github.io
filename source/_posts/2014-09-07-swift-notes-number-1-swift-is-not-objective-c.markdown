---
layout: post
title: "Swift Notes #1: Swift Is Not Objective-C"
date: 2014-09-07 13:01:04 -0700
comments: true
categories: programming
---
Over the summer, I've been really busy trying to ship products for my startup. At the same time, I've been teaching (part-time) a two-month-long iOS bootcamp at [Lighthouse Labs](http://www.lighthouselabs.ca/), which involves preparing lecture material, homework exercises, etc. It's been a busy summer, and I haven't had a chance to play with Swift as much as I wanted to. 

So last week, a few of the Lighthouse Labs instructors and TAs got together for some dedicated Swift time. These are my notes about what I learned, while trying to implement [Skip Wilson's challenge for TopCoder](http://www.topcoder.com/challenge-details/30045372/?utmSource=SkipAllMighty&utmCampaign=learnswift&utmMedium=codechannel) - a playground program to convert any number from -1 million to 1 million from a numerical representation to its english representation. <!--more-->

My first idea on this was that in converting a number to text, it might be easier to treat the number as a string, so I started by looking for a decimal point in a numerical string. The numbers after the decimal point are easier to convert since you can just write them digit by digit (ie. "some number *point* one two three"). 

We need to determine whether a `String` contains a particular character. In Obj-C, we're used to using methods on `NSString` to do whatever we need to do with our string objects. Swift doesn't work that way. If you Cmd-click into `String`, you'll see that...

a) It is a collection type. Collection types have elements accessible with a subscript. In String's case, its elements are instances of Character.

    extension String : CollectionType {
        struct Index : BidirectionalIndexType, Comparable, Reflectable {
            func successor() -> String.Index
            func predecessor() -> String.Index
            func getMirror() -> MirrorType
        }
        var startIndex: String.Index { get }
        var endIndex: String.Index { get }
        subscript (i: String.Index) -> Character { get }
        func generate() -> IndexingGenerator<String>
    }
  
b) Nowhere does Swift's `String` struct provide a function to get the index of a particular character in the string, if it exists. Of course, in Objective-C's `NSString`, we have many a method on `NSString` that we could use for this. 

My first solution was to write something like this:

    extension String {
        func indexOfCharacter(c : Character) -> String.Index.Distance? {
           let length = countElements(self)
           for var i = 0; i < length; i++ {
                let index = advance(self.startIndex, i)
                let char = self[index]
                if char == c { return distance(self.startIndex, index) }
           }
    
           return nil
        }
    }

Writing this was way harder than it looks - it involved a lot of digging in the Swift header. I already knew about `advance(start:, n:)` from a previous experiment where I tried to get a substring from a Swift `String` type (see below). I had to find the `distance(start:,end:)` function as well. I don't remember how I found it (maybe through reading more StackOverflow answers). 

    func distance<T : ForwardIndexType>(start: T, end: T) -> T.Distance

This demonstrates one of things that's a learning curve in Swift. There are generics, and the types can be `typealias`'d to domain-specific names. That will make code much more readable in some senses, but it also means that sometimes you have to decipher what kind of thing you're dealing with. From the above declaration of the distance function, if you Cmd-Click into the `ForwardIndexType`, you'll see:

    protocol ForwardIndexType : _ForwardIndexType {
    }
  
[Don't ask me why this is there](http://en.wikipedia.org/wiki/Vermiform_appendix). But obviously, we need to keep digging, so we Cmd-Click into `_ForwardIndexType`. Voila, finally we see this:

    protocol _ForwardIndexType : _Incrementable {
        typealias Distance : _SignedIntegerType = Int
        typealias _DisabledRangeIndex = _DisabledRangeIndex_
    }
  
A `Distance` is just an `Int`. 

Even up to this point, I've started noticing a pattern. What I tried to do above with my extension to `String` was to shoehorn Swift's `String` type to act more like `NSString`, which I'm used to. But it looks like the "Swiftier" way of doing things is to have global functions that act on generic types.

c) `String` also has several different "views" such as `UTF8View` or `UnicodeScalarView`. These are subtypes of the `String` type (not sure yet if that's the right name for them): 

    extension String {
        struct UTF8View : CollectionType, Reflectable {
            struct Index : ForwardIndexType {
                func successor() -> String.UTF8View.Index
            }
            var startIndex: String.UTF8View.Index { get }
            var endIndex: String.UTF8View.Index { get }
            subscript (i: String.UTF8View.Index) -> CodeUnit { get }
            func generate() -> IndexingGenerator<String.UTF8View>
            func getMirror() -> MirrorType
        }
        var utf8: String.UTF8View { get }
        var nulTerminatedUTF8: ContiguousArray<CodeUnit> { get }
    }
    
    extension String {
        struct UnicodeScalarView : Sliceable, SequenceType, Reflectable {
            struct Index : BidirectionalIndexType, Comparable {
                func successor() -> String.UnicodeScalarView.Index
                func predecessor() -> String.UnicodeScalarView.Index
            }
            var startIndex: String.UnicodeScalarView.Index { get }
            var endIndex: String.UnicodeScalarView.Index { get }
            subscript (i: String.UnicodeScalarView.Index) -> UnicodeScalar { get }
            subscript (r: Range<String.UnicodeScalarView.Index>) -> String.UnicodeScalarView { get }
            struct Generator : GeneratorType {
                mutating func next() -> UnicodeScalar?
            }
            func generate() -> String.UnicodeScalarView.Generator
            func compare(other: String.UnicodeScalarView) -> Int
            func getMirror() -> MirrorType
        }
    }

From this, I noticed that `String` itself is not 'Sliceable' - meaning you can't get substrings from it, unless you get one of the specific views. But one problem I found with the `Sliceable` protocol is that it defines the "subslicing" syntax this way:  
  
    protocol Sliceable : _Sliceable {
        typealias SubSlice : _Sliceable
        subscript (bounds: Range<Self.Index>) -> SubSlice { get }
    }

So, if I wanted to get a substring from index 2 to 4, I'd need something like this:

    let str = <#some string#>
    let subString = str.utf8[ <#some range#> ]

That <#some range#> is a problem. The range that the subscript function on `String.UTF8View` takes is one with `String.UTF8View.Index` indices. As far as I know, there is no way to create these. The only way I've found to get substrings is from this [StackOverflow answer from Rob Napier](http://stackoverflow.com/a/24046551/1827743). 

Ok, so now we know how to find where the decimal point is, and we know how to get substrings. We can break up the number string we're trying to convert to words (eg. "1234.56") into two strings ("1234" and "56"). After that, it's just a question of coming up with an algorithm to do the conversion. That's interesting, but it's not Swift-specific. 

What is Swift-specific is how to better write the `indexOfCharacter` function. As I mentioned above, it seems the "Swiftier" way is to have a global function act on a generic type. Let's look at some examples of such functions that come with Swift:

    func countElements<T : _CollectionType>(x: T) -> T.Index.Distance
  
`countElements` is a function whose sole argument should be any collection type, and its return value is a Distance type of Index type of the collection type that was passed in.

Another example:

    func contains<S : SequenceType where S.Generator.Element : Equatable>(seq: S, x: S.Generator.Element) -> Bool
  
You can use the `contains` function, to, for example, check if a string contains a particular character:

    var str = "my dollar amount: $99"
    var willBeTrue = contains(str, "$")

Notice that this time, we don't care at all that the `str` is a collection type. `contains` only cares that its argument is a sequence type.

    protocol SequenceType : _Sequence_Type {
        typealias Generator : GeneratorType
        func generate() -> Generator
    }
  
And what is `_Sequence_Type`?
  
    protocol _Sequence_Type : _SequenceType {
    
        /// A type whose instances can produce the elements of this
        /// sequence, in order.
        typealias Generator : GeneratorType
    
        /// Return a generator over the elements of this sequence.  The
        /// generator's next element is the first element of the sequence.
        func generate() -> Generator
    }
  
And what is a `GeneratorType`?
  
    protocol GeneratorType {
        /// The type of which `Self` is a generator.
        typealias Element
    
        /// If all elements are exhausted, return `nil`.  Otherwise, advance
        /// to the next element and return it.
        ///
        /// Note: after `next()` on an arbitrary generator has returned
        /// `nil`, subsequent calls to `next()` have unspecified behavior.
        /// Specific implementations of this protocol are encouraged to
        /// respond by calling `preconditionFailure("...")`.
        mutating func next() -> Element?
    }
  
So how can we follow this pattern for our `indexOfCharacter(c:)` function? Looks like the better would be to write something that works with any `_CollectionType`, since that is the protocol that enforces having an index:

    func indexOfElement<T: _CollectionType>(x: T, e: T.Generator.Element) -> T.Index { ... }
  
If we try to implement this function, we soon hit a problem where we can't iterate over the items in our collection with an index because `_CollectionType` does not conform to `SequenceType` (it conforms to `_SequenceType`). So... some more investigation and we find this: 

    protocol CollectionType : _CollectionType, SequenceType {
        subscript (i: Self.Index) -> Self.Generator.Element { get }
    }
  
So, our function should look like: 

    func indexOfElement<T: CollectionType>(x: T, e: T.Generator.Element) -> T.Index { ... }
  
Why is it important that we use a type that conforms to `SequenceType`? Because we need a way of iterating over the collection with an index AND the element at that index, and the function that helps us do that (`enumerate`) works on a `SequenceType`. Our final function ends up looking like this:

    func indexOfElement<T: CollectionType where T.Generator.Element : Equatable>(x: T, e: T.Generator.Element) -> T.Index? {
      let length = countElements(x)
      for (index, element) in enumerate(x) {
        if element == e {
          return advance(x.startIndex, index as T.Index.Distance)
        }
      }
      
      return nil
    }
  
Finally, this does what we want. Of course, after going through this song and dance, I find out there is an existing function called `find`:

    func find<C : CollectionType where C.Generator.Element : Equatable>(domain: C, value: C.Generator.Element) -> C.Index?

As you can tell from the signature, it does exactly what we tried to achieve above (probably with very similar implementation). So this shows yet another example of how I'm trying to shoehorn Swift to become like Objective-C: I would have never thought to look up 'find' but I did look up a few different functions names that seemed intuitive to me from Objective-C: "`indexOfItem`", "`indexOfElement`", "`indexOf`", etc.

It's a new world.