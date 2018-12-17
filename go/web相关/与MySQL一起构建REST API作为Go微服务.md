# 与MySQL一起构建REST API作为Go微服务

#### 设置API

首先要做的事情之一就是选择一个路由包。路由是将URL连接到可执行功能的内容。该[MUX包](https://github.com/gorilla/mux)一直担任我的*路由*，但也有其他的替代方案比如[httprouter](https://github.com/julienschmidt/httprouter)和[pat](https://github.com/bmizerany/pat)。我将在本指南中使用多路复用器。

为简单起见，我们将创建一个打印出消息的单个端点。

```go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/mux"
)

func setupRouter(router *mux.Router) {
	router.
		Methods("POST").
		Path("/endpoint").
		HandlerFunc(postFunction)
}

func postFunction(w http.ResponseWriter, r *http.Request) {
	log.Println("You called a thing!")
}

func main() {
	router := mux.NewRouter().StrictSlash(true)

	setupRouter(router)

	log.Fatal(http.ListenAndServe(":8080", router))
}
```

上面的代码创建了一个**路由器**，将URL与[可执行函数](https://golang.org/pkg/net/http/#HandlerFunc)相关联 -  在我们的例子中是*postFunction* - 并使用该路由器在端口*8080*上启动服务器。很容易，对吧？🤠

#### 数据库连接

让我们通过将它连接到MySQL数据库来构建此代码。Go 为SQL数据库提供了一个[接口](https://golang.org/pkg/database/sql/)，但需要一个驱动程序。我在这个例子中使用[go-sql-driver](https://github.com/go-sql-driver/mysql)。

```go
package db

import (
	"database/sql"

	_ "github.com/go-sql-driver/mysql"
)

func CreateDatabase() (*sql.DB, error) {
	serverName := "localhost:3306"
	user := "myuser"
	password := "pw"
	dbName := "demo"

	db, err := sql.Open("mysql", user+":"+password+"@tcp("+serverName+")/"+dbName+"?&charset=utf8mb4&collation=utf8mb4_unicode_ci&parseTime=true&multiStatements=true")
	if err != nil {
		return nil, err
	}

	return db, nil
}
```

该代码被放置在另一个包，叫做`db`， 并假定有上运行的数据库*本地主机：3306*有一个名为数据库*的演示*。返回的数据库自动处理数据库的连接池。

让我们从前面的代码片段更新*postFunction*，以使用此数据库。

```go
func postFunction(w http.ResponseWriter, r *http.Request) {
	database, err := db.CreateDatabase()
	if err != nil {
		log.Fatal("Database connection failed")
	}

	_, err = database.Exec("INSERT INTO `test` (name) VALUES ('myname')")
	if err != nil {
		log.Fatal("Database INSERT failed")
	}

	log.Println("You called a thing!")
}
```

真的是这样的！它相当简单，但上面的代码存在一些问题，并且缺少一些有趣的东西。这是一个有点棘手的地方，但不要跳船！⚓️

#### 结构和依赖性

如果您已经检查过上面的代码，您可能已经注意到我们正在为每个API调用打开数据库。即使打开的数据库[对于并发使用也是安全的](https://golang.org/pkg/database/sql/#Open)。我们需要一些依赖管理来确保我们只打开一次数据库，为此，我们想要使用一个`struct`。

```go
package app

import (
	"database/sql"
	"log"
	"net/http"

	"github.com/gorilla/mux"
)

type App struct {
	Router   *mux.Router
	Database *sql.DB
}

func (app *App) SetupRouter() {
	app.Router.
		Methods("POST").
		Path("/endpoint").
		HandlerFunc(app.postFunction)
}

func (app *App) postFunction(w http.ResponseWriter, r *http.Request) {
	_, err := app.Database.Exec("INSERT INTO `test` (name) VALUES ('myname')")
	if err != nil {
		log.Fatal("Database INSERT failed")
	}

	log.Println("You called a thing!")
	w.WriteHeader(http.StatusOK)
}
```

我们首先创建一个名为*app*的新包来托管我们的struct及其[methods](https://gobyexample.com/methods)。我们的**App**结构有两个字段; 在*ln 17*和*ln 24*访问的*路由器*和*数据库*。我们还在方法结束时在*ln 30*处手动设置返回的状态代码。

主包和函数还需要进行一些更改才能使用新的**App**结构。我们从这个包中删除了*postFunction和setupRouter*函数，因为它现在位于*app*包中。我们留下：

```go
package main

import (
	"log"
	"net/http"

	"github.com/gorilla/mux"
	"github.com/johan-lejdung/go-microservice-api-guide/rest-api/app"
	"github.com/johan-lejdung/go-microservice-api-guide/rest-api/db"
)

func main() {
	database, err := db.CreateDatabase()
	if err != nil {
		log.Fatal("Database connection failed: %s", err.Error())
	}

	app := &app.App{
		Router:   mux.NewRouter().StrictSlash(true),
		Database: database,
	}

	app.SetupRouter()

	log.Fatal(http.ListenAndServe(":8080", app.Router))
}
```

为了利用我们的新结构，我们打开了一个数据库和一个新的路由器。然后我们将它们插入到新的*App*结构的字段中。

恭喜！您现在已连接到您的数据库，该数据库将同时用于所有传入的API调用🙌

最后一步，我们将在路由器设置中添加*GET* -Method并以*JSON*格式返回数据。我们首先添加一个结构来填充我们的数据，并将字段映射到*JSON*。

```go
package app

import (
	"time"
)

type DbData struct {
	ID   int       `json:"id"`
	Date time.Time `json:"date"`
	Name string    `json:"name"`
}
```

我们通过扩展*app.go*文件，使用新方法*getFunction*来获取并将数据写入客户端响应。最终文件看起来像这样。

```go
package app

import (
	"database/sql"
	"encoding/json"
	"log"
	"net/http"

	"github.com/gorilla/mux"
)

type App struct {
	Router   *mux.Router
	Database *sql.DB
}

func (app *App) SetupRouter() {
	app.Router.
		Methods("GET").
		Path("/endpoint/{id}").
		HandlerFunc(app.getFunction)

	app.Router.
		Methods("POST").
		Path("/endpoint").
		HandlerFunc(app.postFunction)
}

func (app *App) getFunction(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	id, ok := vars["id"]
	if !ok {
		log.Fatal("No ID in the path")
	}

	dbdata := &DbData{}
	err := app.Database.QueryRow("SELECT id, date, name FROM `test` WHERE id = ?", id).Scan(&dbdata.ID, &dbdata.Date, &dbdata.Name)
	if err != nil {
		log.Fatal("Database SELECT failed")
	}

	log.Println("You fetched a thing!")
	w.WriteHeader(http.StatusOK)
	if err := json.NewEncoder(w).Encode(dbdata); err != nil {
		panic(err)
	}
}

func (app *App) postFunction(w http.ResponseWriter, r *http.Request) {
	_, err := app.Database.Exec("INSERT INTO `test` (name) VALUES ('myname')")
	if err != nil {
		log.Fatal("Database INSERT failed")
	}

	log.Println("You called a thing!")
	w.WriteHeader(http.StatusOK)
}
```

#### 数据库迁移

我们将为项目添加一个最终添加项。当数据库与应用程序或服务紧密结合时，您可以通过正确处理所述数据库的迁移来节省您*难以理解*的头痛*程度*。我们将使用[migrate](http://github.com/golang-migrate/migrate)，并扩展我们的*db包*。

在数据库打开之后，我们将向*migrateDatabase*添加另一个函数调用，该函数将执行迁移过程。

我们也将会在过程中添加MigrationLogger结构来处理记录，如看到[这里](https://github.com/johan-lejdung/go-microservice-api-guide/blob/master/rest-api/db/migrationlogger.go)，并在使用*LN 45*。

迁移从正常的sql-queries执行。从*ln 37*处看到的文件夹中读取迁移文件。

每次打开数据库时，都将应用所有未应用的数据库迁移。

这与一个包含数据库的*docker-compose*文件相结合，使得在多台机器上的开发变得简单。

```go
package db

import (
	"database/sql"
	"fmt"
	"log"
	"os"

	_ "github.com/go-sql-driver/mysql"
	"github.com/golang-migrate/migrate"
	"github.com/golang-migrate/migrate/database/mysql"
	_ "github.com/golang-migrate/migrate/source/file"
)

func CreateDatabase() (*sql.DB, error) {
	// The setup ...

	if err := migrateDatabase(db); err != nil {
		return db, err
	}

	return db, nil
}

func migrateDatabase(db *sql.DB) error {
	driver, err := mysql.WithInstance(db, &mysql.Config{})
	if err != nil {
		return err
	}

	dir, err := os.Getwd()
	if err != nil {
		log.Fatal(err)
	}

	migration, err := migrate.NewWithDatabaseInstance(
		fmt.Sprintf("file://%s/db/migrations", dir),
		"mysql",
		driver,
	)
	if err != nil {
		return err
	}

	migration.Log = &MigrationLogger{}

	migration.Log.Printf("Applying database migrations")
	err = migration.Up()
	if err != nil && err != migrate.ErrNoChange {
		return err
	}

	version, _, err := migration.Version()
	if err != nil {
		return err
	}

	migration.Log.Printf("Active database version: %d", version)

	return nil
}
```

#### 把它包起来

所以你已经在这里完成了👏👏

一个不可部署的微服务是没用的，因此我们将添加一个Dockerfile来打包应用程序以便于分发 -  *然后我保证让你离开。*

```bash
FROM golang:1.11 as builder
WORKDIR $GOPATH/src/github.com/johan-lejdung/go-microservice-api-guide/rest-api
COPY ./ .
RUN GOOS=linux GOARCH=386 go build -ldflags="-w -s" -v
RUN cp rest-api /

FROM alpine:latest
COPY --from=builder /rest-api /
CMD ["/rest-api"]
```



完整代码地址：[完整代码](https://github.com/johan-lejdung/go-microservice-api-guide/tree/master/rest-api)





