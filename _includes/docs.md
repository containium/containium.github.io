### Running the Reactor on your machine

Until we release a standalone download you can simply clone the [git repository](https://github.com/containium/containium) and `lein run`.
The full Reactor requires a running Apache Zookeeper server for the Kafka system.


### Production ready Reactor (well, 0.1.0-SNAPSHOT)

Our Containium Reactor runs in production on <a href="https://smartos.org">SmartOS</a> as an SMF service. We need some time to provide a nice example repository. Contact us if you're interested.

To build a release:

```bash
lein do with-profile +aot compile, jar, libdir
mv target/containium*.jar lib/
```

To run it:

```bash
java -d64 -Djava.awt.headless=true -Dfile.encoding=UTF-8 -javaagent:jamm-0.2.5.jar '-XX:OnOutOfMemoryError=kill -9 %p' -cp /var/lib/containium/config:lib/* -server containium.core
```

### Standalone Application

project.clj

```clj
(defproject my-project-using-containium "0.1.0-SNAPSHOT"
  :dependencies [
    [containium "0.1.0-SNAPSHOT" :exclusions [boxure/clojure]]]
  :main my-project.main/main)

```


main.clj

```clj
(ns my-project.main
  (:require [containium.standalone :as standalone]))

(defn start [systems conf & [routes]]
  (println "Starting with config:\n" (print-str conf)))

(defn stop [start-result]
  (printlin "Stopping"))

(defn app [request]
  {:status 200 :body "Hello world!"})

(defn main []
  (standalone/run
    ;; Define config for all the Systems you want here
    {:http-kit {:port 8080} :repl {:port 2121}}
    ;; The "deployment descriptor" of this app
    {:start start :stop stop :ring {:handler 'my-project/app}}))
```

Then just `lein run` and see your app start up.


For a more detailed example, read: [Using Containium as a library](https://github.com/containium/containium#using-containium-as-a-library).