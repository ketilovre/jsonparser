# Alternative JSON parser for Go (so far fastest)

It does not require you to know the structure of the payload (eg. create structs), and allows accessing fields by providing the path to them. So far it is the fastest JSON parser for Go, and it is up to **9 times faster** then standard `encoding/json` package (depending on payload size and usage), **allocates almost no memory**. See benchmarks below.

## Rationale
Originally I made this for a project that relies on a lot of 3rd party APIs that can be unpredictable and complex.
I love simplicity and prefer to avoid external dependecies. `encoding/json` requires you to know exactly your data structures, or if you prefer to use `map[string]interface{}` instead, it will be very slow and hard to manage.
I investigated what's on the market and found that most libraries are just wrappers around `encoding/json`, the only package that had its own parser is `ffjson` (and it is awesome), but it still requires you to create data structures.
Let's be honest, JSON is not the hardest format to parse, so i wrote one that focuses on simplicity and performance.

## Example
For the given JSON our goal is to extract the user's full name, number of github followers and avatar.

```go
import "github.com/buger/jsonparser"

...

data := []byte(`{
  "person": {
    "name": {
      "first": "Leonid",
      "last": "Bugaev",
      "fullName": "Leonid Bugaev"
    },
    "github": {
      "handle": "buger",
      "followers": 109
    },
    "avatars": [
      { "url": "https://avatars1.githubusercontent.com/u/14009?v=3&s=460", "type": "thumbnail" }
    ]
  },
  "company": {
    "name": "Acme"
  }
}`)

// Extracting person variable for caching reasons
// Since we have to fetch more keys from it, and do not want parser to analyze whole record each time
person, _, _, _ := jsonparser.Get(data, "person")

// You can specify key path by providing arguments to Get function
jsonparser.Get(person, "name", "fullName")

// There is `GetNumber` and `GetBoolean` helpers if you exactly know key data type
jsonparser.GetNumber(person, "github", "followers")

// When you try to get object, it will return you []byte slice pointer to data containing it
// In `company` it will be `{"name": "Acme"}`
jsonparser.Get(data, "company")

// If the key doesn't exist it will throw an error
var size float64
if value, _, err := jsonparser.GetNumber(data, "company", "size"); err != nil {
  size = value
}

// Get always returns a byte sequence containing key value, if it is array, object or simple value
// You can use `ArrayEach` helper to iterate items
// Underneath it just calls `Get` until it can't find the next item
arr, _, _, _ := jsonparser.Get(person, "gravatar", "avatars")
jsonparser.ArrayEach(arr, func(value []byte, dataType int, offset int, err error) {
	fmt.Println(jsonparser.Get(value, "url"))
})
```

## Reference

Library API is really simple. You just need the `Get` method to perform any operation. The rest is just helpers around it.

You also can view API at [godoc.org](https://godoc.org/github.com/buger/jsonparser)


### **`Get`**
```
func Get(data []byte, keys ...string) (value []byte, dataType int, offset int, err error)
```
Receives data structure, and key path to extract value from.

Returns:
* `value` - Pointer to original data structure containing key value, or just empty slice if nothing found or error
* `dataType` - 	Can be: `NotExist`, `String`, `Number`, `Object`, `Array`, `Boolean` or `Null`
* `offset` - Offset from provided data structure where key value ends. Used mostly internally, for example for `ArrayEach` helper.
* `err` - If the key is not found or any other parsing issue, it should return error. If key not found it also sets `dataType` to `NotExist`

Accepts multiple keys to specify path to JSON value (in case of quering nested structures).
If no keys are provided it will try to extract the closest JSON value (simple ones or object/array), useful for reading streams or arrays, see `ArrayEach` implementation.

### **`GetBoolean`** and **`GetNumber`**
```
func GetBoolean(data []byte, keys ...string) (val bool, offset int, err error)

func GetNumber(data []byte, keys ...string) (val float64, offset int, err error)
```
If you know the key type, you can use the helpers above. Returns same arguments as `Get` except `dataType`.
If key data type do not match, it will return error.

### **`ArrayEach`**
```
func ArrayEach(data []byte, cb func(value []byte, dataType int, offset int, err error))
```
Needed for iterating arrays, accepts a callback function with the same return arguments as `Get`.
Expects to receive array data structure (you need to `Get` it first). See example above.
Underneath it just calls `Get` without arguments until it can't find next item.


## What makes it so fast?
* It does not rely on `encoding/json`, `reflection` or `interface{}`, the only real package dependency is `bytes`.
* Operates with JSON payload on byte level, providing you pointers to the original data structure: no memory allocation.
* No automatic type conversions, by default everything is a []byte, but it provides you value type, so you can convert by yourself (there is few helpers included).


## Benchmarks

There are 3 benchmark types, trying to simulate real-life usage for small, medium and large JSON payloads.
For each metric, the lower value is better. Values better than standard encoding/json marked as bold text.

Compared libraries:
* https://golang.org/pkg/encoding/json
* https://github.com/Jeffail/gabs
* https://github.com/bitly/go-simplejson
* https://github.com/antonholmquist/jason
* https://github.com/mreiferson/go-ujson
* https://github.com/ugorji/go/codec
* https://github.com/pquerna/ffjson
* https://github.com/buger/jsonparser

#### TLDR
If you want to skip next sections, the winner is `jsonparser` (obviously benchmarks are biased :smirk:).
It is 3-9 times faster then standard `encoding/json` package (depending on payload size and usage), and almost infinitely (literally) better in memory consumption because it operates with data on byte level, and provide direct slice pointers.
The few allocations you see in benchmarks happen because of type conversions.

`ffjson` comes in second place, and looks really amazing considering that it is almost drop-in replacement for `encoding/json`.


#### Small payload

Each test processes 190 bytes of http log as a JSON record.
It should read multiple fields.
https://github.com/buger/jsonparser/blob/master/benchmark/benchmark_small_payload_test.go

| Library | time/op | bytes/op | allocs/op |
| --- | --- | --- | --- | --- |
| encoding/json struct | 6173 | 880 | 18 |
| encoding/json interface{} | 7901 | 1521 | 38|
| Jeffail/gabs | 7836 | 1649 | 46 |
| bitly/go-simplejson | 8273 | 2241 | 36 |
| antonholmquist/jason | 20941 | 7237 | 101 |
| github.com/ugorji/go/codec | 7731 | 2176 | 31 |
| mreiferson/go-ujson | **5701** | **1409** | 37 |
| pquerna/ffjson | **3163** | **624** | **15** |
| buger/jsonparser | **714** | **4** | **2** |

Winners are ffjson and jsonparser, where jsonparser is 8.6x faster then encoding/json and 4.4x faster then ffjson.
If you look at memory allocation, jsonparser has no rivals, as it makes no data copy and operates with raw []byte structures and pointers to it.

#### Medium payload

Each test processes a 2.4kb JSON record (based on Clearbit API).
It should read multiple nested fields and 1 array.

https://github.com/buger/jsonparser/blob/master/benchmark/benchmark_medium_payload_test.go

| Library | time/op | bytes/op | allocs/op |
| --- | --- | --- | --- | --- |
| encoding/json struct | 53251 | 1336 | 29 |
| encoding/json interface{} | 60781 | 10627 | 215 |
| Jeffail/gabs | 71547 | 11202 | 235 |
| bitly/go-simplejson | 67865 | 17187 | 220 |
| antonholmquist/jason | 70964 | 19013 | 247 |
| github.com/ugorji/go/codec | 108198 | 6712 | 152 |
| mreiferson/go-ujson | **45554** | 11547 | 270 |
| pquerna/ffjson | **19634** | **856** | **20** |
| buger/jsonparser | **11442** | **18** | **2** |

The pattern that emerges is clear: the difference between ffjson and jsonparser in CPU usage is smaller, but the memory consumption difference is growing.
gabs, go-simplejson and jason are based on encoding/json and map[string]interface{} and actually only helpers for unstructured JSON, their performance correlate with `encoding/json interface{}`, and they will skip next round.
`go-ujson` while have its own parser, shows same performance as `encoding/json`, also skips next round. Same situation with `ugorji/go/codec`, but it showed unexpectedly bad performance for complex payloads.


#### Large payload

Each test processes a 24kb JSON record (based on Discourse API)
It should read 2 arrays, and for each item in array get a few fields.
Basically it means processing a full JSON file.

https://github.com/buger/jsonparser/blob/master/benchmark/benchmark_large_payload_test.go

| Library | time/op | bytes/op | allocs/op |
| --- | --- | --- | --- | --- |
| encoding/json struct | 602245 | 8273 | 307 |
| encoding/json interface{} | 941123 | 215433 | 3395 |
| pquerna/ffjson | **287151** | **7792** | **298** |
| buger/jsonparser | **193601** | **120** | **32** |

The same patterns as in the medium test appears. Both `ffjson` and `jsonparser` have their own parsing code, and does not depend on `encoding/json` or `interface{}`, thats one of the reasons why they are so fast.

## Questions and support

All bug-reports and suggestions should go though Github Issues.
If you have some private questions you can send them directly to me: leonsbox@gmail.com

## Contributing

1. Fork it
2. Create your feature branch (git checkout -b my-new-feature)
3. Commit your changes (git commit -am 'Added some feature')
4. Push to the branch (git push origin my-new-feature)
5. Create new Pull Request

## Development

All my development happens using Docker, and repo include some Make tasks to simplify development.

* `make build` - builds docker image, usually can be called only once
* `make test` - run tests
* `make fmt` - run go fmt
* `make bench` - run benchmarks (if you need to run only single benchmark modify `BENCHMARK` variable in make file)
* `make profile` - runs benchmark and generate 3 files-  `cpu.out`, `mem.mprof` and `benchmark.test` binary, which can be used for `go tool pprof`
* `make bash` - enter container (i use it for running `go tool pprof` above)
