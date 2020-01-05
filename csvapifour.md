---
layout: default
---

# Instantly convert CSV files to an API using Golang - Part D

* A full demo of this tool with a UI is available [here](http://web-app.326wy59fvd.us-east-1.elasticbeanstalk.com/)
* [Part A](./csvapione.html)
* [Part B](./csvapitwo.html)
* [Part C](./csvapithree.html)

In part C, we completed the functionality to ingest a csv file into the database. In this part, we will develop the functionality to create API end points and handlers that can retrieve the data we stored.

Let's start with adding a new route to our application. In `cmd/web/main.go`. Add the following changes.

```
...

//define the command line arguments.
r := mux.NewRouter()
r.HandleFunc("/", HelloWorld).Methods("GET")
r.HandleFunc("/", app.UploadFile).Methods("POST")
r.HandleFunc("/api/v1/{id:[a-z,0-9,A-Z,_,-]+}/headers", app.GetHeaders).Methods("GET")
http.ListenAndServe(":3000", r)
```

We added a new GET route to the `/api/v1/{id:[a-z,0-9,A-Z,_,-]+}/headers` path. The `{id:[a-z,0-9,A-Z,_,-]+}` part of the URL would map to any alphanumeric string and can be referred to using the variable `id` while parsing the URL. When this end point is called, we will pass the datafilekey as the `id` part of the URL.
The intention of this end-point is to return the headers of a file.

In order to be able to retrieve data from the database, we would have to make changes to the model we defined in `pkg/mysql/hoppy.go`. In this case, we are going to be fetching headers from the database table `uid_dataheaders`. Lets go ahead and define an intermediate struct representation of headers in the file `hoppy.go`

```
//Header : struct that holds a representation of headers
type Header struct {
	Position int
	Value    string
	DataType string
}
```

Now lets define a function to the `hoppy.go` file that can retrieve headers for a file using its datafilekey.
```
func (s *hoppyDB) GetHeaders(datafilekey string) ([]Header, error) {
	var datafileid int
	metadatasql := `SELECT datafileid FROM uid_datafile WHERE datafilekey='%s'`
	metadatasql = fmt.Sprintf(metadatasql, datafilekey)
	row := s.DB.QueryRow(metadatasql)
	err := row.Scan(&datafileid)

	stmt := `SELECT headerid, header, datatype from uid_dataheaders where datafileid=? order by headerid`
	headers := make([]Header, 0)
	rows, err := s.DB.Query(stmt, datafileid)
	if err != nil {
		return nil, err
	}
	for rows.Next() {
		header := new(Header)
		err := rows.Scan(&header.Position, &header.Value, &header.DataType)
		if err != nil {
			return nil, err
		}
		headers = append(headers, *header)
	}
	return headers, nil
}
```

This function accepts a datafilekey and then queries the `uid_dataheaders` table to get all the headers for the datafile. It then iterates over the result and returns an array of structs that represent these headers.
We defined this function to be a receiver on the hoppyDB struct, but we would need this function exposed to the other parts of the web application as well. So we would need to add it to the interface definition as well.

```
type HoppyDB interface {
	InsertMetaData(string) (int, error)
	InsertHeaders(int, []string, []string) (int, error)
	InsertData(int, int, [][]string) error
	GetHeaders(string) ([]Header, error)
}
```

Our GetHeaders function is ready to be called by an API handler. This handler will return a JSON response. Lets go back to the `cmd/web/main.go` file and define what our JSON response would look like. To do this lets, define a struct called `HeaderApiResponse`

```
type HeaderApiResponse struct{
		Payload []msql.Header `json:"payload"`
		Message string         `json:"message"`
}
```
The Payload field here is an array of headers that we defined in the `hoppy.go` file.
Lets now define the actual API Handler `GetHeaders`

```
func (app *Application) GetHeaders(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	//userid will eventually be passed from the context.
	headers, err := app.ModelService.FindHeaders(datafileid)
	if err != nil {
		w.WriteHeader(http.StatusNotFound)
		return
	}
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(HeaderApiResponse{Payload: headers, Message: ""})
	return
}

```
At this point, the `cmd/web/main.go` should look like :
```
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"

	_ "github.com/go-sql-driver/mysql"
	"github.com/gorilla/mux"
	cpros "github.com/mihirkelkar/csvapi/pkg/csvprocessor"
	msql "github.com/mihirkelkar/csvapi/pkg/mysql"
)

type Application struct {
	ModelService msql.HoppyDB
}

type HeaderApiResponse struct {
	Payload []msql.Header `json:"payload"`
	Message string        `json:"message"`
}

func HelloWorld(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "This is a hello from the golang web app")
}

//curl -v -X POST -F "myFile=@./noSession.csv" localhost:3000
func (app *Application) UploadFile(w http.ResponseWriter, r *http.Request) {
	fmt.Println("File Upload Endpoint Hit")
	// Parse our multipart form, 10 << 20 specifies a maximum
	// upload of 10 MB files.
	r.ParseMultipartForm(10 << 20)
	// it also returns the FileHeader so we can get the Filename,
	// the Header and the size of the file
	file, header, err := r.FormFile("myFile")
	if err != nil {
		fmt.Println("Error Retrieving the File")
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	defer file.Close()

	//fit the new file struct and validate for errors.
	err = os.MkdirAll("./data/", 0755)
	if err != nil {
		fmt.Println("File could not be created")
		w.WriteHeader(http.StatusInternalServerError)
	}

	filename := strings.Replace(header.Filename, " ", "_", -1)

	f, err := os.OpenFile("./data/"+filename, os.O_RDWR|os.O_CREATE, 0755)
	if err != nil {
		fmt.Println("file could not be opened")
		w.WriteHeader(http.StatusInternalServerError)
	}

	io.Copy(f, file)
	if err != nil {
		fmt.Println("file could not be copied")
		w.WriteHeader(http.StatusInternalServerError)
	}
	defer f.Close()

	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//Call the CSV processor
	csvFile, err := cpros.NewCsvUpload("./data/" + filename)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//Insert the metadata into the database.
	datakey := csvFile.GetDataFileKey()
	lastID, err := app.ModelService.InsertMetaData(datakey)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	headers := csvFile.GetHeaders()
	datatypes := make([]string, 0)
	for range headers {
		datatypes = append(datatypes, "string")
	}

	//Insert headers into the database
	maxHeaders, err := app.ModelService.InsertHeaders(lastID, headers, datatypes)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//enter the actual file content
	data := csvFile.GetData()
	err = app.ModelService.InsertData(maxHeaders, lastID, data)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	os.Remove("./data/" + filename)
	w.WriteHeader(http.StatusOK)
	fmt.Println(datakey)
	return
}

func (app *Application) GetHeaders(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	//userid will eventually be passed from the context.
	headers, err := app.ModelService.GetHeaders(id)
	if err != nil {
		w.WriteHeader(http.StatusNotFound)
		return
	}
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(HeaderApiResponse{Payload: headers, Message: ""})
	return
}

func GetDataBaseUrl() string {
	user := os.Getenv("USER")
	password := os.Getenv("PASS")
	url := os.Getenv("URL")
	database := os.Getenv("DB")

	return fmt.Sprintf(`%s:%s@tcp(%s:3306)/%s?parseTime=true`, user, password, url, database)
}

func main() {

	dsn := GetDataBaseUrl()
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		panic(err.Error())
	}
	defer db.Close()

	// Open doesn't open a connection. Validate DSN data:
	err = db.Ping()
	if err != nil {
		panic(err.Error()) // proper error handling instead of panic in your app
	}
	fmt.Println("Success")
	app := Application{ModelService: msql.NewHoppyDB(db)}

	//define the command line arguments.
	r := mux.NewRouter()
	r.HandleFunc("/", HelloWorld).Methods("GET")
	r.HandleFunc("/", app.UploadFile).Methods("POST")
	r.HandleFunc("/api/v1/{id:[a-z,0-9,A-Z,_,-]+}/headers", app.GetHeaders).Methods("GET")
	http.ListenAndServe(":3000", r)
}
```

and the `pkg/mysql/hoppy.go` file should look like :
```
package mysql

import (
	"database/sql"
	"fmt"
)

type HoppyDB interface {
	InsertMetaData(string) (int, error)
	InsertHeaders(int, []string, []string) (int, error)
	InsertData(int, int, [][]string) error
	GetHeaders(string) ([]Header, error)
}

type hoppyDB struct {
	DB *sql.DB
}

//Header : struct that holds a representation of headers
type Header struct {
	Position int
	Value    string
	DataType string
}

func NewHoppyDB(db *sql.DB) HoppyDB {
	return &hoppyDB{DB: db}
}

//Insert: Inserts the metadata about the file into the database.
func (s *hoppyDB) InsertMetaData(datafilekey string) (int, error) {

	stmt := `INSERT INTO  uid_datafile (datafilekey, user_id, private, filetype, created_at, updated_at)
           VALUES(?, 0, 0, "csv", UTC_TIMESTAMP(), UTC_TIMESTAMP())`

	result, err := s.DB.Exec(stmt, datafilekey)
	if err != nil {
		return 0, err
	}
	id, err := result.LastInsertId()
	if err != nil {
		return 0, err
	}

	return int(id), nil
}

//InsertHeaders: Insert header information into the headers table. The code
func (s *hoppyDB) InsertHeaders(datafileId int, headers []string, datatype []string) (int, error) {
	stmt := `INSERT INTO uid_dataheaders (datafileid, headerid, header, datatype) VALUES (?, ?, ?, ?)`

	for index, val := range headers {
		_, err := s.DB.Exec(stmt, datafileId, index, val, datatype[index])
		if err != nil {
			return 0, err
		}
	}
	return len(headers), nil
}

func (s *hoppyDB) InsertData(maxHeaders int, datafileId int, data [][]string) error {
	stmt := `INSERT INTO uid_datacontent (datafileid, headerid, rowid, value)
					VALUES (?, ?, ?, ?)`

	for rowid, values := range data[1:] { //need to remove the header row
		for index, val := range values[:maxHeaders] { //only go until the max number of headers
			_, err := s.DB.Exec(stmt, datafileId, index, rowid, val)
			if err != nil {
				return err
			}
		}
	}

	return nil
}

func (s *hoppyDB) GetHeaders(datafilekey string) ([]Header, error) {
	var datafileid int
	metadatasql := `SELECT datafileid FROM uid_datafile WHERE datafilekey='%s'`
	metadatasql = fmt.Sprintf(metadatasql, datafilekey)
	row := s.DB.QueryRow(metadatasql)
	err := row.Scan(&datafileid)

	stmt := `SELECT headerid, header, datatype from uid_dataheaders where datafileid=? order by headerid`
	headers := make([]Header, 0)
	rows, err := s.DB.Query(stmt, datafileid)
	if err != nil {
		return nil, err
	}
	for rows.Next() {
		header := new(Header)
		err := rows.Scan(&header.Position, &header.Value, &header.DataType)
		if err != nil {
			return nil, err
		}
		headers = append(headers, *header)
	}
	return headers, nil
}
```
Let's compile and run this application to test if this works end-to-end. Let's build the docker containers and re-deploy the application. Once the application is deployed, lets upload a file to it using the curl command. If the file upload and parsing is successful, an id should be printed out on the terminal.

Let's use the following curl command to retrieve the headers of the file.
```
curl -X GET localhost:3000/api/v1/<yourid>/headers
```
At this point, the app should return a JSON payload if it's working successfully. Let's now repeat the same process to add another API endpoint to return the top 100 lines of a file as a JSON payload.  

lets add a route first to the `cmd/web/main.go` file.

```
//define the command line arguments.
r := mux.NewRouter()
r.HandleFunc("/", HelloWorld).Methods("GET")
r.HandleFunc("/", app.UploadFile).Methods("POST")
r.HandleFunc("/api/v1/{id:[a-z,0-9,A-Z,_,-]+}/headers", app.GetHeaders).Methods("GET")
r.HandleFunc("/api/v1/{id:[a-z,0-9,A-Z,_,-]+}/data", app.GetData).Methods("GET")
http.ListenAndServe(":3000", r)
```

We defined a new route `/api/v1/<id>/data` that will call the `app.GetData` handler to retrieve the first 100 lines of a file.

Let's  define a database function for this in the `pkg/mysql/hoppy.go` file. Lets call this function, `GetData`. Lets define a struct representation for the content of the file data and call it `DataContent`


```
type DataContent struct {
	HeaderID int
	RowID    int
	Value    string
}
```

```
func (s *hoppyDB) GetData(datafilekey string) ([]DataContent, error) {
	var maxrowid int
	var datafileid int
	var datafileid int
	metadatasql := `SELECT datafileid FROM uid_datafile WHERE datafilekey='%s'`
	metadatasql = fmt.Sprintf(metadatasql, datafilekey)
	row := s.DB.QueryRow(metadatasql)
	err := row.Scan(&datafileid)

	rowstmt := `SELECT MAX(rowid) FROM uid_datacontent where datafileid=?`

	stmt := `SELECT headerid, rowid, value from uid_datacontent where datafileid=? order by headerid LIMIT 100;`

	datalist := make([]DataContent, 0)

	row := s.DB.QueryRow(rowstmt, datafileid)

	err := row.Scan(&maxrowid)

	if err != nil {
		fmt.Println("could not find max rowid for data file")
		return nil, err
	}

	rows, err := s.DB.Query(stmt, datafileid)
	if err != nil {
		return nil, err
	}

	for rows.Next() {
		data := new(DataContent)
		err := rows.Scan(&data.HeaderID, &data.RowID, &data.Value)
		if err != nil {
			return nil, err
		}
		datalist = append(datalist, *data)
	}
	return datalist, nil
}

```

The function follows pretty much the same pattern as the last function we developed. It runs a sql to find the first 100 lines and populates an array of `DataContent` structs with the data. This function then returns the array back to the calling entity. In order to make this function usable we have to add it's signature to the interface.

```
type HoppyDB interface {
	InsertMetaData(string) (int, error)
	InsertHeaders(int, []string, []string) (int, error)
	InsertData(int, int, [][]string) error
	GetHeaders(string) ([]Header, error)
	GetData(string) ([]DataContent, error)
}
```

Now, let's head-back to the `cmd/web/main.go` file and define a JSON response struct for the data content we want to return. It should follow the exact same pattern as the `HeaderApiResponse`.

```
type ContentApiResponse struct {
	Payload []msql.DataContent `json:"payload"`
	Message string             `json:"message"`
}
```

Let's define the handler.
```
func (app *Application) GetData(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	content, err := app.ModelService.GetData(id)
	if err != nil {
		w.WriteHeader(http.StatusNotFound)
		return
	}
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(ContentApiResponse{Payload: content, Message: ""})
	return
}
```

With this our `cmd/web/main.go` file should be complete and should look like this :
```
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"

	_ "github.com/go-sql-driver/mysql"
	"github.com/gorilla/mux"
	cpros "github.com/mihirkelkar/csvapi/pkg/csvprocessor"
	msql "github.com/mihirkelkar/csvapi/pkg/mysql"
)

type Application struct {
	ModelService msql.HoppyDB
}

type HeaderApiResponse struct {
	Payload []msql.Header `json:"payload"`
	Message string        `json:"message"`
}

type ContentApiResponse struct {
	Payload []msql.DataContent `json:"payload"`
	Message string             `json:"message"`
}

func HelloWorld(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "This is a hello from the golang web app")
}

//curl -v -X POST -F "myFile=@./noSession.csv" localhost:3000
func (app *Application) UploadFile(w http.ResponseWriter, r *http.Request) {
	fmt.Println("File Upload Endpoint Hit")
	// Parse our multipart form, 10 << 20 specifies a maximum
	// upload of 10 MB files.
	r.ParseMultipartForm(10 << 20)
	// it also returns the FileHeader so we can get the Filename,
	// the Header and the size of the file
	file, header, err := r.FormFile("myFile")
	if err != nil {
		fmt.Println("Error Retrieving the File")
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	defer file.Close()

	//fit the new file struct and validate for errors.
	err = os.MkdirAll("./data/", 0755)
	if err != nil {
		fmt.Println("File could not be created")
		w.WriteHeader(http.StatusInternalServerError)
	}

	filename := strings.Replace(header.Filename, " ", "_", -1)

	f, err := os.OpenFile("./data/"+filename, os.O_RDWR|os.O_CREATE, 0755)
	if err != nil {
		fmt.Println("file could not be opened")
		w.WriteHeader(http.StatusInternalServerError)
	}

	io.Copy(f, file)
	if err != nil {
		fmt.Println("file could not be copied")
		w.WriteHeader(http.StatusInternalServerError)
	}
	defer f.Close()

	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//Call the CSV processor
	csvFile, err := cpros.NewCsvUpload("./data/" + filename)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//Insert the metadata into the database.
	datakey := csvFile.GetDataFileKey()
	lastID, err := app.ModelService.InsertMetaData(datakey)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	headers := csvFile.GetHeaders()
	datatypes := make([]string, 0)
	for range headers {
		datatypes = append(datatypes, "string")
	}

	//Insert headers into the database
	maxHeaders, err := app.ModelService.InsertHeaders(lastID, headers, datatypes)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	//enter the actual file content
	data := csvFile.GetData()
	err = app.ModelService.InsertData(maxHeaders, lastID, data)
	if err != nil {
		fmt.Println(err.Error())
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	os.Remove("./data/" + filename)
	w.WriteHeader(http.StatusOK)
	fmt.Println(datakey)
	return
}

func (app *Application) GetHeaders(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	//userid will eventually be passed from the context.
	headers, err := app.ModelService.GetHeaders(id)
	if err != nil {
		w.WriteHeader(http.StatusNotFound)
		return
	}
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(HeaderApiResponse{Payload: headers, Message: ""})
	return
}

func (app *Application) GetData(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id := vars["id"]

	content, err := app.ModelService.GetData(id)
	if err != nil {
		w.WriteHeader(http.StatusNotFound)
		return
	}
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(ContentApiResponse{Payload: content, Message: ""})
	return
}

func GetDataBaseUrl() string {
	user := os.Getenv("USER")
	password := os.Getenv("PASS")
	url := os.Getenv("URL")
	database := os.Getenv("DB")

	return fmt.Sprintf(`%s:%s@tcp(%s:3306)/%s?parseTime=true`, user, password, url, database)
}

func main() {

	dsn := GetDataBaseUrl()
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		panic(err.Error())
	}
	defer db.Close()

	// Open doesn't open a connection. Validate DSN data:
	err = db.Ping()
	if err != nil {
		panic(err.Error()) // proper error handling instead of panic in your app
	}
	fmt.Println("Success")
	app := Application{ModelService: msql.NewHoppyDB(db)}

	//define the command line arguments.
	r := mux.NewRouter()
	r.HandleFunc("/", HelloWorld).Methods("GET")
	r.HandleFunc("/", app.UploadFile).Methods("POST")
	r.HandleFunc("/api/v1/{id:[a-z,0-9,A-Z,_,-]+}/headers", app.GetHeaders).Methods("GET")
	r.HandleFunc("/api/v1/{id:[a-z,0-9,A-Z,_,-]+}/data", app.GetData).Methods("GET")
	http.ListenAndServe(":3000", r)
}

```

and the `pkg/mysql/hoppy.go` file should look like this :
```
package mysql

import (
	"database/sql"
	"fmt"
)

type HoppyDB interface {
	InsertMetaData(string) (int, error)
	InsertHeaders(int, []string, []string) (int, error)
	InsertData(int, int, [][]string) error
	GetHeaders(string) ([]Header, error)
	GetData(string) ([]DataContent, error)
}

type hoppyDB struct {
	DB *sql.DB
}

//Header : struct that holds a representation of headers
type Header struct {
	Position int
	Value    string
	DataType string
}

type DataContent struct {
	HeaderID int
	RowID    int
	Value    string
}

func NewHoppyDB(db *sql.DB) HoppyDB {
	return &hoppyDB{DB: db}
}

//Insert: Inserts the metadata about the file into the database.
func (s *hoppyDB) InsertMetaData(datafilekey string) (int, error) {

	stmt := `INSERT INTO  uid_datafile (datafilekey, user_id, private, filetype, created_at, updated_at)
           VALUES(?, 0, 0, "csv", UTC_TIMESTAMP(), UTC_TIMESTAMP())`

	result, err := s.DB.Exec(stmt, datafilekey)
	if err != nil {
		return 0, err
	}
	id, err := result.LastInsertId()
	if err != nil {
		return 0, err
	}

	return int(id), nil
}

//InsertHeaders: Insert header information into the headers table. The code
func (s *hoppyDB) InsertHeaders(datafileId int, headers []string, datatype []string) (int, error) {
	stmt := `INSERT INTO uid_dataheaders (datafileid, headerid, header, datatype) VALUES (?, ?, ?, ?)`

	for index, val := range headers {
		_, err := s.DB.Exec(stmt, datafileId, index, val, datatype[index])
		if err != nil {
			return 0, err
		}
	}
	return len(headers), nil
}

func (s *hoppyDB) InsertData(maxHeaders int, datafileId int, data [][]string) error {
	stmt := `INSERT INTO uid_datacontent (datafileid, headerid, rowid, value)
					VALUES (?, ?, ?, ?)`

	for rowid, values := range data[1:] { //need to remove the header row
		for index, val := range values[:maxHeaders] { //only go until the max number of headers
			_, err := s.DB.Exec(stmt, datafileId, index, rowid, val)
			if err != nil {
				return err
			}
		}
	}

	return nil
}

func (s *hoppyDB) GetHeaders(datafilekey string) ([]Header, error) {
	var datafileid int
	metadatasql := `SELECT datafileid FROM uid_datafile WHERE datafilekey='%s'`
	metadatasql = fmt.Sprintf(metadatasql, datafilekey)
	row := s.DB.QueryRow(metadatasql)
	err := row.Scan(&datafileid)

	stmt := `SELECT headerid, header, datatype from uid_dataheaders where datafileid=? order by headerid`
	headers := make([]Header, 0)
	rows, err := s.DB.Query(stmt, datafileid)
	if err != nil {
		return nil, err
	}
	for rows.Next() {
		header := new(Header)
		err := rows.Scan(&header.Position, &header.Value, &header.DataType)
		if err != nil {
			return nil, err
		}
		headers = append(headers, *header)
	}
	return headers, nil
}

func (s *hoppyDB) GetData(datafilekey string) ([]DataContent, error) {
	var maxrowid int
	var datafileid int
	metadatasql := `SELECT datafileid FROM uid_datafile WHERE datafilekey='%s'`
	metadatasql = fmt.Sprintf(metadatasql, datafilekey)
	row := s.DB.QueryRow(metadatasql)
	err := row.Scan(&datafileid)
	if err != nil {
		fmt.Println("could not find metadata")
		return nil, err
	}

	rowstmt := `SELECT MAX(rowid) FROM uid_datacontent where datafileid=?`

	stmt := `SELECT headerid, rowid, value from uid_datacontent where datafileid=? order by headerid LIMIT 100;`

	datalist := make([]DataContent, 0)

	row = s.DB.QueryRow(rowstmt, datafileid)

	err = row.Scan(&maxrowid)

	if err != nil {
		fmt.Println("could not find max rowid for data file")
		return nil, err
	}

	rows, err := s.DB.Query(stmt, datafileid)
	if err != nil {
		return nil, err
	}

	for rows.Next() {
		data := new(DataContent)
		err := rows.Scan(&data.HeaderID, &data.RowID, &data.Value)
		if err != nil {
			return nil, err
		}
		datalist = append(datalist, *data)
	}
	return datalist, nil
}

```
Let's compile the application again and re-deploy the containers. Once that's done we should upload a file and retrieve its content from the new API end point we defined.

You should be able to query the end point with a curl command like
`curl -X GET localhost:3000/api/v1/<id>/data`

With that we have successfully built a golang web app that can parse a csv and turn it into an API.
The full code for this part is available [here](https://github.com/mihirkelkar/csvapi-article/tree/master/code-part-d)
