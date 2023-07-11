---
title: "Golang Testing: Creating Large Files"
date: 2023-07-11T15:17:34+03:00
draft: false 
categories: ["golang" ]
tags: ["golang", "testing"]
---

Needed a quick hack to generate a known file size, to test that file upload limit logic works in an endpoint

Another option is to just generate a temporary file, using nearly the same process. This approach is fine but can slow down the test since file needs to be generated each time the test is run and beyond a certain size, this file generation time can become non-trivial.
```go
package main

import (
	"encoding/json"
	"fmt"
	"math/rand"
	"os"
	"time"
)

type LargeData struct {
	Data []byte
}

const desiredSize = 12 * 1024 * 1024 // 12MB

func main() {
	rand.Seed(time.Now().UnixNano())

	data := generateRandomData(desiredSize)

	largeData := LargeData{
		Data: data,
	}

	file, err := os.Create("large_data.json")
	if err != nil {
		fmt.Println("Failed to create file:", err)
		return
	}
	defer file.Close()

	encoder := json.NewEncoder(file)
	encoder.SetIndent("", "  ") 

	err = encoder.Encode(largeData)
	if err != nil {
		fmt.Println("Failed to encode JSON:", err)
		return
	}

	fmt.Println("Large data JSON created successfully.")
}

func generateRandomData(size int) []byte {
	data := make([]byte, size)
	_, _ = rand.Read(data)
	return data
}

```
