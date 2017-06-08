# TODO

##

21:49 < aphyr> righto
21:49 < aphyr> yeah, so I'd say for stuff like sum, for instance
21:50 < aphyr> (defn sum [interval & cs] (let [count (atom 0)] (fn stream [event] (swap! state + (:metric event)) ...))
21:50 < aphyr> same pattern as rate, sreduce itself, etc
21:50 < aphyr> close over an atom with your stream fn
21:51 < aphyr> https://github.com/riemann/riemann/blob/master/src/riemann/streams.clj#L220-L223
21:51 < aphyr> https://github.com/riemann/riemann/blob/master/src/riemann/streams.clj#L263-L274
21:51 < aphyr> hopefully a straightforward pattern to apply to your use case

## aggregation

* (fixed-offset-time-window) is too memory intensive
  for min/max/avg we don't need to store all values
→ (samplerr/scamm
    [interval & children]
    "accumulates events during fixed interval and emits exactly five events: sum, count, average, min and max
     it appends "/cfunc/interval" to service where cfunc is one of min, max, avg and count"
     (…)

### POC for min/max/avg/sum/count

* probably need to use (streams/sreduce)

```clojure
(let [index (index)]
  (streams
    (default :ttl 60
      (where metric
        (by [:host :service]
          (sreduce (fn [acc event] (if (or (nil? acc) (expired? acc)) event (if (>= (:metric event) (:metric acc)) event acc)))
            (smap #(assoc % :service (str (:service %) " max"))
              (coalesce 60
                (smap #(first %) index))))
          (sreduce (fn [acc event] (if (or (nil? acc) (expired? acc)) event (assoc event :metric (+ (:metric acc) (:metric event)))))
            (smap #(assoc % :service (str (:service %) " sum"))
              (coalesce 60
                (smap #(first %) index))))
          (sreduce (fn [acc event] (if (or (nil? acc) (expired? acc)) (assoc event :metric 1) (assoc event :metric (+ 1 (:metric acc))))) {:metric 0}
            (smap #(assoc % :service (str (:service %) " count"))
              (coalesce 60
                (smap #(first %) index))))
          (sreduce (fn [acc event] (if (or (nil? acc) (expired? acc)) (assoc event :sum (:metric event) :count 1) (assoc event :sum (+ (:sum acc) (:metric event)) :count (+ 1 (:count acc))))) nil
            (smap #(assoc % :service (str (:service %) " avg") :metric (/ (:sum %) (:count %)))
              (coalesce 60
                (smap #(first %) index))))
          (sreduce (fn [acc event] (if (or (nil? acc) (expired? acc)) event (if (<= (:metric event) (:metric acc)) event acc)))
            (smap #(assoc % :service (str (:service %) " min"))
              (coalesce 60
                (smap #(first %) index)))))))))
```

## aliasing

* use filtered aliases in ES to avoid having duplicate entries e.g. for daily indices between 2016.04 and 2016.04.23 when search query spans multiple days
* sort retention-policies
* verify overlap between periods

* (samplerr/periodically-shift <periodicity> <index-prefix> <alias-prefix> <retention-policies>)

* (samplerr/periodically-shift 86400 samplerr-i samplerr-a [{:es_index "YYYY.MM.DD" :keep 172800}
                                                            {:es_index "YYYY.MM"    :keep 5270400}
                                                            {:es_index "YYYY"}])
* list indices

(list-indices prefix)

(require '[clojurewerkz.elastisch.rest.index :as esri])
(def elastic (samplerr/connect "http://localhost:9200"))

(keys (esri/get-aliases elastic "samplerr-*"))

(def
