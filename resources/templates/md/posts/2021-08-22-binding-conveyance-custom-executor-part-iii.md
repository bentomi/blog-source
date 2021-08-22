{:title "Custom executors and binding conveyance in Clojure, part III"
 :layout :post
 :toc false
 :tags  ["clojure" "executors" "binding conveyance" "testing"]}

In the
[previous post](/posts-output/2021-08-13-binding-conveyance-custom-executor-part-ii/),
I described how to set up dynamic variable bindings for arbitrary executors. I
also pointed out that this method has a slight performance penalty: the bindings
are set up and torn down before and after each submitted task, although we need
this only once for each thread in the pool.

In this part I show a way to get rid of this performance penalty whenever using
an executor accepting a thread factory. The following examples assume the
environment described in the
[first part](/posts-output/2021-08-09-binding-conveyance-custom-executor/) of
this series.

# A thread factory with dynamic binding

The first step is to define a function a thread factory creating threads with
the desired bindings already installed. The function below takes two optional
parameters, a thread factory producing the threads we want to modify and the
bindings to be installed. Both of these have straightforward default values: the
default thread factory from `java.util.concurrent.Executors` and the current
thread bindings.

```clojure
(defn binding-thread-factory
  "Returns a thread factory wrapping `base-factory` that creates threads having
  `bindings` installed."
  [& {:keys [base-factory bindings]
      :or {base-factory (Executors/defaultThreadFactory)
           bindings (get-thread-bindings)}}]
  (reify ThreadFactory
    (newThread [_this runnable]
      (.newThread base-factory #(with-bindings bindings (.run runnable))))))
```

In the last line of this code the `with-bindings` macro is used to set the
desired bindings for the runnable of the new thread.

This function can be used as shown below for creating a fixed thread pool
executor.

```clojure
(defn ^ExecutorService binding-fixed-thread-pool
  "Returns a fixed thread pool with `threads` number of threads using a binding
  thread pool created according to `factory-opts`.
  Also see: `binding-thread-factory`."
  [threads & factory-opts]
  (let [thread-factory (apply binding-thread-factory factory-opts)]
    (Executors/newFixedThreadPool threads thread-factory)))
```

# Using the executor in tests

Like before, it makes sense to define a macro for creating and shutting down the
executor:

```clojure
(spec/fdef with-executor
  :args (spec/cat :binding (spec/spec (spec/cat :name simple-symbol?
                                                :executor any?))
                  :body (spec/+ any?)))

(defmacro with-executor
  "Creates an ExecutorService by calling `executor` and executes `body`.
  The executor service created is bound to `name` and shut down after the
  execution of `body`."
  [[name executor] & body]
  `(let [~name ~executor]
     (try
       ~@body
       (finally
         (.shutdown ~name)))))
```

With this macro a multi-threaded test can be defined such:

```clojure
(deftest multi-thread-bind-once
  (let [threads 8, tasks (* 2 threads)]
    (with-executor [executor (binding-fixed-thread-pool threads)]
      (->> (repeatedly tasks #(.submit executor ^Callable sut-check))
           doall
           (map #(try (.get % 1 TimeUnit/SECONDS)
                      (catch TimeoutException _ ::timeout)))
           (every? true?)
           is))))
```

See the [source code](https://github.com/bentomi/conveyance) if you want to play
with it.
