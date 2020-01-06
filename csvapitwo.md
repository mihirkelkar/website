---
layout: default
---

# Instantly convert CSV files to an API using Golang - Part B

* A full demo of this tool with a UI is available [here](http://web-app.326wy59fvd.us-east-1.elasticbeanstalk.com/)
* [Part A](./csvapione.html)
* [Part C](./csvapithree.html)
* [Part D](./csvapifour.html)

All the code for part-B is available [here](https://github.com/mihirkelkar/csvapi-article/tree/master/code-part-b)  

In part A, we built an importable package that could parse CSV files and convert them to an intermediate struct representation. In this part we are going to setup a basic Golang web-app using mux and then link it to a MySQL database. The web-app will eventually be able to upload csv files, parse them and store the contents into the database. To do local development, we will be using docker.

For now, lets start with creating the basic structure of our golang web-app. Lets create a new directory and `cd` into the new directory.

```
mkdir csvapi && cd "$_"
```

This newly created directory will be the root of our golang web-app. Instead of going through the complicated process of setting up a go development environment, we are going to use go-modules and a docker container to build and test our code.

To initialize a go module, we need a unique path name. I use the format `github.com/<user-name>/<dir-path>`

```
go mod init github.com/<user-name>/csvapi
```

Once you execute this command, you should have a `go.mod` file present in the directory. If you type in the command `go mod verify`, it should print a message like `all modules verified`.

Now, lets go ahead and create a basic directory structure for our go web-app. Create a folder called `cmd` in the root folder. Within the `cmd` folder create a folder called `web`. In `web` lets create a file called `main.go`.

Your directory structure should look like this :
```
├── cmd
│   ├── web
│       ├── main.go
├── go.mod

```

Let's go ahead and edit to the `main.go` file and add the following code :
```
package main

import (
	"fmt"
	"net/http"

	"github.com/gorilla/mux"
)

func HelloWorld(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "This is a hello from the golang web app")
}

func main() {
	//define the command line arguments.
	r := mux.NewRouter()
	r.HandleFunc("/", HelloWorld).Methods("GET")
	http.ListenAndServe(":3000", r)
}
```

The main function in the above file is the entry point to the web-app. At the moment, all it's doing is defining a new router and then pointing the `/` path on the router to a function called `HelloWorld`. The function `HelloWorld` in turn just writes a line to the webpage. _We will be re-factoring how the routes and their handlers are organized as the application matures, but this works for now_

Now, lets define a Dockerfile to run this web-app. Move back into the root directory of the web-app and create a new file called `Dockerfile`. Add the following lines to the file.

```
FROM golang as builder

ENV GO11MODULE=on

WORKDIR /app

COPY go.mod .

COPY go.sum .

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ./cmd/web

EXPOSE 3000

CMD ./web
```

In a nutshell: the docker file defines the guidelines for building a docker container. In this case, it defines golang as its base, then copies over the contents of the working directory, builds the application and runs it.

Now, lets build the web-app and run it.

```
docker build -t csvapi:v001 .
docker run -d -p 3000:3000 -it csvapi:v001
```

If you go to localhost:3000, you should see `This is a hello from the golang web app` printed out on the webpage. Voila your golang web-page is now live !!

To stop the container, go ahead and run  
```
docker stop <container-id>
```
The container id is the long alphanumeric key printed on the terminal after the docker run commmand.

Now let's go ahead and attach a MySQL database to the Golang Service by creating a docker-compose file. A docker-compose file lets you run two or more docker containers simultaneously that can interact and depend on each other. So in the root directory of the app, create a file called `docker-compose.yml`. Add the following code to the file.  

```
version: '3'

services:
 db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: 'hoppy'
      # So you don't have to use root, but you can if you like
      MYSQL_USER: 'web'
      # You can use whatever password you like
      MYSQL_PASSWORD: 'test'
      # Password for root access
      MYSQL_ROOT_PASSWORD: 'test'
    ports:
      # <Port exposed> : < MySQL Port running inside container>
      - '3306:3306'
    expose:
      - '3306'

      # Where our data will be persisted
    volumes:
      - my-datavolume:/var/lib/mysql
 goapp:
    build: .
    ports:
      - "3000:3000"

    links:
      - db

volumes:
  my-datavolume:
```

The `docker-compose.yml` file describes a services section which includes the `goapp` which is our web-app and a `db` which builds off of a `mysql:5.7` docker image. Note how we link the `db` service to the `goapp` service which allows both containers to communicate on the same docker-network. We also define access credentials for the mysql container image in the docker file. Now that we have the docker-compose file ready, lets test the application and the database together.

```
docker-comose build --no-cache
docker-compose up
```

Note: _`docker-compose up` might fail if you haven't stopped your go-app from the previous stage. You can lookup the container id for that using the `docker ps` command. once you stop the old container, you should also run `docker-compose stop` and then repeat the above two commands again_

if the `docker-compose up` command works without any errors and you can access your app at `localhost:3000`, you have successfully created a database and a go-app on the same docker network. There is however no code in the app to interact with the database at the moment.

Let's go back to the `cmd/web/main.go` file and add the following code changes to it:
```
package main

import (
	"database/sql"
	"fmt"
	"net/http"
	"os"

	_ "github.com/go-sql-driver/mysql"
	"github.com/gorilla/mux"
)

func HelloWorld(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprint(w, "This is a hello from the golang web app")
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

We added a `GetDataBaseUrl` function that reads the MySQL credentials from environment variables and builds a connection string. Then in the main function, we open a socket to the database and then try to open a connection to it. If the connection fails, we print out an error; if not we print out "success" on the terminal.

Since the `GetDataBaseUrl` function reads environment variables, we need to set those in the `Dockerfile`. Edit your `Dockerfile` to look like this

```
FROM golang as builder

ENV GO11MODULE=on

WORKDIR /app

COPY go.mod .

COPY go.sum .

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ./cmd/web

ENV USER=web


ENV PASS=test


ENV URL=db


ENV DB=hoppy


EXPOSE 3000

CMD ./web

```

Note how we used the name of the db container instead of localhost to indicate the URL of the mysql database.
Save the `Dockerfile` and re-build the containers using `docker-compose build` and run them using `docker-compose up`.

If you see `success` printed out on the terminal in the logs, you have successfully linked the database to your web-app.

The next steps now are to create the required tables in the MySQL database. We need three main tables for the database.
* uid_datafile : contains metadata around the file itself.
* uid_dataheaders : contains the headers from the files
* uid_datacontent: contains the actual data from the files

The schema for these tables will look like this :

```
create table if not exists uid_datafile(
  datafileid int not null auto_increment,
  datafilekey varchar(500) not null,
  user_id int,
  private boolean,
  filetype varchar(55),
  created_at timestamp default current_timestamp,
  updated_at timestamp default current_timestamp,
  deleted boolean default false,
  primary key(datafileid)
  );

create table uid_dataheaders(
  datafileid int not null,
  headerid int not null,
  header varchar(255) not null,
  datatype varchar(255) not null,
  primary key(datafileid, headerid)
);

create table uid_datacontent(
    datafileid int not null,
    headerid int not null,
    rowid int not null,
    value varchar(1000),
    primary key(datafileid, headerid, rowid)
    );
```

We need to log into the MySQL database and create these tables. Make sure that your app and database are still running, then type in `docker ps` into a new terminal and find the container id associated with the `mysql 5.7` docker image. Lets log into this image by typing `docker exec -it <container-id> /bin/bash`

Once on the terminal of the docker container, type in `mysql -p` and enter in `test` as the password. This can be found in the docker-compose file and you can change it in that file. Then switch the database to hoppy by typing in `use hoppy;`. Then run the create table scripts in the code block above to create the tables. You can verify that the tables were created by running the `show tables` command on mysql.

With this, we have so far :
* Created a working go web-app
* Create a working database.
* Linked the app to the database.
* Created the tables we need to store the csv file data.

In the next part, we will write the handler functions required to parse an uploaded file and store it in the databases as well as the API handlers required for returning the data.

All the code for part-B is available [here](https://github.com/mihirkelkar/csvapi-article/tree/master/code-part-b)  

[Part C](./csvapithree.html)
