---
layout: default
---

# Instantly converting CSV files to an API using Golang - Part A

Off recently, I have been building a lot of projects that rely on ingesting a large amount of CSV data.
This shouldn't be a problem per-se for any big data processing framework but the data I wanted isn't available in bulk  at the same time.  

This meant that the data that I am looking for often arrives as small one-off files once or twice a week. Consuming this data in small quantities didn't make too much sense. Often, the format of the data and the headers of the CSV files available were also different. This meant that I had to build a general purpose tool that could ingest these files and make them available in a sort-of standardized format programatically.  

So I decided to build a tool that could read a CSV file and then make the file content available instantly via an API.

To do this I decided to setup a basic web-app using golang mux and a MySQL backend.  

In this part, I wanted to go over building the CSV processor for the app that we will eventually import as a package into the go mux web app.

I posted a public version of the tool [here](http://web-app.326wy59fvd.us-east-1.elasticbeanstalk.com/)  

## Building the CSV Processor.

Since I needed a way to distinguish CSV files, I decided to represent each CSV file as structure.

```
package csvprocessor

type csvFile struct {
	Headers     []string
	DataFileKey string
	Cols        int
	Rows        int
	Lines       [][]string
}
```

Each CSV file struct has attributes like its
* Headers
* A unique datafilekey that I generate
* the number of columns
* the number of rows
* the actual content of each row of this file.  


Now let's try and read a CSV file and convert it to the struct representation above. We can use the standard csv reader package available for golang and read the contents of the file into a variable called `lines`. The csv reader's `ReadAll`function returns a `[][]string` and it fits the representation of lines in the struct above well.  
```
package csvprocessor

....

func NewCsvUpload(filepath string) (*csvFile, error) {
	//multipart.File fulfils the io.Reader interface which is an input to csv
	//reader
	file, err := os.Open(filepath)
	csvReader := csv.NewReader(file)
	lines, err := csvReader.ReadAll()

	//debug statement
	if err != nil {
		return nil, err
	}
  //create a csvFile struct
  csvFile := &csvFile{}
  //set the header for the new csvFile struct
  //set the rows for the new csvFile struct.
  //set the unique data key for the new csv file struct.
	return csvFile, nil
}
```

We still need to set the headers, rows and a unique data key for the file that we read. To do that, we will be defining receiver functions on the csvFile struct.  

```
package csvprocessor

const (
	letterIdxBits = 6                    // 6 bits to represent a letter index
	letterIdxMask = 1<<letterIdxBits - 1 // All 1-bits, as many as letterIdxBits
	letterIdxMax  = 63 / letterIdxBits   // # of letter indices fitting in 63 bits
)

const letterBytes = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"


....

func GenerateDataFileKey() string {
	b := make([]byte, 10)
	var src = rand.NewSource(time.Now().UnixNano())
	for i, cache, remain := 9, src.Int63(), letterIdxMax; i >= 0; {
		if remain == 0 {
			cache, remain = src.Int63(), letterIdxMax
		}
		if idx := int(cache & letterIdxMask); idx < len(letterBytes) {
			b[i] = letterBytes[idx]
			i--
		}
		cache >>= letterIdxBits
		remain--
	}
	return string(b)
}

//Generate DataFileKey
func (csv *csvFile) GenerateDataFileKey() error {
	csv.DataFileKey = GenerateDataFileKey()
	return nil
}

func (csv *csvFile) SetHeaders() {
	csv.Headers = csv.Lines[0]
	csv.Cols = len(csv.Lines[0])
}

func (csv *csvFile) SetRows() {
	csv.Rows = len(csv.Lines)
}
```

Now, that we have defined functions to set headers, rows and a unique file key. We can now complete the original function `NewCsvUpload` to use the newly defined receiver functions. For good measure, we also defined additional receiver functions that can get the return the data of the csvFile structs.   

At this point, the file `csvprocessor.go` looks like this:  

```
package csvprocessor

import (
	"encoding/csv"
	"fmt"
	"os"
	"strings"
  "math/rand"
	"time"
)

const (
	letterIdxBits = 6                    // 6 bits to represent a letter index
	letterIdxMask = 1<<letterIdxBits - 1 // All 1-bits, as many as letterIdxBits
	letterIdxMax  = 63 / letterIdxBits   // # of letter indices fitting in 63 bits
)

const letterBytes = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

type csvFile struct {
	Headers     []string
	DataFileKey string
	Cols        int
	Rows        int
	Lines       [][]string
}


func NewCsvUpload(filepath string) (*csvFile, error) {
	//multipart.File fulfils the io.Reader interface which is an input to csv
	//reader
	file, err := os.Open(filepath)
	csvReader := csv.NewReader(file)
	lines, err := csvReader.ReadAll()
	//debug statement
	if err != nil {
		return nil, err
	}
	csvFile := &csvFile{Lines: lines}
	csvFile.SetHeaders()
	csvFile.SetRows()
	csvFile.GenerateDataFileKey()
	return csvFile, nil
}


func GenerateDataFileKey() string {
	b := make([]byte, 10)
	var src = rand.NewSource(time.Now().UnixNano())
	for i, cache, remain := 9, src.Int63(), letterIdxMax; i >= 0; {
		if remain == 0 {
			cache, remain = src.Int63(), letterIdxMax
		}
		if idx := int(cache & letterIdxMask); idx < len(letterBytes) {
			b[i] = letterBytes[idx]
			i--
		}
		cache >>= letterIdxBits
		remain--
	}
	return string(b)
}

//Generate DataFileKey
func (csv *csvFile) GenerateDataFileKey() error {
	csv.DataFileKey = GenerateDataFileKey()
	return nil
}

func (csv *csvFile) SetHeaders() {
	csv.Headers = csv.Lines[0]
	csv.Cols = len(csv.Lines[0])
}

func (csv *csvFile) SetRows() {
	csv.Rows = len(csv.Lines)
}

//GetHeaders: Returns the csv headers from the processed file.
func (csv *csvFile) GetHeaders() []string {
	return csv.Headers
}

func (csv *csvFile) GetData() [][]string {
	return csv.Lines
}

func (csv *csvFile) GetDataFileKey() string {
	return csv.DataFileKey
}
```
`
This file can now be used as a package to be imported into a web-app to parse csv files and convert them to an intermediate go struct representation.

In part B, we will build the basics of a golang web-app using Docker and link it to a MySQL database. We will
setup the correct schema for this database and then work to build a function that can parse an uploaded file
and store it in the MySQL database.
