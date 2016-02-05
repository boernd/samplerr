# samplerr

* is a [riemann](http://riemann.io/) plugin
* writes your metrics to [elasticsearch](http://elastic.co/products/elasticsearch)
* aggregates data using arbitrary clojure functions
* aggregates data in realtime into different round robin time-based elasticsearch indices
* manages your time based elasticsearch indices and evicts old data
* ensures your metric stays within storage boundaries

## what's before alpha?

**DISCLAIMER**

this code doesn't work yet, but it will hopefully soon!

## Installation

After cloning the repo, you can build the plugin using [leiningen](/technomancy/leiningen)

```
lein uberjar
```

This will create a plugin jar named `samplerr-x.y.z-SNAPSHOT-standalone.jar` which you can include into your *cjava lasspath*, *e.g.*:

```
java -cp /usr/lib/riemann/riemann.jar:/usr/lib/riemann/samplerr-0.1.1-SNAPSHOT-standalone-up.jar riemann.bin start /etc/riemann/riemann.config
```

On debian or redhat you could also add the classpath using the `EXTRA_CLASSPATH` variable available respectively in `/etc/default/riemann` or `/etc/sysconfig/riemann`.

## Synopsis

```clojure
(load-plugins)

(samplerr/periodically-expire 172800)
(let [rrdtool2016 (samplerr/bulk 
                    {:rra [{:step 20   :keep 86400    :index_ts :day}
                           {:step 600  :keep 2635200  :index_ts :month}
                           {:step 3600 :keep 31556736 :index_ts :year}]})]
  (streams
    (where (tagged "collectd")
      (by [:host :service]
        (async-queue! :samplerr {:queue-size 10000}
          (batch 10000 10 rrdtool2016))))))
```

