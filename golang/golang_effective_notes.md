# 関数
## 複数の戻り値
## 名前付き結果パラメータ
## Defer
Lập lịch gọi 1 hàm thực hiện ngay trước khi return. Một hàm đặc biệt nhưng hiệu quả trong tình huống phải release 1 resource bất kỳ khi nào hàm bị return. Ví dụ kinh điển là unlocking a mutex hoặc đóng file.
```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```
Lợi ích:
- Đảm bảo không quên việc đóng file so với thêm đóng file ở cuối hàm
- Close ngay dưới hàm Open, khiến cho code clear hơn so với viết ở cuối hàm

Giá trị biến đầu vào của defer function được ấn định khi hàm defer thực thi, không phải lúc hàm gọi được thực thi
```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```
Các hàm đã defer được thực thi theo LIFO. -> Đoạn code trên sẽ in ra 4 3 2 1 0 khi hàm return

## `new`でメモリを割り当て
`new(T)`につき、Go の用語では、新しく割り当てられた T 型のゼロ値へのポインタを返します。
## Constructors and composite literals
***Constructors***
```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```
***composite literal***
```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f = File{fd, name, nil, 0}
    return &f
}
```
## `make`でメモリを割り当て
`slices`, `map`, `chanel`のみを作成し、タイプ T (*T ではない) の初期化された (ゼロ化されていない) 値を返します。
```go
var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

// Unnecessarily complex:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Idiomatic:
v := make([]int, 100)
```
## Arrays
- Arrays are values. Assigning one array to another copies all the elements.
- In particular, if you pass an array to a function, it will receive a copy of the array, not a pointer to it.
- The size of an array is part of its type. The types [10]int and [20]int are distinct.
The value property can be useful but also expensive; if you want C-like behavior and efficiency, you can pass a pointer to the array.
```go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // Note the explicit address-of operator
```
But even this style isn't idiomatic Go. Use slices instead.
## Slices
- Slices hold references to an underlying array, and if you assign one slice to another, both refer to the same array. 
- If a function takes a slice argument, changes it makes to the elements of the slice will be visible to the caller, analogous to passing a pointer to the underlying array

### Append関数
```go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // reallocate
        // Allocate double what's needed, for future growth.
        newSlice := make([]byte, (l+len(data))*2)
        // The copy function is predeclared and works for any slice type.
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    copy(slice[l:], data)
    return slice
}
```
## Maps
Key Value struct

Key type: equality operator is defined such as integers, floating point and complex numbers, strings, pointers, interfaces (as long as the dynamic type supports equality), structs and arrays. Slices cannot be used as map keys, because equality is not defined on them
```go
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
offset := timeZone["EST"]
```
Need to distinguish a missing entry from a zero value
```go
var seconds int
var ok bool
seconds, ok = timeZone[tz]

func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```
To delete a map entry, use the delete built-in function, whose arguments are the map and the key to be deleted. It's safe to do this even if the key is already absent from the map.
```go
delete(timeZone, "PDT")  // Now on Standard Time
```
## Constants
- Constants in Go are created at compile time, even when defined as locals in functions, and can only be numbers, characters (runes), strings or booleans.
1<<3 is a constant expression, while math.Sin(math.Pi/4) is not because the function call to math.Sin needs to happen at run time.

