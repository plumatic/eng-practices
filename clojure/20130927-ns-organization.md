Readable Clojure ns Layout
----
(by [aria42](http://github.com/aria42))

It's crucial to create namespaces that not only do their job, but minimize the amount of time it takes a fresh pair of eyes to understand the goal of the namespace, what its abstractions are, and how to use it. 

Clojure provides virtually no constraints about how the layout of a file should be. You could for instance have multiple namespaces in a file. (Please don't ever do this or you will be ostracized.) 

For these reasons, it's crucial to have conventions about the layout of a namespace file. Below is an example of what a good `ns` structure might look like: 

```clojure
(ns flop.linear-regression
  "Standard least squares linear regression algorithm.

   Usage (only public function):
   (learn-linear-regression
       [[50.0, {:slobber 1 :furry 1}],
        [20.0, {:meowy 1 :furry 1}]])
   > {:meowy -3.3333315890376487,
      :furry 23.33332778138144,
      :slobber 26.666659370419094}

  Uses numerical optimization on least squares objective and internally
  indexes features, so should work on very large problems."
  (:use plumbing.core)
  (:require
   [schema.core :as s]
   [flop.optimize :as optimize]
   [flop.array :as array]))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; Schemas

(def SparseFeatureVector
  {Object double})

(def LabeledRegressionExample
  [(s/one double "target value")
   (s/one SparseFeatureVector "sparse feature vector")])

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; Feature Indexing

(defn build-feat-index [training-data]
  (let [all-feats (vec (set (mapcat (comp keys second) training-data)))
        feat-idx (into {} (map-indexed (fn [i x] [x i]) all-feats))]
    [all-feats feat-idx]))

(defn indexed-feat-vec [feat-idx fv]
  (for [[f v] fv
        :let [i (feat-idx f)]
        :when i]
    [i v]))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; Objective Function

(defn least-square-predict [weights indexed-fv]
  (sum (fn [[feat-idx v]]
         (* (array/aget weights feat-idx) v))
       indexed-fv))

(defn least-square-objective [num-feats indexed-training-data weights]
  (let [grad (double-array num-feats)
        val (sum
             (fn [[target idx-fv]]
               (let [guess (least-square-predict weights idx-fv)
                     diff (- guess target)]
                 (doseq [[feat-idx v] idx-fv]
                   (array/ainc grad feat-idx (* diff v)))
                 (* 0.5 diff diff)))
             indexed-training-data)]
    [val grad ]))

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; Public

(s/defn learn-linear-regression :- SparseFeatureVector
  "Takes a sequence of LabeledRegressionExamples and returns a SparseFeatureVector from feature keys to their learned least-squares double value"
  [training-data :- [LabeledRegressionExample]]
  (let [[all-feats feat-idx] (build-feat-index training-data)
        indexed-training-data (map (fn [[target fv]]
                                     [target (indexed-feat-vec feat-idx fv)])
                                   training-data)
        weights (optimize/lbfgs-optimize
                 (partial least-square-objective (count all-feats) indexed-training-data)
                 (double-array (count all-feats))
                 {:print-progress true :max-iters 1000 :thresh 1e-6})]
    (for-map [[f i] feat-idx]
      f (array/aget weights i))))
```
The conventions here aren't without exception. In particular, for a namespace like `plumbing.core` which is exceptional for a number of reasons, this wouldn't make sense, but it's the exception and not the rule. If you generate a new namespace, you should have a good reason to deviate. 

Let's go over this file section by section.

## `ns` form

A good layout starts with the `ns` form at the top of each file.

###  Always format dependencies in namespace form in `use`/`require`/`import` order with each section sorted by topological dependency:

Make sure when you `:require`, that you use the last element of the fully-qualified `ns` unless (1)  the last element is `core`, in which case use the 2nd to last or (2) conflicts with another required `ns`. 

#### JVM Clojure
The only thing you should `use`  is `plumbing.core` and in tests, you can use `clojure.test` and the tested ns. Exceptions to the `:require` above exists for `schema.core`, which can be required as just `s`.

#### ClojureScript
There is no `use`. Ensure your `require`s are dependency-sorted. 

### Use an `ns` doc string to describe the purpose of the namespace 

A good ns doc-string can tell a reader about the basic purpose of the namespace as well as highlight important public functions and namespace conventions. Things you might want to include in the `ns` doc-string

* a 1-2 sentence description of the purpose of the namespace
* example usage
* what is meant to be used by clients? Protocols, functions, schemas, etc.
* more detailed information about implementation
* namespace internal conventions about variables

## Separate code into pragma blocks

Actual `ns` code should be separated into 'pragma' blocks as much as possible to allow for high-level navigation (we can even have a keyboard command to show an outline mode of the file and jump to a pragma block). For instance, 

```clojure
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;; Data serialization

(defn load-articles [^java.io.File in-file] 
   ...)
   
 (defn write-articles [^java.io.File out-file articles]
   ...)
````
to delimit the serialization functions. 

There should be an overall structure of pragma blocks and there should be a convention about the names of pragma blocks we expect to re-use between files, especially for code which is meant to be read/used by clients of the `ns`.

* Schemas: All the schemas that are part of the interface of the `ns` (meaning that others could use the schema for their own functions or these schemas are part of the arguments or return values to public functions in your namespace) 
* Protocols: All the protocols which are part of the interface of the `ns` (either because client `ns` will provide record/type implementations of the protocols or use them via objects you provide).
* Records/Types:  Any records or types you provide (in the case of records, you are also saying the generated factory functions are part of your public interface).
* Public: There should be a `Public` pragma block at the bottom of your file which is meant to be the public functions of your namespace. The only reason this goes at the bottom is because all the private functions you are using to support these functions must come earlier (Using forward `declare` for all your private functions is not tenable).  There are special more stringent rules for Public functions (see below).

You can and should organize all your private/internal code into descriptive pragma blocks even though they aren't part of the interface.

##  The Public Pragma Block

Virtually all `ns` should have a Public pragma block which is what you should read after the `ns` form itself and the public protocols/schemas/records/types (see pragmas above).  All functions in Public should obey the following:

* All should have a doc-string describing what they take and return
* For complex parameter or return types, type-hint using the schemas/protocols/records/types from the pragmas at the top.

The golden rule for public functions really is **they should be understandable after reading the `ns` form, the top protocol/schema/record pragmas, and the function doc-string**. You shouldn't have to read any of the rest of the files or other files (unless they are explicitly used in the exposed schemas/protocols/records) to understand what the function does. 


