---
title: A simple go, gin, and sqlc combination walkthrough
description: A simple go, gin, and sqlc combination walkthrough
date: 2023-05-07
draft: false
---


Well, in this post we will see a partial implementation of a medium.com-like app called [Conduit](https://github.com/gothinkster/realworld). Take a brief look at the backend implementation [specs](https://www.realworld.how/docs/specs/backend-specs/introduction). As you can see, there are APIs for users, articles, tags, and so on. For now, we will implement user registration and login endpoints; api/users and api/users/login. In the end, we will be able to execute the two curl requests successfully:
```bash
    # 1. user registration request
    curl --location 'http://localhost:8085/api/users' \
            --header 'Content-Type: application/json' \
            --data-raw '{
                   "user": {
                       "username": "johndoe",
                       "email": "johndoe@example.com",
                       "password": "johndoepassword"
                    }
              }'  |  jq
    
    # response
    # {
    #   "user": {
    #     "username": "johndoe",
    #     "email": "johndoe@example.com",
    #     "bio": null,
    #     "image": null,
    #     "token": "[REDACTED]"
    #   }
    # }
    
    # 2. user login request
    curl --location 'http://localhost:8085/api/users/login' \
            --header 'Content-Type: application/json' \
            --data-raw '{
                   "user":{
                       "email": "johndoe@example.com",
                       "password": "johndoepassword"
                   }
              }'  |  jq
    # response
    # {
    #   "user": {
    #     "username": "johndoe",
    #     "email": "johndoe@example.com",
    #     "bio": null,
    #     "image": null,
    #     "token": "[REDACTED]"
    #   }
    # }

### Preparing necessary binaries and project structure
```
We will use [sqlc](https://github.com/kyleconroy/sqlc), a powerful SQL compiler, to convert raw SQL queries into go code. Install it with:
```bash
    go install github.com/kyleconroy/sqlc/cmd/sqlc@latest
```
Secondly, we use [Task](https://github.com/go-task/task), an amazing task runner and also a simple Make alternative, to run tasks obviously. Install it using the below command:
```bash
    go install github.com/go-task/task/v3/cmd/task@latest
```
Thirdly, we use [golang-migrate](https://github.com/golang-migrate/migrate), to migrate our SQL queries:
```bash
    cd $HOME/Downloads
    curl -OL https://github.com/golang-migrate/migrate/releases/download/v4.15.2/migrate.linux-amd64.tar.gz
    tar -xf migrate.linux-amd64.tar.gz --one-top-level
    mv migrate.linux-amd64/migrate $HOME/go/bin
    rm -rf migrate.linux-amd64 migrate.linux-amd64.tar.gz
    
    # if successful, you should see the version
    migrate --version
    # 4.15.2
```
Now we prepare the project structure:
```bash
    # create a project folder and cd into it
    mkdir -p $HOME/go/src/conduit && cd $HOME/go/src/conduit 
    
    # open vscode in the current folder
    code . 
    
    # initialize go module
    go mod init conduit
    
    # create some initial folders
    mkdir -p db/{migration,query,sqlc} api config env 
    
    # you will have the following look on tree command
    tree
    # .
    # ├── api
    # ├── config
    # ├── db
    # │   ├── migration
    # │   ├── query
    # │   └── sqlc
    # ├── env
    # └── go.mod

Create docker-compose.yaml with the following definition since we use PostgreSQL:
```yaml
    version: '3.8'
    
    services:
      postgres:
        image: postgres
        environment:
          POSTGRES_DB: conduitdb
          POSTGRES_USER: demouser
          POSTGRES_PASSWORD: demopassword
        ports:
          - 5432:5432
        expose:
          - 5432          
        networks:
          - postgres
        restart: unless-stopped
```    
### Initial data modeling

By looking at the [spec](https://www.realworld.how/docs/specs/backend-specs/api-response-format#users-for-authentication), we can formulate our first table users . Before that let’s create our first migration files:
```bash
    migrate create -ext sql -dir db/migration -seq create_users_table
    # this will create the two files
    tree db/migration
    # db/migration
    # ├── 000001_create_users_table.down.sql
    # └── 000001_create_users_table.up.sql
```
Add some statements into 000001_create_users_table.up.sql like:
```sql
    CREATE TABLE IF NOT EXISTS "users" (
      "id" text not null,
      "username" text not null unique,
      "email" text not null unique,
      "password" text not null,
      "bio" text,
      "image" text,
      "created_at" timestamptz not null default now(),
      "updated_at" timestamptz not null default now(),
    
      PRIMARY KEY (id)
    );
    
    CREATE INDEX IF NOT EXISTS "idx_users_username" ON "users" ("username");
    CREATE INDEX IF NOT EXISTS "idx_users_email" ON "users" ("email");
```
We added extra columns; id , created_at , updated_at and as you might’ve noticed the column id is not autoincremented. In fact, we could use type uuid , but it is too long so I prefer [oklog/ulid](https://github.com/oklog/ulid) and [rs/xid](https://github.com/rs/xid) packages.

And also 000001_create_users_table.down.sql should be like this:
```sql
    DROP TABLE IF EXISTS "users";
```
Now we can either migrate up using migrate cli tool or programmatically; let’s try cli way first:
```bash
    # create postgres container
    docker compose up -d 
    # run migration up
    export POSTGRESQL_URL='postgres://demouser:demopassword@localhost:5432/conduitdb?sslmode=disable'  
    migrate -database ${POSTGRESQL_URL} -path db/migration up  
```
The above creates users table. At this point, I normally would set up a database connection on pgAdmin since I would work with raw SQL queries a lot (prototyping, analyzing queries, and so on). In case you want to install pgAdmin:
```bash
    cd $HOME/Downloads
    curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg
    sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list && apt update'
    sudo apt install pgadmin4-desktop
```
### Implementing endpoints

Now it is time to implement user authentication: registration and login. Before that create two files:
```bash
    touch api/server.go api/user_handler.go
```
Create a Server struct and its helper methods in api/server.go:
```go
    package api
    
    import "github.com/gin-gonic/gin"
    
    type Server struct {
       router *gin.Engine
    }
    
    func NewServer() *Server {
       server := &Server{
          router: gin.Default(),
       }
       return server
    }
    
    func (s *Server) MountHandlers() {
       api := s.router.Group("/api")
       api.POST("/users", s.RegisterUser)    // TODO: implement RegisterUser
       api.POST("/users/login", s.LoginUser) // TODO: implement LoginUser
    }
    
    func (s *Server) Start(addr string) error {
       return s.router.Run(addr)
    }

And registration and login handlers go into api/user_handler.go :

    package api
    
    import "github.com/gin-gonic/gin"
    
    // user response type, common to both handlers
    type userResponse struct {
       User struct {
          Username string  `json:"username"`
          Email    string  `json:"email"`
          Bio      *string `json:"bio"`
          Image    *string `json:"image"`
          Token    string  `json:"token"`
       } `json:"user"`
     }
    
    // user registration handler starts here
    type userRegisterReq struct {
       User struct {
          Username string `json:"username" binding:"required"`
          Email    string `json:"email" binding:"required,email"`
          Password string `json:"password" binding:"required"`
       } `json:"user"`
    }
    
    func (s *Server) RegisterUser(c *gin.Context) { // TODO: POST /users - RegisterUser
       panic("not implemented")
    }
    
    // user login handler starts here
    type userLoginReq struct {
       User struct {
          Email    string `json:"email" binding:"required,email"`
          Password string `json:"password" binding:"required"`
         } `json:"user"`
      }
    
    func (s *Server) LoginUser(c *gin.Context) {  // TODO: POST /users/login - LoginUser
       panic("not implemented")
    }
```
Since we know the request and response types of those endpoints from the [spec](https://www.realworld.how/docs/specs/backend-specs/endpoints/), I created the corresponding structs; userRegisterReq, userLoginReq and userResponse. Now I will take you through RegisterUser implementation steps:
```go
    func (s *Server) RegisterUser(c *gin.Context) { // TODO: POST /users - RegisterUser
        //  1. First bind and validate userRegisterReq
        //  2. Second prepare a user data before storing it in database;
        //     for example, generating unique id for each user.
        //  3. Store the user data in database
        //  4. Lastly construct userResponse data to send out
    }
```
At first sight, we can be sure that the third step requires injecting a database connection into this method and having some sort of CreateUser method that belongs to that connection. Let’s generate that method using sqlc which requires at least two files; sqlc.yaml and and user.sql:
```bash
    touch sqlc.yaml db/query/user.sql
```
sqlc uses sqlc.yaml as a configuration file to generate go code and I will explain it later:
```yaml
    version: "2"
    
    sql:
      - engine: "postgresql"
        queries: "./db/query"
        schema: "./db/migration"
        gen:
          go:
            package: "db"
            sql_package: "pgx/v4"
            out: "./db/sqlc"
            emit_interface: true                 
            emit_json_tags: true                 
            emit_pointers_for_null_types: true
            emit_result_struct_pointers: true
```
And db/query/user.sql contains queries to retrieve, save, update, and delete user-related data. Add the below statement:
```sql
    -- name: CreateUser :one
    INSERT INTO users (
        id,
        username,
        email,
        password 
    ) VALUES (
        $1,
        $2,
        $3,
        $4
    ) 
    RETURNING *;
```
The comment— name: CreateUser :one is important for sqlc’s code generation logic. Since we’re all set, execute sqlc generate command, this will output files like this:
```bash
    go mod tidy # first download pgx libraries
    
    tree
    # .
    # ├── api
    # │   ├── server.go
    # │   └── user_handler.go
    # ├── config
    # ├── db
    # │   ├── migration
    # │   │   ├── 000001_create_users_table.down.sql
    # │   │   └── 000001_create_users_table.up.sql   # sqlc used the file ...
    # │   ├── query
    # │   │   └── user.sql          # sqlc used the file to generate code
    # │   └── sqlc
    # │       ├── db.go             # sqlc-generated code
    # │       ├── models.go         # sqlc-generated code
    # │       ├── querier.go        # sqlc-generated code 
    # │       └── user.sql.go       # sqlc-generated code
    # ├── docker-compose.yaml
    # ├── env
    # ├── go.mod
    # ├── go.sum
    # └── sqlc.yaml                 # sqlc used the file to generate code
```
Have a look at the generated codes in db/sqlc , which should be as:
```go
    // 1. db.go
    
    package db
    
    import (
       "context"
    
       "github.com/jackc/pgconn"
       "github.com/jackc/pgx/v4"
    )
    
    type DBTX interface {
       Exec(context.Context, string, ...interface{}) (pgconn.CommandTag, error)
       Query(context.Context, string, ...interface{}) (pgx.Rows, error)
       QueryRow(context.Context, string, ...interface{}) pgx.Row
    }
    
    func New(db DBTX) *Queries {
       return &Queries{db: db}
    }
    
    
    
    // 2. models.go
    
    package db
    
    import (
       "time"
    )
    
    type User struct {
       ID        string    `json:"id"`
       Username  string    `json:"username"`
       Email     string    `json:"email"`
       Password  string    `json:"password"`
       Bio       *string   `json:"bio"`
       Image     *string   `json:"image"`
       CreatedAt time.Time `json:"created_at"`
       UpdatedAt time.Time `json:"updated_at"`
    }
    
    
    
    // 3. querier.go
    
    package db
    
    import (
       "context"
    )
    
    type Querier interface {
       CreateUser(ctx context.Context, arg CreateUserParams) (*User, error)
    }
    
    var _ Querier = (*Queries)(nil)
    
    
    
    // 4. user.sql.go
    
    package db
    
    import (
     "context"
    )
    
    const createUser = `-- name: CreateUser :one
    INSERT INTO users (
        id,
        username,
        email,
        password 
    ) VALUES (
        $1,
        $2,
        $3,
        $4
    ) 
    RETURNING id, username, email, password, bio, image, created_at, updated_at
    `
    
    type CreateUserParams struct {
       ID       string `json:"id"`
       Username string `json:"username"`
       Email    string `json:"email"`
       Password string `json:"password"`
    }
    
    func (q *Queries) CreateUser(ctx context.Context, arg CreateUserParams) (*User, error) {
       row := q.db.QueryRow(ctx, createUser,
          arg.ID,
          arg.Username,
          arg.Email,
          arg.Password,
       )
       var i User
       err := row.Scan(
          &i.ID,
          &i.Username,
          &i.Email,
          &i.Password,
          &i.Bio,
          &i.Image,
          &i.CreatedAt,
          &i.UpdatedAt,
       )
       return &i, err
    }
```
As you saw, we obtained our sought (q *Queries) CreateUser method which can be injected into (s *Server) RegisterUser handler:
```go
    
    type Server struct {
       config config.Config
       router *gin.Engine
       store  db.Querier
    }
    
    func (s *Server) RegisterUser(c *gin.Context) {
       var (
          req userRegisterReq
          p   db.CreateUserParams
       )
       if err := req.bind(c, &p); err != nil {
          c.JSON(http.StatusUnprocessableEntity, NewValidationError(err))
          return
       }
       user, err := s.store.CreateUser(c, p)
       if err != nil {
          if apiErr := convertToApiErr(err); apiErr != nil {
             c.JSON(http.StatusUnprocessableEntity, NewValidationError(apiErr))
             return
          }
          c.JSON(http.StatusInternalServerError, NewError(err))
          return
       }
       c.JSON(http.StatusCreated, newUserResponse(user))
    }
```
Using db.Querier directly in API handlers is a quick and straightforward way, but it comes with its drawbacks such as handling database errors at the API level.

### Conclusion

In a similar way, the login handler can also be implemented. To make it fast-forward I added more finished code for reference: [https://github.com/aliml92/go-web-demo](https://github.com/aliml92/go-web-demo).

In the next posts, we will explore task runner, database transactions and unit/integration tests, logging, and more.

If you want to see the finished conduit app, refer to this link: [https://github.com/aliml92/realworld-gin-sqlc](https://github.com/aliml92/realworld-gin-sqlc)