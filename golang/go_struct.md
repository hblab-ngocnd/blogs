# Định nghĩa cấu trúc tiết kiệm bộ nhớ và tối ưu hoá cho CPU trong go

## [Source](https://towardsdev.com/golang-writing-memory-efficient-and-cpu-optimized-go-structs-62fcef4dbfd0)

## Định nghĩa cấu trúc

Cấu trúc là 1 kiểu dữ liệu tập hợp của các field, hiệu quả trong việc gom data liên quan thành 1 record.
Điều này cho phép tất cả dữ liệu liên quan đến một thực thể được gói gọn trong một định nghĩa lightweight, 
các hành vi có thể được thực hiện bằng cách xác định các chức năng trên kiểu cấu trúc.

Trong bài viết này sẽ giải thích cách định nghĩa cấu trúc hiệu quả về sử dụng memory và `CPU Cycles`

Chúng ta hãy xem xét cấu trúc này bên dưới, định nghĩa về `terraform` resource cho một số use-case:

```go
type TerraformResource struct {
  Cloud                string                       // 16 bytes
  Name                 string                       // 16 bytes
  HaveDSL              bool                         //  1 byte
  PluginVersion        string                       // 16 bytes
  IsVersionControlled  bool                         //  1 byte
  TerraformVersion     string                       // 16 bytes
  ModuleVersionMajor   int32                        //  4 bytes
}
```
Hãy cùng kiểm tra xem sẽ mất bao nhiêu dung lượng bộ nhớ cần để cấp phát cho mỗi cấu trúc `TerraformResource` sử dụng đoạn mã sau

```go
package main

import "fmt"
import "unsafe"

type TerraformResource struct {
  Cloud                string                       // 16 bytes
  Name                 string                       // 16 bytes
  HaveDSL              bool                         //  1 byte
  PluginVersion        string                       // 16 bytes
  IsVersionControlled  bool                         //  1 byte
  TerraformVersion     string                       // 16 bytes
  ModuleVersionMajor   int32                        //  4 bytes
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

[Link execute](https://go.dev/play/p/KdDWNPr_7dW)


**Output**

```cmd
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

Tổng dung lượng bộ nhớ cần cấp phát cho struct `TerraformResource` là ***88*** bytes.
Hình dưới đây sẽ mô tả cách mà bộ nhớ được cấp phát

![images](https://miro.medium.com/max/1400/1*WgUjXTDLONCXzU5by--Log.jpeg)

Tuy nhiên hãy nhìn lại 1 chút ***16 + 16 + 1 + 16 + 1 + 16 + 4 = 70***, vậy ***18 bytes*** còn lại phát sinh từ đâu?

Khi nói đến `memory allocation` cho kiểu structs, chúng luôn luôn được cấp phát liền kề, liên tục bởi các `byte-aligned blocks` của bộ nhớ, mỗi field sẽ được cấp phát và lưu trữ theo thứ tự mà nó được định nghĩa bên trong cấp trúc. 
Khái niệm `byte-alignment` trong ngữ cảnh này có nghĩa là các khối bộ nhớ liền kề, liên tục được căn chỉnh ở các block bằng với `word size` của nền tảng (trong trường hợp này mỗi block là `8 bytes`).

![images](https://miro.medium.com/max/1400/1*1afhpdMUaIi3IE4y5qKf1A.jpeg)

Chúng ta có thể thấy rõ rằng `TerraformResource.HaveDSL` , `TerraformResource.isVersionControlled` và `TerraformResource.ModuleVersionMajor` chỉ được chiếm dữ tương ứng `1 Byte`, `1Byte` và `4 Bytes`. 
Phần còn lại của không gian được lấp đầy bằng `empty pad bytes`.

>Allocation bytes =16 bytes+16 bytes+ 1 bytes+ 16 bytes+ 1 bytes+ 16 bytes+ 4 bytes = 70 bytes
>
>Empty Pad bytes = 7 bytes + 7 bytes + 4 bytes = 18 bytes
>
>Total bytes = Allocation bytes + Empty Pad bytes = 70 bytes + 18 bytes = 88 bytes

Vậy làm thế nào để sửa nó. Với `data structure alignment` phù hợp điều gì sẽ xảy ra nếu chúng ta định nghĩa lại cấu trúc như sau

```
type TerraformResource struct {
  Cloud                string                       // 16 bytes
  Name                 string                       // 16 bytes
  PluginVersion        string                       // 16 bytes
  TerraformVersion     string                       // 16 bytes
  ModuleVersionMajor   int32                        //  4 bytes
  HaveDSL              bool                         //  1 byte
  IsVersionControlled  bool                         //  1 byte
}
```

Chạy lại đoạn code giống ban đầu sau khi struct được optimized

```go
package main

import "fmt"
import "unsafe"

type TerraformResource struct {
  Cloud                string                       // 16 bytes
  Name                 string                       // 16 bytes
  PluginVersion        string                       // 16 bytes
  TerraformVersion     string                       // 16 bytes
  ModuleVersionMajor   int32                        //  4 bytes
  HaveDSL              bool                         //  1 byte
  IsVersionControlled  bool                         //  1 byte
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

[Link execute](https://go.dev/play/p/ssyyF0E7jzg)

**Output**

```cmd
==============================================================
Total Memory Usage StructType:d main.TerraformResource => [72]
==============================================================
Cloud Field StructType:d.Cloud string => [16]
Name Field StructType:d.Name string => [16]
HaveDSL Field StructType:d.HaveDSL bool => [1]
PluginVersion Field StructType:d.PluginVersion string => [16]
ModuleVersionMajor Field StructType:d.IsVersionControlled bool => [1]
TerraformVersion Field StructType:d.TerraformVersion string => [16]
ModuleVersionMajor Field StructType:d.ModuleVersionMajor int32 => [4]
```

Bây giờ tổng bộ nhớ được cấp phát cho kiểu `TerraformResource` là `72 bytes`. Hãy cùng nhìn xem cách mà bộ nhớ liên kết

![images](https://miro.medium.com/max/1400/1*zdk8sNJKSWyMnF0moMyJGg.jpeg)

Chỉ bằng cách sửa các thành phần của struct phù hợp với `data structure alignment`, chúng ta đã có thể giảm dung lượng bộ nhớ từ `88 bytes` xuống còn `72 bytes` ...wow!

Cùng check lại 1 chút

>Allocation bytes =16 bytes+16 bytes+ 1 bytes+ 16 bytes+ 1 bytes+ 16 bytes+ 4 bytes = 70 bytes
>
>Empty Pad bytes = 2 bytes
>
>Total bytes = Allocation bytes + Empty Pad bytes = 70 bytes + 2 bytes = 72 bytes

`data structure alignment` không chỉ giúp chúng ta sử dụng bộ nhớ hiệu quả mà còn với Chu kỳ đọc của CPU….

 CPU đọc bộ nhớ theo từng ***words***, ***4 bytes trên 32-bit, 8 bytes trên 64-bit systems***.
 Theo khai báo ban đầu của `struct` `TerraformResource` sẽ mất ***11 bytes*** để CPU đọc tất cả

 ![images](https://miro.medium.com/max/1400/1*D8FH5hBAZrII7CtDzPd8qw.jpeg)


 Tuy nhiên sau khi ***optimized** `struct` sẽ chỉ cần `9 Word` theo hình dưới đây:


![images](https://miro.medium.com/max/1400/1*slD7-jFLuWxYitN9B-_bGA.jpeg)


Bằng cách xác định đúng cấu trúc, dữ liệu được căn chỉnh theo cấu trúc, chúng ta có thể sử dụng cấp phát bộ nhớ một cách hiệu quả và giúp cho việc CPU đọc dữ liệu lưu bởi cấu trúc cũng nhanh chóng và hiệu quả.


Đây chỉ là một ví dụ nhỏ, hãy nghĩ về một cấu trúc lớn với 20 hoặc 30 trường với các kiểu khác nhau.
Việc định nghĩa 1 cấu trúc chuẩn, phuf hợp với ***alignment of data structure*** thực sự mang lại hiệu quả cao...

Hy vọng blog này có thể làm sáng tỏ về cấu trúc bên trong, phân bổ bộ nhớ của chúng và chu kỳ đọc CPU cần thiết. 
Hi vọng điều nay có ich cho bạn!!

## Happy Coding!!