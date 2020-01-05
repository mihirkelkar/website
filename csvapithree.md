---
layout: default
---

# Instantly convert CSV files to an API using Golang - Part C

* A full demo of this tool with a UI is available [here](http://web-app.326wy59fvd.us-east-1.elasticbeanstalk.com/)
* [Part A](./csvapione.html)
* [Part B](./csvapitwo.html)
* [Part D](./csvapifour.html)

In part A, we built the package to parse a csv file. In part B: we setup the database and a golang web-app that can be used to parse and store these files. In this part, we will develop an API end-point that can upload a file, parse it and store it in the database.

Let's start with adding a new route to our application. In `cmd/web/main.go`. Add the following changes.

```
...


r := mux.NewRouter()
r.HandleFunc("/", HelloWorld).Methods("GET")
r.HandleFunc("/", UploadFile).Methods("POST")  //we added a new POST method on the / route.
http.ListenAndServe(":3000", r)
```

We added a new POST route to the `/` path. The POST API call should eventually trigger a function called UploadFile which we will define next.

```
func UploadFile(w http.ResponseWriter, r *http.Request) {
	fmt.Println("File Upload Endpoint Hit")
	// Parse our multipart form, 10 << 20 specifies a maximum
	// upload of 10 MB files.
	r.ParseMultipartForm(10 << 20)
	// it also returns the FileHeader so we can get the Filename,
	// the Header and the size of the file
	file, header, err := r.FormFile("myFile")
	if err != nil {
		fmt.Println("Error Retrieving the File")
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

	w.WriteHeader(http.StatusOK)
	return
}
```

The function is very rudimentary in nature. At its core, the function retrieves the file from the request, then it writes the file to a folder on the web-server. Once, a copy of the file is written to the web-server it return a 200 status code. If there is an error along the way it should return a 500 and prints an error message on the terminal. In production ready systems, we would typically write the error to an errorLog. However, we can make those changes at the end of the application for now.

The entire `cmd/web/main.go` file should look like
```
package main

import (
	"database/sql"
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"

	_ "github.com/go-sql-driver/mysql"
	"github.com/gorilla/mux"
)

func HelloWorld(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "This is a hello from the golang web app")
}

func UploadFile(w http.ResponseWriter, r *http.Request) {
	fmt.Println("File Upload Endpoint Hit")
	// Parse our multipart form, 10 << 20 specifies a maximum
	// upload of 10 MB files.
	r.ParseMultipartForm(10 << 20)
	// it also returns the FileHeader so we can get the Filename,
	// the Header and the size of the file
	file, header, err := r.FormFile("myFile")
	if err != nil {
		fmt.Println("Error Retrieving the File")
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

	w.WriteHeader(http.StatusOK)
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

	//define the command line arguments.
	r := mux.NewRouter()
	r.HandleFunc("/", HelloWorld).Methods("GET")
	http.ListenAndServe(":3000", r)
}
```
Let's re-build and deploy the docker containers. Once that is complete, you can check that the new upload end point is working by using the following command to upload a file to the web-server

```
curl -v -X POST -F "myFile=@<yourfile>" localhost:3000
```
_Note: The file path in this command needs to be absolute_

If you get a 200 response back, that means that the file was successfully uploaded. Next, lets build a model for the database tables that we created in part B.

In the `pkg` folder, lets create a folder called `mysql`. Let's now create a file called `hoppy.go` in it.
Lets add the following code to the file

```
package mysql

import "database/sql"

type hoppyDB struct {
	DB *sql.DB
}

func NewHoppyDB(db *sql.DB) *hoppyDB {
	return &hoppyDB{DB: db}
}
```

We have here defined a struct that can hold a database connection. We have also created a function that can return an instance of this struct when the function is invoked. The idea being that we can pass our open connection to the MySQL db and make it a part of this struct.

Then we can define receiver functions on this struct that can accept a csvFile as an input and use the database connection to write the file's content to the database.

let's define the first function that can write metadata about this file to the `uid_datafile` table we created in part B.

```
//Insert: Inserts the metadata about the file into the database.
func (s *hoppyDB) InsertMetaData(datafilekey string) (int, error) {

	stmt := `INSERT INTO  uid_datafile (datafilekey, user_id, private, filetype, created_at, updated_at)
           VALUES(?, 0, 'false', "csv", UTC_TIMESTAMP(), UTC_TIMESTAMP())`

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
```

This function is pretty simple. It accepts a datafilekey as an input and then runs an insert command on the database using the passed parameters.

Let's define two more functions that can insert data into the `uid_dataheaders` and `uid_datacontent` tables.
```
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
```

This function accepts a DataFileID, a list of headers and their corresponding data types. The function runs an insert statement and inserts each header into the table.
so an input like `fileone, [header_one, header_two], [string, string]` will translate into database rows like:
```
 datafileID   header_index  	values		 datatype
 fileone 			0 							header_one string
 fileone 			1 							header_two string
 ...
```

The following function accepts the data parsed from the csv file and then inserts it into the database.
```
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
```

This function iterates over the list of csv rows and then inserts each row into the database. As an example
an input of `maxHeaders int, datafileId int, data [][]string` like `5 fileone [[xx, yy, zz], [aa, bb, cc]]`

would be written to the database like :
```
datafileID  rowNumber		headerIndex		value
fileone 		0 					0							xx
fileone			0						1							yy
fileone			0						2							zz
fileone			1						0							aa
fileone			1						1							bb
fileone			1						2							cc
```

The entire file should look like this :

```
package mysql

import "database/sql"

type hoppyDB struct {
	DB *sql.DB
}

func NewHoppyDB(db *sql.DB) *hoppyDB {
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

```

As a general practice when dealing with databases, you don't want to expose the functions that interact with the database directly to the application code. To avoid that, lets develop an interface. Think of an interface like a contract you can define with function signatures. Any struct that can implement these function signatures can fit an interface. In return, you can now use the interface as a way to expose your hidden struct and its functions to other parts of your codebase without revealing their functionality.

Let's go ahead and define the `HoppyDB` interface.

```
type HoppyDB interface{
	InsertMetaData(string) (int, error)
	InsertHeaders(int, []string, []string) (int, error)
	InsertData(int, int, [][]string) error
}
```

This interface defines 3 function signatures that any structure needs to fit to fulfill the contract. We have already defined these function on our hoppyDB struct. So the hoppyDB struct complies with the defined interface. We just need to change the `NewHoppyDB` function to return an interface instead of the struct itself. This can be done by just changing the functions return type.

```
func NewHoppyDB(db *sql.DB) HoppyDB {
	return &hoppyDB{DB: db}
}
```
We simply changed the return parameter from `*hoppyDB` to `HoppyDB` here.

The processing logic of this application is mostly ready at this point. We need to import it into our main.go file now. We need to do 3 main things in the main file now.
* Create an instance of `HoppyDB`
* Give the `UploadFile` function access to HoppyDB so it can insert the data.

In order to do this, lets define a new struct in the main file that acts like a wrapper for the HoppyDB interface. Add the following code to `cmd/web/main.go`

```
import msql "github.com/mihirkelkar/csvapi/pkg/mysql"

type Application struct{
	ModelService msql.HoppydB
}
```

In order to give the `UploadFile` function access to the database, lets make that a receiver function on the `Application` struct by changing its function definition to look like :

```
func (app *Application) UploadFile(w http.ResponseWriter, r *http.Request){
	.....
}
```
In the main function of the file, we need to create an application struct and then change the route to call the receiver function version of uploadfile to comply with the change we made above. The function would eventually look like this:
```
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
	http.ListenAndServe(":3000", r)
}
```

We now need to integrate our parser with the `uploadFile` file function. To do that, lets create a `csvprocessor` folder within the `pkg` folder and move the file from Part - A to the folder.

Then let's the `uploadFile` function and add the following code to it.

```

....
csvFile, err := cpros.NewCsvUpload("./data/" + filename)
if err != nil {
	fmt.Println(err.Error())
	w.WriteHeader(http.StatusInternalServerError)
	return
}

//Insert the metadata into the database.
lastID, err := app.ModelService.InsertMetaData(csvFile.GetDataFileKey())
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
return
}
```

Once the file is uploaded, we make a call to the csvprocessor's `NewUploadedFile` function create a intermediate representation of the csvFile. Then we call the HoppyDB functions exposed via the interface to insert the metadata, headers and content into the database.

The full `main.go` file looks like this:
```
package main

import (
	"database/sql"
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

func HelloWorld(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "This is a hello from the golang web app")
}

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
	datakey := csvFile.GetDataFileKey()
	//Insert the metadata into the database.
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
	http.ListenAndServe(":3000", r)
}
```

Let's re-compile and test this application and test it out. If you get a 200 Status code on your Curl request. You successfully uploaded, parsed and stored a csv file in a database. Note that the file ID gets printed out on the terminal after a successful upload.

In part D, we will build the API end points and handlers that will return this stored data for us. The entire codebase until this point is available [here](https://github.com/mihirkelkar/csvapi-article/tree/master/code-part-c)
