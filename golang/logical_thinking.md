## Tựa đề
## Bài toán
Hiện tại mình đang làm việc cho  khách hàng dưới dạng remote onsite.

Mình được sắp xếp thuộc team `content engineer`

Công nghệ team mình đang sử dụng chủ yếu là : `golang`, `aws`, `docker`,`serverless`,`reactjs`, `terraform`,...

Bài toán bắt đầu khi mình được giao một task hạ level log từ `error` xuống `warn` ở những nơi gọi tới một `service` có trong project đang phát triển để tiện lợi trong việc điều tra lỗi.

### Phân tích bài toán
Việc đầu tiên trong task này là liệt kê những chỗ trong source code thuộc folder handler các http request  gọi đến server.
Xác nhận với leader có cần thiết phải hạ level log đối với những xử lý cụ thể hay không.

### Phân tích hiện trạng source code
***Cấu trúc folder trong dự án***
```
project
│   README.md
│   file001.txt    
│
└───serviceTranslate
│   │   translate.go //file chứa chứ ký của interface, dùng trong việc tạo mock test
│   │   aa.go
│   
│   
└───web //folder chứa souce code handler các http request
│   │   translate.go
│   │   aa.go
│   │   ...
│   │   web.go //file khai báo, khởi tạo các singleton object services
```

Mỗi service sẽ được tổ chức thành từng folder, trong đó file cùng tên package sẽ định nghĩa interface chứa các method public được gọi từ bên ngoài service.
Trong ví dụ này là file `translate.go` có dạng như sau.

```go
##serviceTranslate/translate.go

package translate
...

var (
	svc            Service
	once           sync.Once
)
//Singleton Design Pattern
func GetService(ctx context.Context) (Service,error) {
	once.Do(func() {
        cf := &Config{
      		Enpoint : os.Getenv("ENPOINT"),
     		Port : os.Getenv("POST"),
        }
    	svc = initService(cf)
    })
	return svc
}

type Service interface {
	TranslateData(context.Context, []models.Word) []models.Word
    PutData(context.Context, []models.Word) error
 	BatchTranslate(context.Context, []models.Word) []models.Word
    ...
}

type service struct {
	 conn *db.Conn
}

func initService(cf Config) *service {
	//call config bla..bla
    ...
    conn := &db.Conn{}
	return &service{
    	conn : conn
    }
}
func (t *service) TranslateData(ctx context.Context, data []models.Word) []models.Word {
	//Implement here
}

func (t *service) PutData(ctx context.Context, data []models.Word) error{
	//Implement here
}

func (t *service) BatchTranslate(ctx context.Context, data []models.Word) error{
	//Implement here
}
```
`web` : package chứa code các router thực hiện handler các request

 ```go
 ##web/web.go
 
 package web
 ...
 var (
 	translateService = translate.GetService
	...
 )
 
 func NewRouter() *chi.Mux {
 	r := chi.NewRouter()
	r.Route("/translate", routeTranslate)
    ....
  	return r
 }
 ```
Các `http request handler` sẽ gọi tới service thông qua biến global này theo dạng sau.
Mục đích là để inject `mock test`  service dành cho việc viết `unit test`

 ```go
 ##web/translate.go
 
 package web
 
 func routeTranslate(r chi.Router) {
	r.Get("/", h.get)
}

func (h *healthcheck) get(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
    tranService, err := translateService(ctx)
  	if err != nil {
 		//handler err
 	}
	...
    res,err := tranService.TranslateData(ctx,data) //call service function
  	if err != nil {
 		//handler err
 	}
	sendBody(w, r, res, http.StatusOK)
}
```
###  Cách xử lý thủ công
Liệt kê các method có trong interface, thông qua việc dùng IDE tìm kiếm thông qua từng tên các method được định nghĩa ở interface.
Điều này có nghĩa là mình sẽ lặp lại các hành động dưới đây đối với cùng một method name.

1. Tìm kiếm bằng từ khoá tên method
2. Với mỗi file, các dòng gọi đến service method cần note lại vị trí `local path`
3. Tạo link show code trên github.com có dạng như sau [link](https://github.com/hblab-ngocnd/get-started/blob/60d03839c89719a1c158d9512d244e88230fbfad/controller/dictionary.go#L82)

Hiện tại trong interface khai báo khoảng 20 method tức mình phải lặp lại các hành động trên 20 lần, mỗi lần sẽ phụ thuộc vào số lần xuất hiện của method name trong file.

humm... Việc này quả là nhàm chán đối với 1 engineer

Vì cách hành động trên là các công việc lặp lại, nên tại sao mình không giải quết nó bằng code nhỉ !!

### Cách tiếp cận bằng code
Để giải quyết bài toán yế tưởng của mình là sẽ viết 1 hàm test để cùng folder chứa service, trong hàm sẽ làm những việc sau

- Việc đầu tiên cần liệt kê các method có trong interface. -> tạo regex để tìm kiếm
- Đọc các file có trong folder web, trừ file có đuôi _test.go là file chứ unitest
- Đối với mỗi file đọc từng dòng để tìm kiếm nơi gọi đến method của service
- Thống kê kết quả và in ra màn hình list link trỏ đến dòng code cần xác nhận

Cùng giải quyết lần lượt các step

#### Liệt  kê các method name của interface, sinh ra chuỗi regex
Đối với việc này chúng ta có thể thực hiện bằng tay tuy nhiên phương án tiếp cận là code nên mình sẽ tìm cách xử lý bằng code.
Thật may trong `golang` có cách để chúng ta làm điều này
```go
##serviceTranslate/translate_test.go

itf := reflect.TypeOf((*Service)(nil)).Elem()
var methods []string
for i := 0; i < itf.NumMethod(); i++ {
	// thường sẽ gọi theo dạng tran.MethodName, size của biến tran là 1 or more, trước đấy sẽ có dấu cách
	methods = append(methods, strings.Join([]string{"[ ].{1,}[.]", itf.Method(i).Name}, ""))
}
```
Tạo regex đùng cho việc tìm kiếm
```go
func makeHasCallFuncRegex(s []string) *regexp.Regexp {
	strRaw := strings.Join(s, "|")
	reg := regexp.MustCompile(strRaw)
	return reg
}
```
#### Đọc các file code có trong folder web

```go
##serviceTranslate/translate_test.go

webPath := path.Join(curPath, "../web")
files, err := ioutil.ReadDir(webPath)
if err != nil {
	log.Fatal(err)
}
results := make([]fs.FileInfo, 0, len(files))
for _, file := range files {
	if strings.Index(file.Name(), "_test.go") == -1 {
		results = append(results, file)
	}
}
```
#### Xử lý tìm kiếm đối với từng file

Theo suy nghĩ ban đầu chúng ta sẽ xử lý tuần tự lần lượt với từng file.
Tổng thời gian xử lý sẽ mất bằng tổng các thời gian xử lý mỗi file.
Tuy nhiên việc này khá mất thời gian vì số lượng file ở folder web khá nhiểu.
Hơn nữa việc đọc file sẽ mất thời gian phụ thuộc vào độ dài của file. (Mỗi file hiện tại khoảng 3000 dòng)
Vì kết quả của việc này là liệt kê nên thứ tự file là không quan trọng nên
`Golang` có hướng tiếp cận việc này khá hiệu quả với khái niệm `concurrency`

***==> Hướng xử lý sẽ là đọc đồng thời theo kiểu `concurrency`***

Rất may `golang` hỗ trợ điều này rất tốt, giúp chúng ta dễ dàng thực hiện việc này.

```go
##serviceTranslate/translate_test.go

//Tạo buffer chanel nhận lấy kết quả từ các  go goroutines, truyền cho goroutines chính
c := make(chan []UsedInfo)
defer close(c)
for _, file := range results {
	go GetFileLineUsedTranlateService(path.Join(webPath, file.Name()), reg, c)
}
```
#### Tạo link show source code trên github
Xử lý trong hàm `GetFileLineUsedTranlateService`
Check từng dòng, nếu gặp call đến biến local `translateService(ctx)` để lấy `service instance`
```go
if strings.Index(fileScanner.Text(), "translateService(ctx)") > -1
```
Thì thực hiện set biến `tranServiceObjectName` lưu tên `service instance` được set trong code

Gặp line text pass regex sẽ tạo instance struct `UsedInfo` lưu thông tin xuất hiên.
```go
type UsedInfo struct {
	FileName string
	Content  string
	Line     int
	Link     string
}
...
result = append(result, UsedInfo{
				FileName: path.Base(filePath),
				Line:     line,
				Content:  fileScanner.Text(),
				Link:     strings.Join([]string{baseLink, path.Base(filePath), fmt.Sprintf("#L%d", line)}, "/"),
                })
```
Điều kiện check pass regex và gọi qua tên biến lưu ở `daletServiceObjectName`
```go
if used := reg.FindString(fileScanner.Text()); used != "" && strings.Contains(used, daletServiceObjectName)
```
Toàn bộ xử lý của hàm
```go
func GetFileLineUsedTranlateService(filePath string, reg *regexp.Regexp, c chan<- []UsedInfo) {
	readFile, err := os.Open(filePath)
	if err != nil {
		fmt.Println(err)
	}
	defer readFile.Close()
	baseLink := "https://github.com/abema/mam-tool-api/blob/master/web"
	fileScanner := bufio.NewScanner(readFile)
	fileScanner.Split(bufio.ScanLines)
	line := 1
	result := make([]UsedInfo, 0, 10)
	tranServiceObjectName := ""
	for fileScanner.Scan() {
		if strings.Index(fileScanner.Text(), "translateService(ctx)") > -1 {
			tranServiceObjectName = strings.TrimSpace(strings.Split(fileScanner.Text(), ",")[0])
			line++
			continue
		}
        if daletServiceObjectName == "" {
			line++
			continue
		}
		if used := reg.FindString(fileScanner.Text()); used != "" && strings.Contains(used, daletServiceObjectName) {
			result = append(result, UsedInfo{
				FileName: path.Base(filePath),
				Line:     line,
				Content:  fileScanner.Text(),
				Link:     strings.Join([]string{baseLink, path.Base(filePath), fmt.Sprintf("#L%d", line)}, "/"),
			})
		}
		line++
	}
    //Gửi kết quả đến chanel
	c <- result
}
```
#### Thống kê kết quả và in ra màn hình
```go
usedInfoList := make([]UsedInfo, 0, len(results)*10)
for i := 0; i < len(results); i++ {
	//Lấy kết quả từ các goroutines
	info := <-c
	usedInfoList = append(usedInfoList, info...)
}
for _, info := range usedInfoList {
		fmt.Println(info.Link)
}
```
#### Kết quả
Chạy hàm test trên sẽ ra kết quả dạng như sau
```cmd
https://github.com/hblab-ngocnd/get-started/blob/main/controller/dictionary.go#L10
https://github.com/hblab-ngocnd/get-started/blob/main/controller/dictionary.go#L82
https://github.com/hblab-ngocnd/get-started/blob/main/controller/file.go#L10
https://github.com/hblab-ngocnd/get-started/blob/main/controller/file.go#L82
....
```
#### TODO: check lại và fix bug