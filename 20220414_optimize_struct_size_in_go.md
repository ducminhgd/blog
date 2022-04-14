# Optimize Struct size trong Go

Mình đọc [bài viết này](https://towardsdev.com/golang-writing-memory-efficient-and-cpu-optimized-go-structs-62fcef4dbfd0) mấy tuần trước, giờ mới có thời gian note lại. Mình note lại vì sợ đường dẫn kia sẽ có lúc không truy cập được, mình cũng sẽ note lại bằng tiếng Việt dự trên bài viết của tác giả.

Cho cái struct như sau:

```go
type TerraformResource struct {
  Cloud                string                       // 16 Bytes
  Name                 string                       // 16 Bytes
  HaveDSL              bool                         //  1 Bytes
  PluginVersion        string                       // 16 Bytes
  IsVersionControlled  bool                         //  1 Bytes
  TerraformVersion     string                       // 16 Bytes
  ModuleVersionMajor   int32                        //  4 Bytes
}
```

và đoạn chương trình

```go
package main

import "fmt"
import "unsafe"

type TerraformResource struct {
  Cloud                string                       // 16 Bytes
  Name                 string                       // 16 Bytes
  HaveDSL              bool                         //  1 Byte
  PluginVersion        string                       // 16 Bytes
  IsVersionControlled  bool                         //  1 Byte
  TerraformVersion     string                       // 16 Bytes
  ModuleVersionMajor   int32                        //  4 Bytes
}

func main() {
    var d TerraformResource
    d.Cloud = "aws"
    d.Name = "ec2"
    d.HaveDSL = true
    d.PluginVersion = "3.64"
    d.TerraformVersion = "1.1"
    d.ModuleVersionMajor = 1
    d.IsVersionControlled = true
    fmt.Println("==============================================================")
    fmt.Printf("Total Memory Usage StructType:d %T => [%d]\n", d, unsafe.Sizeof(d))
    fmt.Println("==============================================================")
    fmt.Printf("Cloud Field StructType:d.Cloud %T => [%d]\n", d.Cloud, unsafe.Sizeof(d.Cloud))
    fmt.Printf("Name Field StructType:d.Name %T => [%d]\n", d.Name, unsafe.Sizeof(d.Name))
    fmt.Printf("HaveDSL Field StructType:d.HaveDSL %T => [%d]\n", d.HaveDSL, unsafe.Sizeof(d.HaveDSL))
    fmt.Printf("PluginVersion Field StructType:d.PluginVersion %T => [%d]\n", d.PluginVersion, unsafe.Sizeof(d.PluginVersion))
    fmt.Printf("ModuleVersionMajor Field StructType:d.IsVersionControlled %T => [%d]\n", d.IsVersionControlled, unsafe.Sizeof(d.IsVersionControlled))
    fmt.Printf("TerraformVersion Field StructType:d.TerraformVersion %T => [%d]\n", d.TerraformVersion, unsafe.Sizeof(d.TerraformVersion))
    fmt.Printf("ModuleVersionMajor Field StructType:d.ModuleVersionMajor %T => [%d]\n", d.ModuleVersionMajor, unsafe.Sizeof(d.ModuleVersionMajor))  
}
```

Kết quả

```shell
==============================================================
Total Memory Usage StructType:d main.TerraformResource => [88]
==============================================================
Cloud Field StructType:d.Cloud string => [16]
Name Field StructType:d.Name string => [16]
HaveDSL Field StructType:d.HaveDSL bool => [1]
PluginVersion Field StructType:d.PluginVersion string => [16]
ModuleVersionMajor Field StructType:d.IsVersionControlled bool => [1]
TerraformVersion Field StructType:d.TerraformVersion string => [16]
ModuleVersionMajor Field StructType:d.ModuleVersionMajor int32 => [4]
```

Như ta thấy, toàn bộ struct là 88 bytes, trong khi nếu cộng lại theo từng phần tử thì chỉ có 70 bytes.

Lý do: khi khai báo vùng nhớ cho một struct thì các thuộc tính được đặt liên tiếp nhau và chiếm từng block của vùng nhớ, mỗi block bao gồm 8 bytes -> do đó, nếu sử dụng 1 byte thì vẫn khai báo một block và  block đó còn 7 bytes trống nhưng không sử dụng được.

Thay đổi lại

```go
type TerraformResource struct {
  Cloud                string                       // 16 Bytes
  Name                 string                       // 16 Bytes
  PluginVersion        string                       // 16 Bytes
  TerraformVersion     string                       // 16 Bytes
  ModuleVersionMajor   int32                        //  4 Bytes
  HaveDSL              bool                         //  1 Byte
  IsVersionControlled  bool                         //  1 Byte
}
```

Sẽ thấy sự khác biệt