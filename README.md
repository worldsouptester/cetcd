# cetcd

A Clojure wrapper for [etcd]. Uses [http-kit] to talk to `etcd`, so we get callbacks for free.

[![Build Status](https://circleci.com/gh/dwwoelfel/cetcd.png?circle-token=e7ee56b54be2ff8df039f4ea955f7e7111d91c08)](https://circleci.com/gh/dwwoelfel/cetcd)

## Installation

`cetcd` is available as a Maven artifact from [Clojars](https://clojars.org/cetcd).

[cetcd "0.2.2"]

## Changelog

### Version 0.2.2

- Adds support for prevValue and prevIndex to delete, implements compare-and-delete! (fixes #4). Thanks pjlegato!

### Version 0.2.1

- Update http-kit to make 307 redirects work (fixes #5). Thanks pjlegato!

### Version 0.2.0

- URL encode all keys

## Usage

### Set the value to a key

```clojure
user> (require '[cetcd.core :as etcd])
nil
user> (etcd/set-key! :a 1)
{:action "set",
 :node
 {:key "/:a",
  :modifiedIndex 10,
  :createdIndex 10,
  :value "1",
  :prevValue "1"}}
```

### Get the value of a key

```clojure
user> (etcd/get-key :a)
{:action "get",
 :node {:key "/:a", :modifiedIndex 10, :createdIndex 10, :value "1"}}
```

Let's get the actual value:

```clojure
user> (-> (etcd/get-key :a) :node :value)
"1"
```

Notice that `cetcd` didn't preserve the type of the key's value. That's a job that's left to the caller:

```clojure
user> (etcd/set-key! :clojure-key (pr-str 1))
{:action "set", :node {:key "/:clojure-key", :value "1", :modifiedIndex 14, :createdIndex 14}
user> (-> (etcd/get-key :clojure-key)
          :node
          :value
          (clojure.edn/read-string))
1

```

`cetcd` also doesn't do much to help you if the key doesn't exist:

```clojure
user> (etcd/get-key :b)
{:errorCode 100, :message "Key Not Found", :cause "/:b", :index 22}
```

But sane callers should be fine:
```clojure
user> (-> (etcd/get-key :b) :node :value) ;; return nil if key doesn't exist
nil
```

`cetcd` purposely doesn't return the value because directories require you to know about nodes
```clojure
user> (do
        (etcd/set-key! :foo/bar 1)
        (etcd/set-key! :foo/baz 2)
        (etcd/get-key :foo))
{:action "get",
 :node
 {:key "/:foo",
  :dir true,
  :nodes
  [{:key "/:foo/bar", :value "1", :modifiedIndex 26, :createdIndex 26}
   {:key "/:foo/baz",
    :value "2",
    :modifiedIndex 27,
    :createdIndex 27}],
  :modifiedIndex 24,
  :createdIndex 24}}
```

### Delete a key

```clojure
user> (etcd/delete-key! :a)
{:action "delete",
 :node
 {:key "/:a", :modifiedIndex 11, :createdIndex 10, :prevValue "1"}}
```


Everything that `etcd` does can be accomplished with the `set-key!`, `get-key`, and `delete-key!` functions above, by passing in the proper keyword args. There are also a few helper functions to keep things a bit cleaner.


### Compare and swap

```clojure
user> (etcd/compare-and-swap! :a 2 {:prevValue 1})
{:action "compareAndSwap", :node {:key "/:a", :prevValue "1", :value "2", :modifiedIndex 15, :createdIndex 13}}
```

You have to check manually if the condition failed:

```clojure
user> (etcd/compare-and-swap! :a 2 {:prevValue 1})
{:errorCode 101, :message "Test Failed", :cause "[1 != 2] [0 != 22]", :index 22}
```

### Compare and delete

```clojure
user> (etcd/set-key! :cad-key 2)
{:action "set", :node {:key "/:cad-key", :value "2", :modifiedIndex 47, :createdIndex 47}}
user> (etcd/compare-and-delete! :cad-key {:prevValue 2})
{:action "compareAndDelete", :node {:key "/:cad-key", :modifiedIndex 48, :createdIndex 47}, :prevNode {:key "/:cad-key", :value "2", :modifiedIndex 47, :createdIndex 47}}
```

You have to check manually if the condition failed:

```clojure
user> (etcd/set-key! :cad-key 2)
{:action "set", :node {:key "/:cad-key", :value "2", :modifiedIndex 49, :createdIndex 49}}
user> (etcd/compare-and-delete! :cad-key {:prevValue 1})
{:errorCode 101, :message "Compare failed", :cause "[1 != 2] [0 != 49]", :index 49}
```

### Wait for a value

```clojure
user> (future (println "new value is:" (-> (etcd/watch-key :a) :node :value)))
#<core$future_call$reify__6267@ddd23bc: :pending>
user> (etcd/set-key! :a 3)
new value is: 3
{:action "set", :node {:key "/:a", :prevValue "2", :value "3", :modifiedIndex 16, :createdIndex 16}}
```

If you provide a callback function, then it will return immediately with a promise:

```clojure
user> (def watchvalue (atom nil)) ;; give us a place store the resp in the callback
#'user/watchvalue
user> (etcd/watch-key :a :callback (fn [resp]
                                       (reset! watchvalue resp)))
#<core$promise$reify__6310@144d3f4b: :pending>
user> (etcd/set-key! :a 4)
{:action "set", :node {:key "/:a", :prevValue "3", :value "4", :modifiedIndex 20, :createdIndex 20}}
user> watchvalue
#<Atom@69bcc736: {:action "set", :node {:key "/:a", :prevValue "3", :value "4", :modifiedIndex 20, :createdIndex 20}}>
```

### Create a directory

```clojure
user> (etcd/create-dir! "foo/bar/baz")
{:action "set",
 :node
 {:key "/foo/bar/baz", :dir true, :modifiedIndex 37, :createdIndex 37}}
```

## License

Copyright © 2013

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.

[etcd]: https://github.com/coreos/etcd
[http-kit]: http://http-kit.org/
