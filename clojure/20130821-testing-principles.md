#A Few Testing Principles 

(by [ihat](http://github.com/ihat))

##Agenda

1. [Why test?](#why)
2. [Different types of tests](#types)
3. [Real world testing](#real)

###<a id="why"></a>1. Why test?

#### A short story

This past monday, @cpetzold and I paired on creating activity notifications for comment submissions.

Turns out, the submit comment API endpoint contained a lot of (unrefactored) logic for not only the creation of the comment, but also for email notifications to the original poster. We wanted to extract functionality without breaking the endpoint.

Fortunately, there were tests! @cpetzold and I refactored with confidence, and eventually slimmed down the submit comment test and created finer-grained unit tests around email notifications and activity notifications.

#### Moral of the story

The three goals of testing (from ["Fast Test, Slow Test"](http://www.youtube.com/watch?v=RAxiiRPHS9k), Gary Bernhardt)

1. Prevent regression
2. Prevent fear
3. Prevent bad design

The short story shows:

- (Prevent regression) We could introduce change that would not break current functionality.
- (Prevent regression) "If you don't test your code, your customers will" ([Pragmatic Programmer](http://www.amazon.com/Pragmatic-Programmer-Journeyman-Master-ebook/dp/B000SEGEKI)).
- (Prevent fear) The tests encouraged us to refactor aggressively.
- (Prevent fear) The tests served as documentation and contract; got us up to speed faster.

I'm skipping how testing drives the code design process (TDD) for now.  But generally speaking, if you are able to write testable code, then chances are the code is loosely coupled and highly cohesive.

#### Two more reasons!

I learned these the hard way from past and current jobs.

1. The person writing the code, where the mistake happened, and the person testing the code, are different -- pain is not felt where it should be felt.
2. Consider the life-time value of code. Don't optimize for write-only.


###<a id="types"></a> 2. Different types of tests: unit, integration and acceptance tests
Following material is a summary of these two sources:

- [Clean Code Talks](http://misko.hevery.com/2008/11/04/clean-code-talks-unit-testing/) by Misko Hevery (author of AngularJS)
- [Growing Object-Oriented Software, Guided by Tests](http://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627) by Freeman, Pryce

#### Analogy: Testing a car

- Framework for testing a car: build a machine that pretends you're a driver, step on brakes and accelerator, etc.
    - Known as a scenario test
    - Prove the car works
    - The problem is with the execution
    - It's horribly slow
    - When something breaks, it doesn't pinpoint what's wrong
- You'd want to test each main component separately
    - e.g., the powertrain, which consists of the engine, transmission, drive shafts, differentials, etc.
- And you can want to isolate a component of the powertrain for testing
    - e.g., the engine


#### Three types of tests defined

Unit tests

- What: Test individual methods in isolation
- Answers: Do our functions do the right thing? Are they convenient to work with?

Integration tests

- What: Test collection of functions across namespaces as subsystems
- Answers: Does our code work against code we can't change?
- Examples:
    - In Rails / Django, these are typically the controller tests that require the database
    - Any test that involves actual MySQL, Mongo, third party software that we haven't provided a test substitute

Acceptance tests

- What: Test the whole system pretending to be the user
- Answers: Does the whole system work?
- Examples:
    - Spinning up selenium
    - api/system-test: spinning up server



#### Trade-offs between these tests

![Test tradeoff](http://f.cl.ly/items/170o0c3m100e2l101g33/testing.jpg)


Acceptance tests:
- High confidence happy paths OK
- Hard / slow to reproduce
- Many things come into play
- Highest customer / external feedback
- Slow!

Integration tests:
- High confidence integrated sub-systems OK
- Lower app coverage, need more of them
- Still need to debug what went wrong

Unit tests:
- High confidence the function under test OK
- Need lots of these (a good thing)
- Quickest developer feedback
- Shows developers the internal quality


#### "Integration tests are a scam"

(see [The Code Whisperer](http://blog.thecodewhisperer.com/2010/10/16/integrated-tests-are-a-scam/))

- path count: 2^n number of tests for integration tests
    - any `if` statement, `try` ... `catch`
    - testing the whole thing
    - 500 conditions: 2^500
    - suite runtime is super linear. Integration test is super linear.
- The worst feeling ever? Make a bunch of changes, then run system test or selenium test and
  a whole bunch of stuff broke. e.g., a random 500 response. worst. Start up the logger and
  off we go...


###<a id="real"></a> 3. Real world testing

> "There is no secret to writing tests... there are only secrets to writing testable code"

> -- Misko Hevery


#### In an ideal world

- All functions are pure
- All computation is local 
- Data in and data out (no side effects)
- Testing is simple: you'd just assert the expected output given input

The idealized view is:
```
x -> (f) -> y
```
... where `x`, `y` are data and `(f)` is a function.




#### But real life is messy

- Real life is full of mutation and state, so we write functions that:
    - Have side effects, e.g.,
        - Creating a user record in a database
        - Issues an API request to mark docs as viewed
    - Depend on other functions that have side effects

Our simple diagram above now looks like this:
```
   x -> (f) -> y
         |
         ---> (g)
```



#### Rules of the game change when using functions with side effects

- The order of evaluation matters
- We need to know the context in which a function is run
- Testing is no longer simple



Scenario: How would you add functionality to the following untested code?
- We would like to record public linkage events to Mixpanel
- The API endpoints end up calling `record-public-linkage-event` below:

```clojure
(ns user)
(require '[clojure.test :as test])
(require '[user-data.event-log :as event-log])

;; compile the other file

(defn record-public-linkage-event
  [mongo-datastore env request user-id event-body]
  (let [event-type (:type event-body)
        public-id (:public-id event-body)
        event-payload (assoc event-body
                        :munged_public_id (.replace public-id "-" "_")
                        :user_id user-id)]
    (event-log/write-public-linkage-event mongo-datastore event-payload)))
```


What is this function doing?
- it munges `request`, `user-id` and `event-body` into a form suitable for logging (functional)
- it calls `event-log/write-public-linkage-event`, a collaborating function


Suppose we wanted to test this, how do we do it?
1. First write the expectation
    - `event-log/write-public-linkage-event` should receive `mongo-datastore` and the munged event data

```clojure
(deftest record-public-linkage-event-test
  (record-public-linkage-event mongo-datastore env request user-id event-body)
  (testing "event-log/write-public-linkage-event should be called with transformed event"
    (should-receive event-log/write-public-linkage-event
                    mongo-datastore env request user-id
                    munged-event-body)))
```


2. Fill in the details...

```clojure
(deftest record-public-linkage-event-test
  (let [mongo-datastore (gensym)
        env :test
        request {:cookies {"stage_p_public" {:value "foobar-cookie"}}}
        user-id (rand-int 10)
        event-body {:type "public-linkage" :key1 "val1"}]

    (testing "event-log/write-public-linkage-event should be called with transformed event"
      (record-public-linkage-event mongo-datastore env request user-id event-body))))
```

But how do I assert? Use a test double! (For details see [Martin Fowler's blog](http://martinfowler.com/articles/mocksArentStubs.html)

```clojure
(defmacro test-double
  "Creates a test double of a function given a fully qualified function name."
  ([] `(gensym))

  ([fqfn]
     (let [args (first (:arglists (meta (resolve fqfn))))
           args-count (count args)]
       `(let [received-args# (repeatedly ~args-count #(atom nil))]
          (reify
            clojure.lang.IFn
            (invoke [this# ~@args]
              (doseq [[arg# received#] (map list ~args received-args#)]
                (reset! received# arg#)))

            clojure.lang.ILookup
            (valAt [this# k# not-found#]
              (-> (map (fn [arg# received#] [(keyword arg#) @received#]) '~args received-args#)
                  (#(into {} %))
                  (get k# not-found#)))

            (valAt [this# k#] (.valAt this# k# nil)))))))

(test/deftest record-public-linkage-event-test
  (let [mongo-datastore (test-double)
        env :test
        request {:cookies {"stage_p_public" {:value "foobar-cookie"}}}
        user-id (rand-int 10)
        event-body {:type "public-linkage" :key1 "val1" :public-id "foobar-cookie"}
        event-log-double (test-double event-log/write-public-linkage-event)

        expected-event {:type "public-linkage"
                        :key1 "val1"
                        :public-id "foobar-cookie"
                        :munged_public_id "foobar_cookie"
                        :user_id user-id}]

    (with-redefs [event-log/write-public-linkage-event event-log-double]
      (test/testing "event-log/write-public-linkage-event should be called with transformed event"
        (record-public-linkage-event mongo-datastore env request user-id event-body)
        (test/is (= mongo-datastore (:mongo-datastore event-log-double)))
        (test/is (= expected-event (:event event-log-double)))))))
```


The `with-redefs` is *isolating* the function under test.

Taking a step back, the only thing worth testing in this function is the data manipulation, and that's borderline trivial. The data manipulation part could in principle be decoupled and tested. Whether the components are hooked up may be tested via an integration test, but we don't get much benefit out of that test.

Wrinkle in the above example. Turns out `event-log/write-public-linkage-event` had hidden dependencies defined elsewhere `event-log`. This is a good example of how implicit dependencies can really hurt testability and actual reasoning of the code.

#### Holy grail

Have a functional core (data manipulation)
- Lots of decision paths, no dependencies, isolated
- Conducive to unit testing

Surrounded by an imperative shell (side effects)
- Lots of dependencies, few decision paths
- Conducive to integration testing


One could actually make the argument that this code is not worth testing because it's so obviously correct.

But now that we have a test in place, we know that changes to this will:
- Prevent regression
- Prevent fear of refactoring

```clojure
(deftest record-public-linkage-event-test
  (let [mongo-datastore (test-double)
        env :test
        request {:cookies {"stage_p_public" {:value "foobar-cookie"}}}
        user-id 101
        event-body {:type "public-linkage" :key1 "val1"}
        event-log-double (test-double user-data.event-log/write-public-linkage-event)
        mixpanel-double (test-double mixpanel/enqueue-event!)

        expected-event {:type "public-linkage"
                        :key1 "val1"
                        :public_id "foobar-cookie"
                        :user_id user-id}]

    (with-redefs [event-log/write-public-linkage-event event-log-double
                  mixpanel/enqueue-event! mixpanel-double]
      (record-public-linkage-event mongo-datastore env request user-id event-body)

      (testing "event-log/write-public-linkage-event should be called with transformed event"
        (is-= mongo-datastore (:mongo-datastore event-log-double))
        (is-= expected-event-payload (:event event-log-double)))

      (testing "mixpanel/enqueue-event! should be called with transformed event"
        # I can now add a test to check whether mixpanel is called properly! Perhaps with more data transformation as well.
        ))))
```


#### Other (inchoate) critiques of this function

- I shouldn't need to know that `event-log/write-public-linkage-event` requires `mongo-datastore`
- How can we best separate instantiation of object graph from business logic under test?


#### Parting thoughts

- Testing is all about trade-offs
- [Test Driven Development (TDD) is a pragmatic choice](http://blog.8thlight.com/uncle-bob/2013/03/06/ThePragmaticsOfTDD.html)
- When you are optimizing for very short term, TDD may be too costly, as in the case of:
    - GUIs
    - functions that are trivial and obviously correct
- There are certain (few limited) cases where the benefits of testing are outweighed by the cost of maintenance.
- Some things are also very hard to test, e.g., UI, especially when they are still being designed.
- Many front-end projects don't have large testing components.
- But remember, programmers are constantly in maintenance mode. Optimize your code for readability and maintenance, not the first 10 minutes of its life.
