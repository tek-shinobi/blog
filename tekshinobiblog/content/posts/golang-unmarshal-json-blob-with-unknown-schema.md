---
title: "Golang: Unmarshal Json Blob With Unknown Schema"
date: 2021-06-11T22:07:45+03:00
draft: false 
---
There are two ways to handle this:

## Approach 1: Using `json.RawMessage`

`json.RawMessage`: represents a raw JSON value

It and can be used to delay parsing until the schema is known.

Here's an example of how to use `json.RawMessage`:
```go
type UnknownSchema struct {
    Data json.RawMessage `json:"data"`
}

func main() {
    jsonStr := `{"data":{"name":"John","age":30}}`

    var obj UnknownSchema
    err := json.Unmarshal([]byte(jsonStr), &obj)
    if err != nil {
        panic(err)
    }

    // The schema of the data field is still unknown at this point
    fmt.Println(string(obj.Data))

    // Now we can unmarshal the data field using the appropriate schema
    var data map[string]interface{}
    err = json.Unmarshal(obj.Data, &data)
    if err != nil {
        panic(err)
    }

    fmt.Println(data["name"]) // "John"
    fmt.Println(data["age"]) // 30
}
```

Here, we have a struct `UnknownSchema` with a field `Data` of type `json.RawMessage`. We can then unmarshal the JSON string into this struct.

After unmarshalling, the Data field will contain the raw JSON value, which we can print to the console. At this point, we don't know the schema of the data field.

We can then unmarshal the Data field into a map of type `map[string]interface{}`, which will allow us to access the individual fields of the JSON object.

> Note that using `map[string]interface{}` can be less type-safe than using a defined struct, but it allows us to handle unknown JSON schemas.


## Approach 2: using `json.Decoder`

```go
func main() {
    jsonStr := `{"name":"John","age":30}`

    dec := json.NewDecoder(strings.NewReader(jsonStr))

    var data map[string]interface{}
    err := dec.Decode(&data)
    if err != nil {
        panic(err)
    }

    fmt.Println(data["name"]) // "John"
    fmt.Println(data["age"]) // 30
}
```

Here, we create a new `json.Decoder` with a `strings.Reader` that contains our JSON string. We then decode the JSON data into a map of type `map[string]interface{}`, which allows us to access the individual fields of the JSON object.

Using `json.Decoder` can be more memory efficient than using `json.Unmarshal`, especially for large JSON strings, because it reads and decodes the JSON data in a stream-like fashion rather than loading the entire JSON string into memory at once. Additionally, `json.Decoder` can be used to decode JSON data from an `io.Reader` source, such as an HTTP response or a file, which can be useful in certain situations.