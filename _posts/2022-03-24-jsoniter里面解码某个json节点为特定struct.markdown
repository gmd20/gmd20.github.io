```go
package main

import (
	jsoniter "github.com/json-iterator/go"
)

type Info map[string]string

func main() {
	input := []byte(`{"type":"test","desc":"test","info":[{"a": "dd", "b":"ddd"}]}`)
	m := map[string]interface{}{}
	var info []GuestInfo

	var json = jsoniter.ConfigCompatibleWithStandardLibrary
	json.Unmarshal(input, &m)

                node := json.Get(input, "info")
                json.UnmarshalFromString(node.ToString(), &info)


                info[0]["a"] = "abcd"
	info[0]["xx"] = "456"
	if len(info) == 0 {
		delete(m, "info")
	} else {
		m["info"] = info
	}

	output, _ := json.Marshal(m)
	println(string(output))
}
```
