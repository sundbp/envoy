## Diplomatic rank

[source](https://en.wikipedia.org/wiki/Diplomatic_rank#Historical_ranks.2C_1815-1961):

> The rank of Envoy was short for "Envoy Extraordinary and Minister Plenipotentiary", and was more commonly known as Minister.[2] For example, the Envoy Extraordinary and Minister Plenipotentiary of the United States to the French Empire was known as the "United States Minister to France" and addressed as "Monsieur le Ministre."

- [How to play](#how-to-play)
- [Map to Consul](#map-to-consul)
- [Consul to Map](#consul-to-map)
- [Watch for key/value changes](#watch-for-keyvalue-changes)
- [Consul CRUD](#consul-crud)
  - [Adding to Consul](#adding-to-consul)
  - [Reading from Consul](#reading-from-consul)
  - [Deleting from Consul](#deleting-from-consul)
- [License](#license)

## How to play

In order to follow all the docs below, bring envoy in:

```clojure
$ boot repl
boot.user=> (require '[envoy.core :as envoy :refer [stop]])
nil
```

## Map to Consul

Since most Clojure configs are EDN maps, you can simply push the map to Consul with preserving the hierarchy:

```clojure
boot.user=> (def m {:hubble
                    {:store "spacecraft://tape"
                     :camera 
                      {:mode "color"}
                     :mission "Horsehead Nebula"}})

boot.user=> (envoy/map->consul "http://localhost:8500/v1/kv" m)
nil
```

done.

you should see Consul logs confirming it happened:

```bash
2016/11/02 02:04:32 [DEBUG] http: Request PUT /v1/kv/hubble/mission (144.536µs) from=127.0.0.1:60189
2016/11/02 02:04:32 [DEBUG] http: Request PUT /v1/kv/hubble/camera/mode (121.157µs) from=127.0.0.1:60189
2016/11/02 02:04:32 [DEBUG] http: Request PUT /v1/kv/hubble/store (94.573µs) from=127.0.0.1:60189
```

and a visual:

<p align="center"><img src="doc/img/map-to-consul.png"></p>

## Consul to Map

In case a Clojure map with config read from Consul is needed it is just `consul->map` away:

```clojure
boot.user=> (envoy/consul->map "http://localhost:8500/v1/kv/")
{:hubble
 {:camera {:mode "color"},
  :mission "Horsehead Nebula",
  :store "spacecraft://tape"}}
```

you may notice it comes directly from "the source" by looking at Consul logs:

```bash
2016/11/02 02:04:32 [DEBUG] http: Request GET /v1/kv/?recurse (76.386µs) from=127.0.0.1:54167
```

## Watch for key/value changes

Adding a watcher is simple: `envoy/watch-path path fun`

`fun` is going to be called with a new value each time the `path`'s value is changed.

```clojure
boot.user=> (def store-watcher (envoy/watch-path "http://localhost:8500/v1/kv/hubble/store"
                                                 #(println "watcher says:" %)))
```

creates a `envoy.core.Watcher` and echos back the current value:

```clojure
#'boot.user/store-watcher
watcher says: {:hubble/store spacecraft}
```

it is an `envoy.core.Watcher`:

```clojure
boot.user=> store-watcher
#object[envoy.core.Watcher 0x72a190f0 "envoy.core.Watcher@72a190f0"]
```

that would print to REPL, since that's the function provided `#(println "watcher says:" %)`, every time the key `hubble/store` changes.

let's change it to "Earth":
<p align="center"><img src="doc/img/store-update.png"></p>

once the "UPDATE" button is clicked REPL will notify us with a new value:

```clojure
watcher says: {:hubble/store Earth}
```

same thing if it's changed with `envoy/put`:

```clojure
boot.user=> (envoy/put "http://localhost:8500/v1/kv/hubble/store" "spacecraft tape")
watcher says: {:hubble/store spacecraft tape}
{:opts {:body "spacecraft tape", :method :put, :url "http://localhost:8500/v1/kv/hubble/store"}, :body "true", :headers {:content-length "4", :content-type "application/json", :date "Wed, 02 Nov 2016 03:22:41 GMT"}, :status 200}
```

`envoy.core.Watcher` is stoppable:

```clojure
boot.user=> (stop store-watcher)
true
"stopping" "http://localhost:8500/v1/kv/hubble/store" "watcher"
```
## Consul CRUD

### Adding to Consul

The map from above can be done manually by "puts" of course:

```clojure
boot.user=> (envoy/put "http://localhost:8500/v1/kv/hubble/mission" "Horsehead Nebula")
{:opts {:body "Horsehead Nebula", :method :put, :url "http://localhost:8500/v1/kv/hubble/mission"}, :body "true", :headers {:content-length "4", :content-type "application/json", :date "Wed, 02 Nov 2016 02:57:40 GMT"}, :status 200}

boot.user=> (envoy/put "http://localhost:8500/v1/kv/hubble/store" "spacecraft")
{:opts {:body "spacecraft", :method :put, :url "http://localhost:8500/v1/kv/hubble/store"}, :body "true", :headers {:content-length "4", :content-type "application/json", :date "Wed, 02 Nov 2016 02:58:13 GMT"}, :status 200}

boot.user=> (envoy/put "http://localhost:8500/v1/kv/hubble/camera/mode" "color")
{:opts {:body "color", :method :put, :url "http://localhost:8500/v1/kv/hubble/camera/mode"}, :body "true", :headers {:content-length "4", :content-type "application/json", :date "Wed, 02 Nov 2016 02:58:36 GMT"}, :status 200}
```

### Reading from Consul

```clojure
boot.user=> (envoy/get-all "http://localhost:8500/v1/kv/hubble")
{:hubble/camera/mode "color",
 :hubble/mission "Horsehead Nebula",
 :hubble/store "spacecraft://tape"}

boot.user=> (envoy/get-all "http://localhost:8500/v1/kv/hubble/store")
{:hubble/store "spacecraft"}
```

in case there is no need to convert keys to keywords, it can be disabled:

```clojure
boot.user=> (envoy/get-all "http://localhost:8500/v1/kv/" :keywordize? false)
{"hubble/camera/mode" "color",
 "hubble/mission" "Horsehead Nebula",
 "hubble/store" "spacecraft://tape"}
```

### Deleting from Consul

```clojure
boot.user=> (envoy/delete "http://localhost:8500/v1/kv/hubble/camera")
{:opts {:method :delete, :url "http://localhost:8500/v1/kv/hubble/camera?recurse"}, :body "true", :headers {:content-length "4", :content-type "application/json", :date "Wed, 02 Nov 2016 02:59:26 GMT"}, :status 200}

boot.user=> (envoy/get-all "http://localhost:8500/v1/kv/hubble")
{:hubble/ nil, :hubble/mission "Horsehead Nebula", :hubble/store "spacecraft"}
```

## License

Copyright © 2016 tolitius

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
