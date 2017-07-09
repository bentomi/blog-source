{:title "Sending Node.js readable streams to core.async channels respecting backpressure"
 :layout :post
 :toc true
 :tags  ["nodejs" "readable stream" "clojurescript" "core.async" "channel" "backpressure"]}

## The naive solution

The simplest solution (also suggested
by
[Stack Overflow](https://stackoverflow.com/questions/39291779/idiomatic-conversion-of-node-js-apis-to-clojurescript))
is the following:

```
(defn put-stream-unsafe! [rs ch]
  (.on rs "data" #(async/put! ch %))
  (.on rs "end" #(async/close! ch)))
```

The function `put-stream-unsafe!` takes a
Node.js
[stream.Readable](https://nodejs.org/api/stream.html#stream_class_stream_readable) instance
as `rs` and a [core.async](https://github.com/clojure/core.async)
channel as `ch` and puts the data chunks read onto the channel.

This works most of the time, except when the producer of the readable
stream is faster than the consumer of the channel.  This is what
happens in such a situation:

```
Error: Assert failed: No more than 1024 pending puts are allowed on a single channel. Consider using a windowed buffer.
(< (.-length puts) impl/MAX-QUEUE-SIZE)
```

The error says that we are trying to make the 1025th put on the
channel without the channel being read.  We also get the advice to use
a windowed buffer on the channel, but this is only a solution if we
don't mind losing data.  If we do mind losing data, then we have to
tell the producer to slow down enough so that the consumer can keep
up.  One way a consumer can inform the producer that it has to slow
down is to apply *backpressure*.

## Applying backpressure on readable streams

Node.js readable streams support backpressure via the `pause` and
`resume` methods.  When `pause` is called on a readable stream, it
will not call emit `'data'` events until `resume` is called.  On the
other side the core.async `put!` function has a three argument
version, where the last argument is a callback that's called when the
data we put arrives on the channel.  Data can arrive either in the
buffer of the channel (if there is free place there) or can be read by
the consumer.  Armed with these functions, we can implement the safe
version of the put stream function:

```
(defn put-stream! [rs ch]
  (.on rs "data" (fn [data]
                   (.pause rs)
                   (async/put! ch data #(.resume rs))))
  (.on rs "end" #(async/close! ch)))
```

When we get a chunk of data, we pause the stream and only resume it
when the data arrives on the channel.

## Test code

The following code uses the `/dev/zero` device which produces an
endless stream of zeros.  We use one of the functions above to send
the contents of such a stream to an unbuffered core.async channel.
You can experiment by changing the call to `put-stream!` in `-main` to
`put-stream-unsafe!`.


```
(ns readable-stream.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [cljs.nodejs :as nodejs]
            [cljs.core.async :as async :refer [<!]]))

(nodejs/enable-util-print!)

(def fs (nodejs/require "fs"))

(defn put-stream-unsafe! [rs ch]
  (.on rs "data" #(async/put! ch %))
  (.on rs "end" #(async/close! ch)))

(defn put-stream! [rs ch]
  (.on rs "data" (fn [data]
                   (.pause rs)
                   (async/put! ch data #(.resume rs))))
  (.on rs "end" #(async/close! ch)))

(defn -main [& args]
  (let [rs (.createReadStream fs "/dev/zero" #js {:encoding "binary"})
        ch (async/chan)]
    (go
      (dotimes [_ 5]
        (-> ch <! count println)))
    (put-stream! rs ch)))

(set! *main-cli-fn* -main)
```
