#Purpose

A minimal approach (three functions) to support Server-Sent events from Kafka using a `ring` compliant web server.

The defaults can be tweaked by code or configuration.

#Default-based approach (one function)

```clojure
kafka->sse-ch [request topic-name]
```

The function returns a core.async channel that 
- maps from the data on a Kafka channel to the HTML5 EventSource format
- emits SSE comments every 5 seconds to maintain the connection

You can use that channel to stream data out with any web server that supports core.async channels.
 
I have provided an example using `Aleph` and `Compojure`.

#Message format

The default output of the channel complies to the HTML5 `EventSource` spec and has these semantics:

- id (item offset in kafka topic)
- event (the message key as a string)
- data (the message value as a string)

```
id: 556
event: customer
data: {"id" 745 "message" "Hello SSE"}

id: 557
event: product-campaign
data: {"id" 964 "message" "Hello SSE"}
```

The data in the examples is JSON but it could be any string.

Of course, to use these defaults, messages placed on the Kafka topic must conform to these semantics.

#Filtering

The channel is unfiltered by default but supports a request parameter `filter[event]` to filter responses by matching on event name:

```
http://server-name/sse?filter[event]=customer
```

Regular expressions are also supported in a comma separated list:

```
http://server-name/sse?filter[event]=customer,product.*
```


#Composition approach (two functions)

##Hand assembly

```clojure
kafka-consumer->sse-ch [consumer transducer]
```

This function takes a Kafka consumer and transducer to assemble how the data on the channel is processed.

###Kafka Consumer

```clojure
sse-consumer [topic offset]
sse-consumer [topic offset options]
sse-consumer [topic offset options brokers]
sse-consumer [topic offset options brokers marshallers]
```

The second function is to enable the safe construction of a Kafka consumer for SSE purposes.

I make this a clear distinction because Kafka has mature support for maintaining positions in the stream for event stream processors.

We do not need that degree of sophistication for SSE - we assume clients want to read from one of two places on the Kafka topic
- the latest position (the default)
- a specific offset to enable consumption of any missing records during a network outage

In the latter case, EventSource clients send the `Last-Event-Id` header.

Providing the Kafka offset as the event id keeps it simple.

This consumer enables you to provide or as much or as little config as you need.

###core.async transducer

```clojure
(kafka-consumer->sse-ch consumer (comp (filter #(name-matches? event-filter (.key %)))
                                           (map consumer-record->sse)))
```

This is the code used to create the default transducer. 

Of course you can reuse one of the provided helper functions (`name-matches` and `consumer-record->sse`) or provide your own.

#Operations
The table shows the supported environment variables and defaults.

(def ^:private brokers-from-env (if-let [u (env :sse-proxy-kafka-broker-url)]
                                  {"bootstrap.servers" u}))


| Environment Variable | Meaning | Default |
| ---------------------| ------- | --------|
| SSE_PROXY_KAFKA_BROKER_URL  | List of brokers | localhost:9092 |
| SSE_PROXY_POLL_TIMEOUT_MILLIS  | Max time in ms for each poll on Kafka | 100 |
| SSE_PROXY_BUFFER_SIZE          | Size the async.core channel buffer | 512 |
| SSE_PROXY_KEEP_ALIVE_MILLIS    | Interval in ms between emitting SSE comments | 5000 |


#Testing

working on it with embedded K / ZK

#Example (using Aleph and Compojure)

```clojure
(def ^:private TOPIC (config/env-or-default :sse-proxy-topic "simple-proxy-topic"))

(defn sse-handler-using-defaults
  "Stream SSE data from the Kafka topic"
  [request]
  (let [topic-name (get (:params request) "topic" TOPIC)
        ch (sse/kafka->sse-ch request topic-name)]
    {:status  200
     :headers {"Content-Type"  "text/event-stream;charset=UTF-8"
               "Cache-Control" "no-cache"}
     :body    (s/->source ch)}))


(def handler
  (params/wrap-params
    (compojure/routes
      (GET "/kafka-sse" [] sse-handler-using-defaults)
      (route/not-found "No such page."))))
```



#Other Kafka libraries for Clojure

Kafka consumers are complex and there are several general purpose Clojure libraries to provide access to all of those functions in idiomatic Clojure and from which I have stolen ideas and code.
 
I like and recommend Franzy https://github.com/ymilky/franzy
