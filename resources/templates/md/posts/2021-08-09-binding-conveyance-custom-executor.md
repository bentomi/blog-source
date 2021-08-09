{:title "Custom executors and binding conveyance in Clojure"
 :layout :post
 :toc true
 :tags  ["clojure" "executors" "binding conveyance" "testing"]}

Every now and then I run into a situation when I need to execute code in
multiple threads in a controlled way. A typical example for such a situation is
when I want to test my code for thread safety or performance under various
concurrency settings.

The JVM offers a convenient way for doing this: the
[`java.util.concurrent.Executors`](https://docs.oracle.com/en/java/javase/15/docs/api/java.base/java/util/concurrent/Executors.html)
class provides a number of useful predefined `ExecutorService` types with
different concurrency properties. However, using them in Clojure properly is not
entirely trivial, as in many cases dynamic variable bindings have to be taken
into account.

# A simple test case

To illustrate the point, let's imagine that we have a function that should work
right even when invoked from concurrent threads. As an (admittedly contrived)
example, the function `sut` below has the task of setting the value in the
`global` atom to the next higher value divisible by `modulus`.

```clojure
(def modulus 97)

(def global (atom 0))

(defn sut []
  (let [c @global]
    (Thread/sleep (inc (rand-int 20)))
    (let [m (mod c modulus)
          result (if (pos? m)
                   (swap! global + (- modulus m))
                   c)]
      (when (pos? (mod @global modulus))
        (case (rand-int 200)
          0 (throw (ex-info "error detected" {}))
          1 (Thread/sleep 60000)
          nil))
      result)))
```

This function is obviously not thread safe: reading and writing the global
variable is not done atomically. Sleeping between reading and writing the global
variable makes it very likely that in the presence of concurrent calls the end
value will not be divisible by `modulus`. The function also simulates some
common erroneous behaviour: whenever it sees that adjusting the global value
failed, with a small probability it throws an exception or just hangs for a long
time.

Ideally, we want our tests to explicitly report all of these problems, when the
state gets an invalid value, when the computation fails and when the computation
blocks.

# Testing from a single thread

We define a fixture that sets `global` to a random value between 0 and 9999
before a test is executed. The test itself makes two checks. First, if the
result of `sut` is divisible with `modulus` and second, if the result is a
natural integer.

```clojure
(test/use-fixtures :each (fn [f]
                           (reset! global (rand-int 10000))
                           (f)))

(defn sut-check []
  (is (zero? (mod (sut) modulus)))
  (is (nat-int? (sut))))

(deftest single-thread
  (sut-check))
```

The test `single-thread` always passes.

# Multi-threaded testing the naive way

The test below calls the same function doing the checks as the single threaded
one, but starts multiple concurrent calls.

```clojure
(deftest multi-thread-naive
  (let [threads 8, tasks (* 2 threads)
        executor (Executors/newFixedThreadPool threads)]
    (try
      (->> (repeatedly tasks #(.submit executor ^Callable sut-check))
           doall
           (map #(try (.get % 1 TimeUnit/SECONDS)
                      (catch TimeoutException _ ::timeout)))
           (every? true?)
           is)
      (finally
        (.shutdown executor)))))
```

It first creates an executor with a pool of 8 threads, then submits 16 tasks to
it. `repeatedly` delivers a lazy sequence, so the submissions are forced with
`doall`. This results in a list of futures and we get their values with a `map`
call. In case we do not get the result within a second, we return
`::timeout`. (We know that `sut-check` cannot deliver this value, so it
unambiguously identifies a timeout.) Then we check that the result of each task
is `true`, which is the value the `is` macro delivers when its assertion
holds. Finally, we shut down the executor.

The output of running this test depends on the test runner and the environment.
Running this test with `cognitect.test-runner` in a shell produces an output
like this:

```shell
$ clojure -X:run :vars '[com.github.bentomi.demo-test/multi-thread-naive]'

Running tests in #{"test"}

Testing com.github.bentomi.demo-test

FAIL in
FAIL in  () (demo_test.clj:49)
FAIL in

FAIL in
FAIL in() (demo_test.clj:49)
() (demo_test.clj:49)
() (demo_test.clj:49)
() (demo_test.clj:49)
FAIL in

FAIL in () (demo_test.clj:49)
 () (demo_test.clj:49)
expected: expected:(zero? (mod (sut) modulus))
expected: (zero? (mod (sut) modulus))
(zero? (mod (sut) modulus))
expected: expected:(zero? (mod (sut) modulus))
(zero? (mod (sut) modulus))
expected:expected:  (zero? (mod (sut) modulus))(zero? (mod (sut) modulus))

  actual:  actual:  (not (zero? 1))
  actual: (not (zero? 33))(not (zero? 65))

  actual: (not (zero? 66))
  actual: (not (zero? 2))
  actual:  actual:  (not (zero? 34))(not (zero? 34))


Ran 1 tests containing 1 assertions.
0 failures, 0 errors.
```

We can see some garbled error messages and the report at the end that test
passed with one assertion. This is because the assertion in `multi-thread-naive`
can only detect when `sut-check` takes longer than a second or when the second
call to `is` throws an exception. In the first case the corresponding value is
`::timeout`, in the second case it's `nil`. As the likelihood of these failures
is low, the test passes most of the time. In an IDE the output is often hidden,
and we can only see that test passes.

The assertions in `sut-check` are not taken into account in the report, because
their results are collected in a Ref stored in the
`clojure.test/*report-counters*` dynamic variable which is not seen in the
threads making the calls.

Fortunately, Clojure supports binding conveyance, that is, if you make a call
using `future`, the call will be executed in another thread but it will still
see the bindings existing in the current thread.

# Multi-threaded testing with binding conveyance

The functions `future` and `future-call` execute their arguments with the same
executor that is used with agents when their task is submitted with
`send-off`. As discussed in
[an earlier post](/posts-output/2017-06-04-message-sending-clojure-elixir/),
the function `set-agent-send-off-executor!` can be used to set our custom
executor for use by `send-off`, `future` and `future-call`. This means that, as
long as the code we are testing is not using any functions relying on this
executor, we can override it for the scope of the test. Unfortunately,
`set-agent-send-off-executor!` returns the executor we set, not the original
one, so we have to read `clojure.lang.Agent/soloExecutor` explicitly.

```clojure
(deftest multi-thread-conveying
  (let [threads 8, tasks (* 2 threads)
        executor (Executors/newFixedThreadPool threads)
        original-executor clojure.lang.Agent/soloExecutor]
    (set-agent-send-off-executor! executor)
    (try
      (->>
       (repeatedly tasks #(future-call sut-check))
       doall
       (map #(deref % 1000 ::timeout))
       (every? true?)
       is)
      (finally
        (set-agent-send-off-executor! original-executor)
        (.shutdown executor)))))
```

This version of the test differs from the naive version only in that it installs
our executor as the agent send off executor for the scope of the test and
instead of dealing with Java's Futures, it uses Clojure's `deref` to obtain the
result values.

Running this test produces an output like this:

```shell
$ clojure -X:run :vars '[com.github.bentomi.demo-test/multi-thread-conveying]'

Running tests in #{"test"}

Testing com.github.bentomi.demo-test

FAIL in (multi-thread-conveying) (demo_test.clj:29)

FAIL in (multi-thread-conveying) (demo_test.clj:29)
FAIL in
 (multi-thread-conveying) (demo_test.clj:29)
expected: (zero? (mod (sut) modulus))
expected: (zero? (mod (sut) modulus))
expected: (zero? (mod (sut) modulus))

FAIL in
FAIL in(multi-thread-conveying) (demo_test.clj:29)
(multi-thread-conveying) (demo_test.clj:29)
expected: (zero? (mod (sut) modulus))
expected: (zero? (mod (sut) modulus))
  actual: (not (zero? 70))
  actual:  actual:   actual:(not (zero? 88))
  (not (zero? 88))
(not (zero? 79))
  actual: (not (zero? 79))

FAIL in (multi-thread-conveying) (demo_test.clj:29)
expected: (zero? (mod (sut) modulus))
  actual: (not (zero? 70))

FAIL in (multi-thread-conveying) (demo_test.clj:29)
expected: (zero? (mod (sut) modulus))
  actual: (not (zero? 43))

Ran 1 tests containing 33 assertions.
7 failures, 0 errors.
```

Here we can see that there are 33 assertions not just one and, more importantly,
that there are seven failures. Now, even if we cannot see the output of the
tests, we can notice that they fail.

## Reducing the boilerplate

Since it's awkward and error prone setting up the executor like this, we better
extract the ceremony into a macro:

```clojure
(spec/fdef with-send-off-executor
  :args (spec/cat :binding (spec/spec (spec/cat :name simple-symbol?
                                                :executor any?))
                  :body (spec/+ any?)))

(defmacro with-send-off-executor
  "Creates an ExecutorService by calling `executor`, sets it for the scope
  of the form as executor for `send-off`, `future`, etc. and executes `body`.
  The executor service created is bound to `name` and shut down after the
  execution of `body`."
  [[name executor] & body]
  `(let [~name ~executor
         original-executor# clojure.lang.Agent/soloExecutor]
     (set-agent-send-off-executor! ~name)
     (try
       ~@body
       (finally
         (set-agent-send-off-executor! original-executor#)
         (.shutdown ~name)))))
```

`with-send-off-executor` is symmetrical to Clojure's `with-open` macro. It
allows us to name the executor created and manipulate it in the body of the
form. When the execution leaves the form, the executor is shut down.

Using this macro we can write the test such:

```clojure
(deftest multi-thread-conveying
  (let [threads 8, tasks (* 2 threads)]
    (with-send-off-executor [_executor (Executors/newFixedThreadPool threads)]
      (->> (repeatedly tasks #(future-call sut-check))
           doall
           (map #(deref % 1000 ::timeout))
           (every? true?)
           is))))
```

Have a look at the [source code](https://github.com/bentomi/conveyance) if you
want to play with it.
