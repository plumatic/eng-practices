"Objects" in Clojure: design choices for data and (polymorphic) operations
----
(by [w01fe](http://github.com/w01fe))

In Clojure, there are a potentially daunting number of ways to represent a slice of data that would have been an Object in an OO-land.  For example, we could represent a `Person` with a `firstName` and `lastName` as any of:
 
 - tuple: `["joe" "schmoe"]`
 - plain old map: `{:first-name "joe" :last-name "schmoe"}`
 - struct-map: (Clojure 1.0, basically subsumed by records, forget I mentioned them.)
 - defrecord: `(defrecord Person [first-name last-name])`
 - deftype: `(deftype Person [first-name last-name])`
 - reify: `(defn person [first last] (reify Human (first-name [this] first-name) (last-name [this] last-name)])`
 
At first, I thought this session would just cover these data representations.  But, the whole reason we care about data representation is because we want to make it easy to do the **operations** we want on our data -- thus, it makes no sense to think about data in the absence of functions.  A complicating factor is that we sometimes want these functions to be **polymorphic** -- that is, work (differently) across a variety of different data types.  Again, we are provided with a family of options:
 
 - plain old functions (with `instance?` and explicit conditional logic for polymorphism)
 - multimethods 
 - protocols
 - raw Java interfaces

We'll start by surveying these ingredients and their pros and cons independently, and then discuss some "recipes" for combining them in ways I've found fruitful in various circumstances. 


## Operations

While data abstractions are arguably more fundamental, we'll start by reviewing the options for operations on data.   This way, we'll have full context when we get to the data types.  Here's a table describing major features of the four options mentioned above.

<table border=1>
<tr>
<th></th>
<th><pre>defn</pre></th>
<th><pre>defmulti</pre></th>
<th><pre>defprotocol</pre></th>
<th><pre>definterface</pre></th>
</tr>
<tr>
<td>JVM representation</td>
<td>field lookup + virtual method call</td>
<td>hand-rolled hierarchical dispatch</td>
<td>instanceof check, then: <br>
    - if so, virtual method call<br>
    - otherwise, polymorphic inline cached call
</td>
<td>virtual method call</td>
</tr>
<tr>
<td>Dispatch on?</td>
<td><i>static (not polymorphic)</i></td>
<td><b>any function of arguments</b></td>
<td>class of first argument</td>
<td>class of first argument</td>
</tr>
<tr>
<td>Open (extensible to existing data types)?</td>
<td>N/A</td>
<td><b>yes</b></td>
<td><b>yes</b></td>
<td>no</td>
</tr>
<tr>
<td>Bundled (by data type)?</td>
<td>N/A</td>
<td>no</td>
<td><b>yes</b></td>
<td><b>yes</b></td>
</tr>
<tr>
<td>Efficient?</td>
<td>very good</td>
<td>okay</td>
<td>good</td>
<td><b>best possible</b></td>
</tr>
<tr>
<td>Primitives?</td>
<td>yes, but up to 4 args, only long/double</td>
<td>no</td>
<td>no</td>
<td><b>full support</b></td>
</tr>
<tr>
<td>Repl redefinition</td>
<td><b>great</b></td>
<td>meh</td>
<td>meh <b><a href="#footnotes">[2]</a></b></td>
<td>meh <b><a href="#footnotes">[2]</a></b></td>
</tr>
<tr>
<td>Documentation</td>
<td>docstrings <b><a href="#footnotes">[1]</a></b></td>
<td>docstrings</td>
<td>docstrings</td>
<td><b>static types</b></td>
</tr>
<tr>
<td>Ease of use</td>
<td><b>great</b></td>
<td>good</td>
<td>good</td>
<td>okay <a href="#footnotes">[2]</a></td>
</tr>
<!-- <tr>
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
//-->
</table>



## Data types 

Moving on to data types, we'll cover the features of the four main contenders above.  We intentionally omit 

 - tuples, which in our experience are rarely the right choice, and certainly aren't well-suited for polymorphism
 - `struct-map`s, which are deprecated
 - `proxy` and `gen-class`, which exist primarily for Java interop.
   

<table border=1>
<tr>
<th></th>
<th><pre>hash-map</pre></th>
<th><pre>defrecord</pre></th>
<th><pre>deftype</pre></th>
<th><pre>reify</pre></th>
</tr>
<tr>
<td>JVM representation</td>
<td><a href="http://en.wikipedia.org/wiki/Hash_array_mapped_trie">hash array mapped trie</a></td>
<td>named class with public fields + hash-map for extra keys</td>
<td>named class with public fields</td>
<td>anonymous class with private fields</td>
</tr>
<tr>
<td>Memory usage</td>
<td>up to ~10x contents</td>
<td><b>compact</b></td>
<td><b>compact</b></td>
<td><b>compact</b></td>
</tr>
<tr>
<td>Field access</td>
<td>map get</td>
<td><b>static (.field), fast (:key), slower but dynamic (get); direct access in protocol/interface implementations</b></td>
<td>static (.field); direct access in protocol/interface implementations</td>
<td><i>no</i></td>
</tr>
<tr>
<td>Lookup performance</td>
<td><i>hash lookups</i></td>
<td><b>field access</b><br>(or slightly slower, but optimized keyword lookup)</td>
<td><b>field access</b></td>
<td>N/A (protocol/interface methods only)</td>
</tr>
<tr>
<td>Extensibility (can add arbitrary mappings)</td>
<td><b>a map</b></td>
<td><b>behaves like a map</b></td>
<td>no</td>
<td>N/A</td>
</tr>
<tr>
<td>Primitive support</td>
<td>no</td>
<td><b>yes</b> (on base members)</td>
<td><b>yes</b></td>
<td>yes (as supported by interfaces)</td>
</tr>
<tr>
<td>Typed Object subclass members</td>
<td>no</td>
<td>no(t yet)</td>
<td>no(t yet)</td>
<td>N/A</td>
</tr>
<tr>
<td>Equality</td>
<td>value</td>
<td>value+type<br>(just value for Java .equals/.hashCode)</td>
<td>identity<br>(overridable)</td>
<td>identity<br>(overridable)</td>
</tr>
<tr>
<td>Mutable fields</td>
<td>no</td>
<td>no</td>
<td><b>yes</b> (private only)</td>
<td>no</td>
</tr>
<tr>
<td>Serialization</td>
<td><b>great</b></td>
<td>good<br>(pr-str works, but json encoding loses types, etc.)</td>
<td>ok<br>(custom)</td>
<td>no</td>
</tr>
<tr>
<td>Type for dispatch</td>
<td>not really<br>(ad-hoc :type field)</td>
<td><b>yes</b><br>(generates named class)</td>
<td><b>yes</b><br>(generates named class)</td>
<td>sort of<br>(generates anonymous class)</td>
</tr>
<tr>
<td>Works with protocols/interfaces?</td>
<td>no</td>
<td><b>yes</b>
<td><b>yes</b>
<td><b>yes</b>
</tr>
<tr>
<td>Repl redefinition</td>
<td><b>great</b></td>
<td>meh <b><a href="#footnotes">[2]</a></b></td>
<td>meh <b><a href="#footnotes">[2]</a></b></td>
<td>good</td>
</tr>
<tr>
<td>Documentation</td>
<td>meh <b><a href="#footnotes">[1]</a></b></td>
<td>ok <b><a href="#footnotes">[1]</a></b></td>
<td>ok</td>
<td>ok</td>
</tr>
<tr>
<td>Ease of use</td>
<td><b>great</b></td>
<td>good<br>(supports map lookups, but no map-vals, etc)</td>
<td>ok<br>(Java field access only, you bring the sugar)</td>
<td>good<br>(no ceremony around names)</td>
</tr>
<!--
<tr>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
//-->
</table>


## Recipes 

Now I understand how all the pieces work.  But how should I represent my Widget?

This section will aim to provide some rules and heuristics for selecting appropriate data and operation types.  Of course, please keep in mind that the answer is not not always clear-cut, since it involves the interactions of the above features, which can be even more complex and nuanced than the individual differences in data types and operations.  


### No polymorphism

You have a single data type, and one or more operations you want to perform on it.

This is a relatively easy case, since you can mostly refer to the 'data types' table to figure out your best course of action.  If you need primitive support (etc.), you should probably use a defrecord.  If you (really) need mutable fields, you're stuck with deftypes.  If efficiency is not a concern, maps are the simplest and easiest option.

Regardless, if your data is shared across many namespaces, or serialized and stored or shared across process boundaries, you should probably have a concrete schema to refer back to.  deftypes and defrecords give you some of this, but don't capture field constraints beyond primitive values, so it's prudent to use a schema library [[1]](#footnotes) to precisely describe your data for documentation and safety, which can work just as well with plain maps as more structured types.  

For operating on your data, plain old functions are the simplest and typically best option.  We recommend that you use `safe-get` [[3]](#footnotes) to access fields, use docstrings and schemas to document your code, and organize your namespace into clear public and private sections.  

With this discipline under your belt, the only real benefits of using records and interfaces or protocols are primitive support, and the appearance of lexical scope for your data members.  The price you pay for these features is the extra ceremony around declaring interfaces and data classes, plus (in my opinion) slightly decreased ease of use.  

If you do need polymorphism, however, ordinary functions are usually not a great choice.  A single `instance?` check or `case` on `:type` isn't the end of the world, and sometimes is the simplest and cleanest solution -- but once these conditionals start appearing in multiple places, there's a good chance you're doing it wrong.



### Extreme polymorphism, or polymorphism without data types

If you need polymorphism of an exotic form, where you're not just conditioning on the class of the first argument, then you need multimethods.  In our experience, this is a pretty rare occurrence.  Multimethods are so general that you can pick the data format best suited to your application on its own merits.  If you want to use maps for your data representation, a simple `:type` field mapping to a keyword can be used for dispatch.

Similarly, if you want an extensible method without a corresponding concrete data type, multimethods give you a way to declare open dispatch without tying you down to a concrete data representation.  For example, you can make a function that dispatches on its first argument value (not class), which anyone can extend.  

Beyond these cases, you should probably think hard before using a multimethod.  This is especially true if you have multiple polymorphic methods, since you'll need to repeat your dispatch logic in each multimethod if you choose this option.  


### Maximum mungeability 

We've almost reached the end of the road for plain old maps as well.  But before we get there, it's probably worth mentioning one more way to achieve polymorphism: storing functions as fields, ala JavaScript (or many languages that came before it).  

```clojure
(def my-obj 
  {:foo (fn [this y] (bar (:baz this) y))
   :baz 12})

;; caller 
((:foo my-obj) my-obj 12)             
```

This gives you flexibility to do crazy things that are difficult to achieve with more rigid interfaces and records; you can `merge` "objects", `assoc` new "methods", and bring all the other tools you usually use to manipulate data to bear on constructing your polymorphic objects. 

That said, this method is rather clunky and hard to understand.  (Where the hell is the :foo function of my-other-obj defined?).  We've only had one or two cases where we felt that we needed this power, and they've all been replaced with Graph [[3]](#footnotes), which has a similar model but abstracts away some of the complexity of this approach, when your object is really trying to represent a flexible computation process with many steps.

And now, we've hit the end of the road for plain old maps.


### Maximum efficiency

If you need maximal memory efficiency and/or unboxed primitives, you must use defrecord, deftype, or reify.  If you need mutable members, you must use deftype.  If you need custom equality semantics or map semantics, you must use deftype or reify.  If you want efficiency and (less efficient) extensibility, you probably want defrecord.  

If you need complex logic that returns primitives, you need to use Java interfaces to work with these objects.

If you need none of these things, you don't want `deftype` or `definterface`.


### Simple polymorphism 

We're left with a common case, where you do need polymorphism but don't require extreme performance or complex dispatch.  

We've already covered the cases where you want to use plain old functions, multimethods, and interfaces; and plain old maps and `deftype`s on the data type side.  This leaves us with a single operation type, protocols (or `definterface+` [[2]](#footnotes), if you're concerned about [memory/perf implications of protocol dispatch](https://groups.google.com/forum/?fromgroups#!topic/clojure-dev/jKMliKIf9Fg)), and two data types, `defrecords` and `reify`.  

At this point, the decision is pretty simple, based on your use case.

`reify` is simpler -- you don't have to explicitly name your fields since Clojure automatically captures things referenced in lexical scope, and you don't need to define a separate constructor function if you want constructor-like logic.  The price you pay for this simplicity is that the objects you create are *opaque*: if you want access to any 'fields' you will have to create protocol methods for this purpose.  

On the other hand, `defrecord` is *transparent* -- people can examine your object, pull out fields, reason about the `Class` of your data, and so on.  

Thus, the choice of `reify` or `defrecord` primarily comes down to what you are trying to represent; if you're primarily concerned with *data*, you probably want `defrecord`, whereas if you only care about *behavior* then `reify` may be a simpler choice.


### Abstract data members.

We're basically done with our tour, but there's one issue we haven't touched on yet: abstract data members.  What if you have multiple data types (posts and URL documents, employees and customers, etc.) and want a data-centric interface (all documents have titles, all people have names, etc.).  None of Clojure's data types allow for implementation inheritance, so if your employers and customers are separate records, you're out of luck for getting the static checking of Java-style field access (.first-name r).

In this case, there are three options at your disposal, none of which is really ideal:

 - Use an informal interface (a.k.a docstring): "All people have :first-name and :last-name keys".  This should probably be backed by schemas [[1]](#footnotes) and liberal use of `safe-get` [[3]](#footnotes) to ensure your data measures up.
 - Be oh-so-formal: declare a protocol full of 'getter' functions, and fill each of your records with methods like `(first-name [this] first-name)`.  This pain can sometimes be alleviated by defining a *single* record or reify that goes through this ceremony, and letting each of your 'objects' share this single constructor -- they can even pass in functions that the single implementation delegates to, if you need limited polymorphism.
 - If you're not concerned with polymorphism but just a hierarchy of data types, you may be able to flip things around so that all data types are represented with a single `defrecord`, that has a `:type` and `:type-info` fields to allow extensibility.
 

-----------------

<a name="footnotes"></a>
 
  1. [Schema](https://github.com/plumatic/schema) is a library for declaring data shapes, and annotating functions with input and output schemas.  Besides their other benefits, Records have the documentation advantage of having a concrete description that is type-hintable; among other things, schema brings these same benefits to ordinary Clojure maps.
  2. [Potemkin](https://github.com/ztellman/potemkin) provides some great tools for dealing with interfaces, protocols, records, and so on.  In particular, it provides variants of `defprotocol` and `defrecord` that are more repl-friendly, and an implementation of `definterface` that's a drop-in replacement for `defprotocol`, allowing full primitive support with automatic wrapper functions (but without the open-ness of protocols, of course).
  3. [Plumbing](https://github.com/plumatic/plumbing) is a library of Clojure utility functions, including Graph, a tool for declarative description of functional processes.
  


