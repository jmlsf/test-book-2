Play with specter in a repl:

```
$ boot -d com.rpl/specter repl
boot.user=> (use 'com.rpl.specter)
nil
```

Punch keys into map:

```
boot.user=> (def x [{:a {}} {:a {:b {:c -13}}}])
#'boot.user/x
boot.user=> (setval [ALL (keypath :a :b :c)] 42 x)
[{:a {:b {:c 42}}} {:a {:b {:c 42}}}]
```

Can't punch into a val that is a non-map structure:

```
boot.user=> (def x [{:a "something"} {:a {:b {:c -13}}}])
#'boot.user/x
boot.user=> (transform [ALL (keypath :a :b :c)] (fn [elt] 42) x)
java.lang.ClassCastException: java.lang.String cannot be cast to clojure.lang.Associative
```

Transform two things at once \(helpful for atoms with watchers\):

```
boot.user=> (defn by-id [id] (fn [elt] (= (:id elt) id)))
#'boot.user/by-id
boot.user=> (def data [{:id 1 :name "one"} {:id 2 :name "two"}])
#'boot.user/data
boot.user=> (defn by-id [id] (fn [elt] (= (:id elt) id)))
#'boot.user/by-id
boot.user=>  (multi-transform [ALL (multi-path [(by-id 1) (terminal-val "ONE")]
       #_=>                                    [(by-id 2) (terminal-val "TWO")])]
       #_=>                   data)
["ONE" "TWO"]
```



