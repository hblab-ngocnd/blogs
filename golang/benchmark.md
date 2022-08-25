## Test benchmark fibonaci recursive/non recursive

### Động cơ, lý do

Trước giờ mình vẫn có thắc mắc là đối với cùng 1 bài toán giữa recursive và non recursive thì cái nào sẽ là cách giải quyết hiệu quả hơn.
Nhân dịp mình có review code của các member trong team, mọi người có trao đổi về recursive/ non recursive nên mình quyết định tìm hiểu sâu hơn xem suy nghĩ từ đầu của mình đã đúng chưa.

Mình có tìm hiểu các tài liệu thì hầu như mọi người đều có chung nhận xét như sau:

>Ưu điểm của đệ quy (recursive): Gọn gàng dễ hiểu, dễ viết code

>Nhược điểm: Tốn không gian bộ nhớ, và thời gian xử lý do phải switch context giữa các lần gọi hàm.

Trên đây đều là những nhận xét mang tính lý thuyết. Với góc nhìn của một người hiện tại đang là một kỹ sư từng là học sinh chuyên lý, mình chỉ tin khi có kết quả thí nghiệm là những con số cụ thể.

Và thật may `go` giúp mình làm điều này khá dễ dàng bằng công cụ [test benchmarks](https://pkg.go.dev/testing#hdr-Benchmarks) 

Không nói nhiều nữa, chúng ta cùng vào chủ đề chính.

## Test benchmarks với fibonaci recursive

Đầu tiên chúng ta thực hiện cài đặt thuật toán với cách tiếp cận là đệ quy.

Cài đặt test benchmark với đầu vào N = 10

```go
package bench

import (
	"testing"
)

func FibRecursive(n int) int {
	if n < 2 {
		return n
	}
	return FibRecursive(n-1) + FibRecursive(n-2)
}

func BenchmarkFibRecursive10(b *testing.B) {
	// run the FibRecursive function b.N times
	for n := 0; n < b.N; n++ {
		FibRecursive(10)
	}
}
```
Thực hiện test benchmark bằng lệnh sau

`go test -bench=. -benchmem -memprofile memprofile.out -cpuprofile profile.out`

Kết quả

```
goos: darwin
goarch: amd64
pkg: github.com/hblab-ngocnd/csv-demo/bench
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkFibRecursive10-12       4973802               239.1 ns/op             0 B/op          0 allocs/op
PASS
ok      github.com/hblab-ngocnd/csv-demo/bench  1.751s
```

Nhận xét: 
Lặp `4973802` lần, mỗi lần mất `239.1 ns` tổng `1189236058.2 ns`

***Máy mình tận `i7` lận, giờ mới để ý***

Phân tích sâu hơn 1 tý

### Phân tích thời gian chạy

Chạy lệnh

`go tool pprof profile.out`

```cmd
Type: cpu
Time: Aug 25, 2022 at 3:12pm (+07)
Duration: 1.64s, Total samples = 1.23s (75.02%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
```
Dùng lệnh top để lấy những xử lý mất thời gian nhất, sắp xếp theo thứ tự giảm dần

Kết quả
```
Showing nodes accounting for 1.23s, 100% of 1.23s total
      flat  flat%   sum%        cum   cum%
     1.21s 98.37% 98.37%      1.21s 98.37%  github.com/hblab-ngocnd/csv-demo/bench.FibRecursive
     0.01s  0.81% 99.19%      1.22s 99.19%  github.com/hblab-ngocnd/csv-demo/bench.BenchmarkFibRecursive10
     0.01s  0.81%   100%      0.01s  0.81%  runtime/pprof.(*profMap).lookup
         0     0%   100%      0.01s  0.81%  runtime/pprof.(*profileBuilder).addCPUData
         0     0%   100%      0.01s  0.81%  runtime/pprof.profileWriter
         0     0%   100%      1.22s 99.19%  testing.(*B).launch
         0     0%   100%      1.22s 99.19%  testing.(*B).runN
```
Mất tổng thời gian chạy cho `FibRecursive` là `1.21s`

Tiếp tục phân tích sâu hơn từng dòng lệnh đối với hàm `FibRecursive`

`(pprof) list FibRecursive`

Kết qủa:

```
Total: 1.23s
ROUTINE ======================== github.com/hblab-ngocnd/csv-demo/bench.BenchmarkFibRecursive10 in /Users/S16593/Hblab/Motify/csv-demo/bench/Figo_test.go
      10ms      1.22s (flat, cum) 99.19% of Total
         .          .     11:   return FibRecursive(n-1) + FibRecursive(n-2)
         .          .     12:}
         .          .     13:
         .          .     14:func BenchmarkFibRecursive10(b *testing.B) {
         .          .     15:   // run the FibRecursive function b.N times
      10ms       10ms     16:   for n := 0; n < b.N; n++ {
         .      1.21s     17:           FibRecursive(10)
         .          .     18:   }
         .          .     19:}
         .          .     20:
         .          .     21://func FibNonRecursive(n int) int {
         .          .     22:// if n < 2 {
ROUTINE ======================== github.com/hblab-ngocnd/csv-demo/bench.FibRecursive in /Users/S16593/Hblab/Motify/csv-demo/bench/Figo_test.go
     1.21s      1.99s (flat, cum) 161.79% of Total
         .          .      2:
         .          .      3:import (
         .          .      4:   "testing"
         .          .      5:)
         .          .      6:
     540ms      540ms      7:func FibRecursive(n int) int {
      40ms       40ms      8:   if n < 2 {
     200ms      200ms      9:           return n
         .          .     10:   }
     430ms      1.21s     11:   return FibRecursive(n-1) + FibRecursive(n-2)
         .          .     12:}
         .          .     13:
         .          .     14:func BenchmarkFibRecursive10(b *testing.B) {
         .          .     15:   // run the FibRecursive function b.N times
         .          .     16:   for n := 0; n < b.N; n++ {
```
Show kết quả graph bằng png, pdf dùng command:

`(pprof) web`

![images](https://hblab-ngocnd.github.io/blogs/golang/images/fibonaci-recursive.svg)

Kết thúc phân tích

`(pprof) q`

### Phân tích về cấp phát bộ nhớ

`go tool pprof memprofile.out`

```
(pprof) top
Showing nodes accounting for 5140.68kB, 100% of 5140.68kB total
Showing top 10 nodes out of 30
      flat  flat%   sum%        cum   cum%
 1537.69kB 29.91% 29.91%  1537.69kB 29.91%  runtime.allocm
 1024.41kB 19.93% 49.84%  1024.41kB 19.93%  runtime.malg
  902.59kB 17.56% 67.40%  1553.21kB 30.21%  compress/flate.NewWriter
  650.62kB 12.66% 80.05%   650.62kB 12.66%  compress/flate.(*compressor).init
  512.88kB  9.98% 90.03%   512.88kB  9.98%  flag.(*FlagSet).Var
  512.50kB  9.97%   100%   512.50kB  9.97%  hash/crc32.simpleMakeTable (inline)
         0     0%   100%  1553.21kB 30.21%  compress/gzip.(*Writer).Write
         0     0%   100%   512.88kB  9.98%  flag.Var (inline)
         0     0%   100%   512.50kB  9.97%  hash/crc32.init
         0     0%   100%   512.88kB  9.98%  main.main
```

`(pprof) web`

![images](https://hblab-ngocnd.github.io/blogs/golang/images/fibonaci-recursive-mem.svg)


## Test benchmarks với fibonaci non recursive

Cài đặt thuật toán với cách tiếp cận là khử đệ quy.

Cài đặt test benchmark với đầu vào N = 10 (giống bước trên)

```go
package bench

import (
	"testing"
)

func FibNonRecursive(n int) int {
	if n < 2 {
		return n
	}
	f1 := 0
	f2 := 1
	for i := 2; i < n; i++ {
		tmp := f2
		f2 = f1 + f2
		f1 = tmp
	}
	return f2
}

func BenchmarkFibNonRecursive10(b *testing.B) {
	// run the FibNonRecursive function b.N times
	for n := 0; n < b.N; n++ {
		FibNonRecursive(10)
	}
}
```

***Optimized***

Do biến `tmp` được cấp phát trong vòng lặp làm tăng bộ nhớ khá nhiều, gây nên kết luận sai. Sau khi tìm hiểu được nguyên nhân thì mình sửa thành như sau

```go
package bench

import (
	"testing"
)

func FibNonRecursive(n int) int {
	if n < 2 {
		return n
	}
	f1 := 0
	f2 := 1
    tmp := 0
	for i := 2; i < n; i++ {
		tmp = f2
		f2 = f1 + f2
		f1 = tmp
	}
	return f2
}

func BenchmarkFibNonRecursive10(b *testing.B) {
	// run the FibNonRecursive function b.N times
	for n := 0; n < b.N; n++ {
		FibNonRecursive(10)
	}
}
```

#### Phân tích
Tiếp tục chạy lệnh tương tự

`go test -bench=. -benchmem -memprofile memprofile.out -cpuprofile profile.out`

Kết quả:

```
goos: darwin
goarch: amd64
pkg: github.com/hblab-ngocnd/csv-demo/bench
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkFibNonRecursive10-12           223505781                5.321 ns/op           0 B/op          0 allocs/op
PASS
ok      github.com/hblab-ngocnd/csv-demo/bench  4.895s
```
có `223505781` phép toán mỗi phép toán mất `5.321 ns` tổng `1189274260.7 ns`

***Optimized***

```
goos: darwin
goarch: amd64
pkg: github.com/hblab-ngocnd/csv-demo/bench
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkFibNonRecursive10-12           238679884                4.987 ns/op           0 B/op          0 allocs/op
PASS
ok      github.com/hblab-ngocnd/csv-demo/bench  1.936s
```

Chạy lệnh

`go tool pprof profile.out`

`(pprof) top`
```
Type: cpu
Time: Aug 25, 2022 at 3:43pm (+07)
Duration: 1.85s, Total samples = 1.48s (79.92%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 1480ms, 100% of 1480ms total
      flat  flat%   sum%        cum   cum%
     980ms 66.22% 66.22%      980ms 66.22%  github.com/hblab-ngocnd/csv-demo/bench.FibNonRecursive (inline)
     360ms 24.32% 90.54%     1460ms 98.65%  github.com/hblab-ngocnd/csv-demo/bench.BenchmarkFibNonRecursive10
     120ms  8.11% 98.65%      120ms  8.11%  runtime.asyncPreempt
      20ms  1.35%   100%       20ms  1.35%  runtime.nanotime1
         0     0%   100%       20ms  1.35%  runtime.nanotime
         0     0%   100%       20ms  1.35%  testing.(*B).StopTimer
         0     0%   100%     1480ms   100%  testing.(*B).launch
         0     0%   100%     1480ms   100%  testing.(*B).runN
         0     0%   100%       20ms  1.35%  time.Since
```

Mất tổng thời gian chạy cho `FibNonRecursive` là `980ms`

***Optimized***

```
Type: cpu
Time: Aug 26, 2022 at 3:20am (+07)
Duration: 1.85s, Total samples = 1.51s (81.53%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 1.51s, 100% of 1.51s total
Showing top 10 nodes out of 20
      flat  flat%   sum%        cum   cum%
     0.90s 59.60% 59.60%      0.90s 59.60%  github.com/hblab-ngocnd/csv-demo/bench.FibNonRecursive (inline)
     0.55s 36.42% 96.03%      1.47s 97.35%  github.com/hblab-ngocnd/csv-demo/bench.BenchmarkFibNonRecursive10
     0.02s  1.32% 97.35%      0.02s  1.32%  runtime.asyncPreempt
     0.02s  1.32% 98.68%      0.02s  1.32%  runtime.libcCall
     0.01s  0.66% 99.34%      0.02s  1.32%  runtime.kevent
     0.01s  0.66%   100%      0.02s  1.32%  runtime.pthread_cond_wait
         0     0%   100%      0.03s  1.99%  runtime.findrunnable
         0     0%   100%      0.02s  1.32%  runtime.mPark (inline)
         0     0%   100%      0.03s  1.99%  runtime.mcall
         0     0%   100%      0.02s  1.32%  runtime.netpoll
```
Mất tổng thời gian chạy cho `FibNonRecursive` là `0.90s`

`(pprof) list FibNonRecursive`
```
(pprof) list FibNonRecursive
Total: 1.48s
ROUTINE ======================== github.com/hblab-ngocnd/csv-demo/bench.BenchmarkFibNonRecursive10 in /Users/S16593/Hblab/Motify/csv-demo/bench/Figo_test.go
     360ms      1.46s (flat, cum) 98.65% of Total
         .          .     32:   return f2
         .          .     33:}
         .          .     34:
         .          .     35:func BenchmarkFibNonRecursive10(b *testing.B) {
         .          .     36:   // run the FibNonRecursive function b.N times
     110ms      160ms     37:   for n := 0; n < b.N; n++ {
     250ms      1.23s     38:           FibNonRecursive(10)
         .          .     39:   }
         .       70ms     40:}
ROUTINE ======================== github.com/hblab-ngocnd/csv-demo/bench.FibNonRecursive in /Users/S16593/Hblab/Motify/csv-demo/bench/Figo_test.go
     980ms      980ms (flat, cum) 66.22% of Total
         .          .     22:   if n < 2 {
         .          .     23:           return n
         .          .     24:   }
         .          .     25:   f1 := 0
         .          .     26:   f2 := 1
     980ms      980ms     27:   for i := 2; i < n; i++ {
         .          .     28:           tmp := f2
         .          .     29:           f2 = f1 + f2
         .          .     30:           f1 = tmp
         .          .     31:   }
         .          .     32:   return f2
```

***Optimized***

```
(pprof) list FibNonRecursive
Total: 1.51s
ROUTINE ======================== github.com/hblab-ngocnd/csv-demo/bench.BenchmarkFibNonRecursive10 in /Users/S16593/Hblab/Motify/csv-demo/bench/Figo_test.go
     550ms      1.47s (flat, cum) 97.35% of Total
         .          .     33:   return f2
         .          .     34:}
         .          .     35:
         .          .     36:func BenchmarkFibNonRecursive10(b *testing.B) {
         .          .     37:   // run the FibNonRecursive function b.N times
     190ms      200ms     38:   for n := 0; n < b.N; n++ {
     360ms      1.26s     39:           FibNonRecursive(10)
         .          .     40:   }
         .       10ms     41:}
ROUTINE ======================== github.com/hblab-ngocnd/csv-demo/bench.FibNonRecursive in /Users/S16593/Hblab/Motify/csv-demo/bench/Figo_test.go
     900ms      900ms (flat, cum) 59.60% of Total
         .          .     23:           return n
         .          .     24:   }
         .          .     25:   f1 := 0
         .          .     26:   f2 := 1
         .          .     27:   tmp := f2
     900ms      900ms     28:   for i := 2; i < n; i++ {
         .          .     29:           tmp = f2
         .          .     30:           f2 = f1 + f2
         .          .     31:           f1 = tmp
         .          .     32:   }
         .          .     33:   return f2
```
`(pprof) list web`

![images](https://hblab-ngocnd.github.io/blogs/golang/images/fibonaci-non-recursive.svg)

***Optimized***

![images](https://hblab-ngocnd.github.io/blogs/golang/images/fibonaci-non-recursive-optimized.svg)

`(pprof) q`

### Phân tích về cấp phát bộ nhớ

`go tool pprof memprofile.out`

`(pprof) top`

```
Type: alloc_space
Time: Aug 25, 2022 at 3:43pm (+07)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 5811.98kB, 100% of 5811.98kB total
Showing top 10 nodes out of 26
      flat  flat%   sum%        cum   cum%
 2050.25kB 35.28% 35.28%  2050.25kB 35.28%  runtime.allocm
 1184.27kB 20.38% 55.65%  1184.27kB 20.38%  runtime/pprof.StartCPUProfile
  902.59kB 15.53% 71.18%  1553.21kB 26.72%  compress/flate.NewWriter
  650.62kB 11.19% 82.38%   650.62kB 11.19%  compress/flate.(*compressor).init
  512.20kB  8.81% 91.19%   512.20kB  8.81%  runtime.malg
  512.05kB  8.81%   100%  1696.32kB 29.19%  runtime.main
         0     0%   100%  1553.21kB 26.72%  compress/gzip.(*Writer).Write
         0     0%   100%  1184.27kB 20.38%  main.main
         0     0%   100%  1025.12kB 17.64%  runtime.mcall
         0     0%   100%  1025.12kB 17.64%  runtime.mstart
```

***Optimized***

```
Type: alloc_space
Time: Aug 26, 2022 at 3:20am (+07)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 3746.37kB, 100% of 3746.37kB total
Showing top 10 nodes out of 21
      flat  flat%   sum%        cum   cum%
 1537.69kB 41.04% 41.04%  1537.69kB 41.04%  runtime.allocm
 1184.27kB 31.61% 72.66%  1184.27kB 31.61%  runtime/pprof.StartCPUProfile
 1024.41kB 27.34%   100%  1024.41kB 27.34%  runtime.malg
         0     0%   100%  1184.27kB 31.61%  main.main
         0     0%   100%  1184.27kB 31.61%  runtime.main
         0     0%   100%   512.56kB 13.68%  runtime.mcall
         0     0%   100%  1025.12kB 27.36%  runtime.mstart
         0     0%   100%  1025.12kB 27.36%  runtime.mstart0
         0     0%   100%  1025.12kB 27.36%  runtime.mstart1
         0     0%   100%  1537.69kB 41.04%  runtime.newm
```

`(pprof) web`

![images](https://hblab-ngocnd.github.io/blogs/golang/images/fibonaci-non-recursive-mem.svg)

***Optimized***

![images](https://hblab-ngocnd.github.io/blogs/golang/images/fibonaci-non-recursive-mem-optimized.svg)

## So sánh và kết luận với trường hợp N = 10

| Tiêu chí      | Recursive |  Non Recursive     | Non Recursive Optimized |
| :---        |    :----:   |          ---: | ---: |
| Time      | `1.21s`      |  `0.98s`  | `0.90s` |
| Memory   | `5140.68kB`        | `5811.98kB`      |  `3746.37kB` |

Với trường hợp N = 10 không có quá nhiều khác biệt. Tuy nhiên với số lần lặp lớn hơn khả năng sự khác biệt sẽ càng rõ.

***Updated***

Đúng với nhận xét ban đầu, sau khi optimized `Non Recursive` có thời gian chạy thấp hơn và tốn ít bộ nhớ hơn so với `Recursive`

Bạn hãy test thử xem ***let try !!!***

### Tài liệu tham khảo

- https://uet.vnu.edu.vn/~chauttm/dsa2015f/readings/Recursion.html
- https://users.soict.hust.edu.vn/trungtt/uploads/slides/KTLT_Bai03.pdf
- https://medium.com/@felipedutratine/profile-your-benchmark-with-pprof-fb7070ee1a94

***Happy codding !!!***
