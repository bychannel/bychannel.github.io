# go反序列化proto3协议导致零值字段不显示


在我们这个项目中，数据落库的结构是用proto协议来定义的，proto协议生成的结构体中，json tag都是默认带有omitempty，如果直接转成json会导致零值字段不显示，在实际展示中会有很多问题，可以使用如下方法解决

示例代码

```go
package main

import (
    "./pb"
    "bytes"
    "encoding/json"
    "fmt"
    "github.com/golang/protobuf/jsonpb"
    "github.com/golang/protobuf/proto"
)

func main() {
    var i int32 = 0
    test(i)
}

func test(t int32) {
    d := &pb.FrameD6 {
        Datetimehour: "2020033014",
        Recordcnt:    t,
    }
    str,_ := json.Marshal(d)
    s := TransProtoToJson(d)
    fmt.Printf("@@@--incorrect JSON---> %+v \n",string(str))
    fmt.Printf("@@@--correct JSON---> %+v \n",s)
}

func TransProtoToJson (pb proto.Message) string{
    var pbMarshaler jsonpb.Marshaler
    pbMarshaler = jsonpb.Marshaler{
        EmitDefaults: true,
        OrigName:     true,
        EnumsAsInts:  true,
    }
    _buffer := new(bytes.Buffer)
    _ = pbMarshaler.Marshal(_buffer, pb)
    return string(_buffer.Bytes())
}
```

运行结果，输出是满足我们需求的

```scss
@@@--incorrect JSON---> {"datetimehour":"2020033014"} 
@@@--correct JSON---> {"recordid":"","laneid":"","programver":"","datetimehour":"2020033014","recordcnt":0,"moneycnt":0,"companyid":"","parkid":""} 
```

不过，我们项目使用的是protobuf的v1版本的包，是提供有 `jsonpb.Marshaler` 这个结构支持自定义设置，如果使用的是v2版本的包，相关的结构已经被废弃了，以上方法不适用。

