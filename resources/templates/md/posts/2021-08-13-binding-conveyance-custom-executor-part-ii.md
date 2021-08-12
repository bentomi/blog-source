{:title "Custom executors and binding conveyance in Clojure, part II"
 :layout :post
 :toc false
 :tags  ["clojure" "executors" "binding conveyance" "testing"]}

In the
[previous post](/posts-output/2021-08-09-binding-conveyance-custom-executor/),
I described how the agents' send-off pool can be overridden to ensure binding
conveyance for calls running on a thread different from that of the caller.

This method has good performance but comes with limitations: it obviously
interferes with agents getting tasks with `send-off` and can cause problems if
we want to run tasks in the thread pool that should not see the bindings in the
caller's environment. Also, it relies on internal code.

# Using the public API

If performance is not an issue or the limitations are unacceptable, it is better
to use the public `bound-fn` macro or `bound-fn*` functions. These return a
function that installs the bindings of their caller before executing their
arguments and clean them up afterwards. In other words, the bindings are pushed
and popped every time the function is executed.

With a small change (see line number 3 below) we can fix the
`multi-thread-naive` test:

```clojure
(deftest multi-thread-bound-fn
  (let [threads 8, tasks (* 2 threads)
        ^Callable sut-check (bound-fn* sut-check)
        executor (Executors/newFixedThreadPool threads)]
    (try
      (->> (repeatedly tasks #(.submit executor sut-check))
           doall
           (map #(try (.get % 1 TimeUnit/SECONDS)
                      (catch TimeoutException _ ::timeout)))
           (every? true?)
           is)
      (finally
        (.shutdown executor)))))
```

The `sut-check` function is replaced with another function installing the
bindings by calling `bound-fn*`.

You can have a look at the [source code](https://github.com/bentomi/conveyance)
if you want to play with it.
