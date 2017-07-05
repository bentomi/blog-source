{:title "Message sending with Clojure agents, core.async and Elixir"
 :layout :post
 :toc true
 :tags  ["clojure" "agents" "core.async" "elixir" "message send"
         "benchmark" "process scheduling"]}

## The "messages sent through a chain" problem

This is an often used toy problem to benchmark the message passing
overhead in asynchronous message passing systems.  We create M number
of processes/agents in a chain and send N messages through the whole
chain.

I looked at three different implementations of the benchmark, an
Elixir version serving as the baseline and two Clojure versions, one
with agents and one with core.async.  Here are the basic versions of
each.

## Elixir solution

```
defmodule Chain do
  def relay(next_pid) do
    receive do
      message ->
        send next_pid, message
        relay(next_pid)
    end
  end

  def create_senders(m) do
    Enum.reduce 1..m, self(), fn (_, prev) -> spawn(Chain, :relay, [prev]) end
  end

  def test(m, n) do
    message = 0
    start = create_senders(m)
    Enum.each 1..n, fn (_) -> send start, message end
    Enum.each 1..n, fn (_) -> receive do x -> x end end
  end

  def run(m, n) do
    IO.puts inspect :timer.tc(Chain, :test, [m, n])
  end
end
```

The `test` function creates `m` Erlang processes each running in a
loop receiving messages and sending them on to the next process (the
last process receiving the message is the main process itself).  Then
it asynchronously sends `n` messages to the `start` process, and
finally it waits for all `n` messages to arrive back.

The test with 10000 processes sending around 10000 messages can be
executed by running the following command:

```
elixir --erl "+P 10000" -r chain.exs -e "Chain.run(10000, 10000)"
```

On my maching it prints `{5877968, :ok}`, meaining that everything
went well and the test took about 5.9 seconds.  During this time, all
CPU cores had 100% load.

## Clojure agents solution

```
(ns chain.agents
  (:import [java.util.concurrent SynchronousQueue]))

(defn relay [s m]
  (if (instance? clojure.lang.Agent s)
    (send s relay m)
    (.put ^SynchronousQueue s m))
  s)

(defn create-senders [m start]
  (reduce (fn [next _] (agent next)) start (range (dec m))))

(defn run [m n]
  (let [message 0
        q (SynchronousQueue.)
        start (create-senders m (agent q))]
    (dotimes [_ n]
      (send start relay message))
    (dotimes [_ n]
      (.take q))))

(defn -main [m n]
  (time (run (Integer/parseInt m) (Integer/parseInt n)))
  (shutdown-agents))
```

Here we create `m` agents linked to each other, except that the first
agent gets a queue as its state.  Then, just like in the Elixir
version, `n` messages are sent asynchronously to the start agent and
finally the main thread takes the messages from the queue.

After running `lein uberjar`, the program can be executed with
```
java -cp target/chain-standalone.jar clojure.main -m chain.agents 10000 10000
```
and prints `"Elapsed time: 89672.860248 msecs"`.  The first thing that
leaps to the eye is that this is about 15 times slower than the
Elixir version.  The second thing is that the CPU usage is about 20%
on all cores.  The latter suggest, that we might have some problems
with scheduling the work, so let's tune that a bit.

### Tuning agent action scheduling

The `send` function dispatches actions to the agents using a Java
fixed thread pool executor with pool size set to the number of
available (logical) processors plus two.  This default executor can be
overwritten by the `set-agent-send-executor!` function.  By changing
`-main` we can experiment with various pool sizes:

```
(ns chain.agents
  (:import [java.util.concurrent SynchronousQueue Executors]))

;; functions before -main unchanged

(defn -main [p m n]
  (set-agent-send-executor! (Executors/newFixedThreadPool (Integer/parseInt p)))
  (time (run (Integer/parseInt m) (Integer/parseInt n)))
  (shutdown-agents))
```

It turns out, that on my machine with eight logical processors the
program runs fastest with three threads in the thread pool:
`"Elapsed time: 61878.63678 msecs"`.  This clearly indicates that more
threads just hinder each other in doing useful work.

It is a bit annoying that we cannot use all processors, but
fortunately, since Java 1.8 the concurrency framework provides a work
stealing pool which works in a way very similar to the Erlang
scheduler.

```
;; everything before -main unchanged

(defn -main [m n]
  (set-agent-send-executor! (Executors/newWorkStealingPool))
  (time (run (Integer/parseInt m) (Integer/parseInt n)))
  (shutdown-agents))
```

This version prints `"Elapsed time: 35880.203266 msecs"` and produces
100% CPU usage.  It seems, scheduling is not the bottleneck anymore.
How does the work stealing pool do this?

The main difference between the work stealing and the fixed thread
pool is, that the latter has a single queue of tasks from which the
worker threads can take tasks, and where new tasks can be put.  At
high levels of parallelism this results in contention.  The work
stealing pool uses several queues which are dedicated to threads and
when a thread produces work, it is put the queue of thread.  Only when
the queue of a thread is empty does the thread go to see if it can
steal tasks from the queue of another thread.  This means, that as
long as the threads produce enough work for themselves (which the
agents in our example do), there will be no contention and no waiting
at all.

This is all very nice, but the Clojure agents are still about seven
times slower than the Elixir/Erlang processes.  Looking at the hot
spots in Java VisualVM reveals that the function
`push-thread-bindings` used for binding the dynamic `*agent*` variable
to the agent we are sending work to is responsible for about 20% of
the runtime.  Let's have a look at a solution that doesn't have this
overhead.

## Clojure core.async solution

In this solution we use core.async go blocks to represent relays.

```
(ns chain.async
  (:require [clojure.core.async :as async :refer [<! >! <!!]]))

(defn relay [in out]
  (async/go-loop []
    (>! out (<! in))
    (recur)))

(defn create-senders [m start]
  (reduce (fn [in _] (let [out (async/chan)] (relay in out) out))
          start (range m)))

(defn run [m n]
  (let [message 0
        start (async/chan)
        end (create-senders m start)]
    (async/go
      (dotimes [_ n]
        (>! start message)))
    (dotimes [_ n]
      (<!! end))))

(defn -main [m n]
  (time (run (Integer/parseInt m) (Integer/parseInt n))))
```

The structure of the program is the same as before, but here we have
to generate channels to be able to communicate between processes.
Executing this program with
```
java -cp target/chain-standalone.jar clojure.main -m chain.async 10000 10000
```
we get `Elapsed time: 81157.570264 msecs"` and 40% CPU usage on all
cores.

We seem to have a scheduling problem again.  But before looking for
ways to override the executor again (which is not supported out of the
box, see [ASYNC-94](https://dev.clojure.org/jira/browse/ASYNC-94))
let's compare this solution and the Elixir one.

Both solutions create processes for relaying the messages, but only
the Elixir solution is truly asynchronous.  In Elixr, each process has
a mailbox where the messages sent to it land and the sender doesn't
wait until the addressee pulls the message.  In the core.async
solution the processes have to synchronise, as the channels have no
buffers.  Let's see what happens if we decouple the processes a bit.

### Tuning core.async buffering

```
(defn create-senders [m start]
  (reduce (fn [in _] (let [out (async/chan 1000)] (relay in out) out))
          start (range m)))

(defn run [m n]
  (let [message 0
        start (async/chan 1000)
        end (create-senders m start)]
    (async/go
      (dotimes [_ n]
        (>! start message)))
    (dotimes [_ n]
      (<!! end))))
```

Now a go block can send 1000 messages without waiting for the other
end to handle them.  When executed, the program prints
`Elapsed time: 7157.570264 msecs"` and the CPU usage goes up to 100%
on all cores.  This is comparable to the Elixir version. Also, the
VisualVM sampler shows that the time is spent in the put and take
operations, which is as expected.

## Bigger messages

One aspect of this benchmark that's quite artificial is that the
messages are small integers.  Let's see what happens if instead of `0`
we send around a list of twenty integers.  This is how the Elixir
version looks like:

```
  def test(m, n) do
    message = Enum.to_list 1..20 # <- this is the only change
    start = create_senders(m)
    Enum.each 1..n, fn (_) -> send start, message end
    Enum.each 1..n, fn (_) -> receive do x -> x end end
  end
```

And this is the equivalent change for the core.async version:

```
(defn run [m n]
  (let [message (into () (range 20)) ; <- this is the only change
        start (async/chan 1000)
        end (create-senders m start)]
    (async/go
      (dotimes [_ n]
        (>! start message)))
    (dotimes [_ n]
      (<!! end))))
```

This change more than doubles the runtime of the Elixir version:
`{16641097, :ok}`, while the runtime of the Clojure versions remains
roughly unchanged: `"Elapsed time: 9755.010961 msecs"`. This is
because Erlang processes share nothing, and the message is copied each
time it is sent.  In Clojure there is a global heap, so only a
refernce to the list has to be sent around.  Why does the Erlang
virtual machine do this copying?  After all, Erlang lists are
immutable so sharing them should not be dangerous.  The reason is that
not sharing data enables the Erlang VM to maintain separate heaps for
the individual processes which means these heaps stay small and can be
garbage collected separately.  This in turn results in much shorter
and more predictable GC related latencies.

## Conclusion

I compared asynchronous message passing performance of Elixir and
Clojure, in Clojure I looked at both agents and core.async.  I
discussed issues related to scheduling, buffering and garbage
collection.  The following points seem to be the most important take
aways:

1. The Erlang process scheduler and message passing system is an
   impressive piece of work, and offers excellent out of the box
   experience.
1. In Clojure, agents are not the way to go if message passing
   performance is vital.
1. Clojure's core.async library offers simiar message passing
   performance as Erlang, especially if the messages are bigger than
   just a couple of small numbers and buffering can be tweaked.
1. Erlang's GC probably causes smaller latencies than the default JVM
   garbage collector.

Feel free to experiment with
the [code](https://github.com/bentomi/message-passing-test) yourself.
