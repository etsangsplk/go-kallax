Kallax - PostgreSQL ORM for Go
=============================

[![GoDoc](https://godoc.org/github.com/src-d/go-kallax?status.svg)](https://godoc.org/github.com/src-d/go-kallax) [![Build Status](https://travis-ci.org/src-d/go-kallax.svg?branch=master)](https://travis-ci.org/src-d/go-kallax) [![codecov](https://codecov.io/gh/src-d/go-kallax/branch/master/graph/badge.svg)](https://codecov.io/gh/src-d/go-kallax) [![Go Report Card](https://goreportcard.com/badge/github.com/src-d/go-kallax)](https://goreportcard.com/report/github.com/src-d/go-kallax) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)


Kallax is a PostgreSQL typesafe ORM for the Go language.

It aims to provide a way of programmatically write queries and interact with a PostgreSQL database without having to write a single line of SQL, use strings to refer to columns and use values of any type in queries.

For that reason, the first priority of kallax is to provide type safety to the data access layer.
Another of the goals of kallax is make sure all models are, first and foremost, Go structs without having to use database-specific types such as, for example, `sql.NullInt64`.
Support for arrays of all basic Go types and all JSON and arrays operators is provided as well.

## Contents

* [Installation](#installation)
* [Usage](#usage)
* [Define models](#define-models)
  * [Struct tags](#struct-tags)
  * [Primary keys](#primary-keys)
  * [Model constructors](#model-constructors)
  * [Model events](#model-events)
* [Model schema](#model-schema)
  * [Automatic schema generation and migrations](#automatic-schema-generation-and-migration)
  * [Use schema](#use-schema)
* [Manipulate models](#manipulate-models)
  * [Insert models](#insert-models)
  * [Update models](#update-models)
  * [Save models](#save-models)
  * [Delete models](#delete-models)
* [Query models](#query-models)
  * [Simple queries](#simple-queries)
  * [Query with relationships](#query-with-relationships)
  * [Querying JSON](#querying-json)
* [Transactions](#transactions)
* [Contributing](#contributing)

## Installation

The recommended way to install `kallax` is:

```
go get -u github.com/src-d/kallax/...
```

> *kallax* includes a binary tool used by [go generate](http://blog.golang.org/generate),
please be sure that `$GOPATH/bin` is on your `$PATH`

## Usage
 
Imagine you have the following file in the package where your models are.
```go
package models

type User struct {
        kallax.Model         `table:"users"`
        ID       kallax.ULID `pk:""`
        Username string
        Email    string
        Password string
}
```

Then put the following on any file of that package:

```go
//go:generate kallax gen
```

Now all you have to do is run `go generate ./...` and a `kallax.go` file will be generated with all the generated code for your model.

If you don't want to use `go generate`, even though is the preferred use, you can just go to your package and run `kallax gen` yourself.

### Excluding files from generation

Sometimes you might want to use the generated code in the same package it is defined and cause problems during the generation when you regenerate your models. You can exclude files in the package by changing the `go:generate` comment to the following:

```go
//go:generate kallax gen -e file1.go -e file2.go
```

## Define models

A model is just a Go struct that embeds the `kallax.Model` type. All the fields of this struct will be columns in the database table.

A model also needs to have one (and just one) primary key. That is whatever field of the struct with the struct tag `pk`, which can be `pk:""` for a non auto-incrementable primary key or `pk:"autoincr"` for one that is auto-incrementable.
More about primary keys is discussed at the [primary keys](#primary-keys) section.

First, let's review the rules and conventions for model fields:
* All the fields with basic types or types that implement [sql.Scanner](https://golang.org/pkg/database/sql/#Scanner) and [driver.Valuer](https://golang.org/pkg/database/sql/driver/#Valuer) will be considered a column in the table of their matching type.
* Arrays or slices of types mentioned above will be treated as PostgreSQL arrays of their matching type.
* Fields that are structs (or pointers to structs) or interfaces not implementing [sql.Scanner](https://golang.org/pkg/database/sql/#Scanner) and [driver.Valuer](https://golang.org/pkg/database/sql/driver/#Valuer) will be considered as JSON. Same with arrays or slices of types that follow these rules.
* Fields that are structs (or pointers to structs) with the struct tag `kallax:",inline"` or are embedded will be considered inline, and their fields would be considered as if they were at the root of the model.
* All pointer fields are nullable by default. That means you do not need to use `sql.NullInt64`, `sql.NullBool` and the likes because kallax automatically takes care of that for you. **WARNING:** all JSON and `sql.Scanner` implementors will be initialized with `new(T)` in case they are `nil` before they are scanned.
* By default, the name of a column will be the name of the struct field converted to lower snake case (e.g. `UserName` => `user_name`, `UserID` => `user_id`). You can override it with the struct tag `kallax:"my_custom_name"`.
* Slices or arrays of structs (or pointers to structs) that are models themselves will be considered a 1:N relationship.
* A struct or pointer to struct field that is a model itself will be considered a 1:1 relationship.
* For relationships, the foreign key is assumed to be the name of the model converted to lower snake case plus `_id` (e.g. `User` => `user_id`). You can override this with the struct tag `fk:"my_custom_fk"`.
* For inverse relationship, you need to use the struct tag `fk:",inverse"`. You can combine the `inverse` with overriding the foreign key with `fk:"my_custom_fk,inverse"`. In the case of inverses, the foreign key name does not specify the name of the column in the relationship table, but the name of the column in the own table. The name of the column in the other table is always the primary key of the other model and cannot be changed for the time being.
* Foreign keys *do not have to be in the model*, they are automagically managed underneath by kallax.

Kallax also provides a `kallax.Timestamps` struct that contains `CreatedAt` and `UpdatedAt` that will be managed automatically.

Let's see an example of models with all these cases:

```go
type User struct {
        kallax.Model       `table:"users"`
        kallax.Timestamps
        kallax.ID int64    `pk:"autoincr"`
        Username  string
        Password  string
        Emails    []string
        // This is for demo purposes, please don't do this
        // 1:N relationships load all N rows by default, so
        // only do it when N is small.
        // If N is big, you should probably be querying the posts
        // table instead.
        Posts []*Post `fk:"poster_id"`
}

type Post struct {
        kallax.Model      `table:"posts"`
        kallax.Timestamps
        ID       int64    `pk:"autoincr"`
        Content  string   `kallax:"post_content"`
        Poster   *User    `fk:"poster_id,inverse"`
        Metadata Metadata `kallax:",inline"`
}

type Metadata struct {
        MetadataType MetadataType
        Metadata map[string]interface{} // this will be json
}
```

### Struct tags

| Tag | Description | Can be used in |
| --- | --- | --- | --- |
| `table"table_name"` | Specifies the name of the table for a model. If not provided, the name of the table will be the name of the struct in lower snake case (e.g. `UserPreference` => `user_preference`) | embedded `kallax.Model` |
| `pk:""` | Specifies the field is a primary key | any field with a valid identifier type |
| `pk:"autoincr"` | Specifies the field is an auto-incrementable primary key | any field with a valid identifier type |
| `kallax:"column_name"` | Specifies the name of the column | Any model field that is not a relationship |
| `kallax:"-"` | Ignores the field and does not store it | Any model field |
| `kallax:",inline"` | Adds the fields of the struct field to the model. Column name can also be given before the comma, but it is ignored, since the field is not a column anymore | Any struct field |
| `fk:"foreign_key_name"` | Name of the foreign key column | Any relationship field |
| `fk:",inverse"` | Specifies the relationship is an inverse relationship. Foreign key name can also be given before the comma | Any relationship field |

### Primary keys

Primary key types need to satisfy the [Identifier](https://godoc.org/github.com/src-d/go-kallax/#Identifier) interface. Even though they have to do that, the generator is smart enough to know when to wrap some types to make it easier on the user.

The following types can be used as primary key:

* `int64`
* [`uuid.UUID`](https://godoc.org/github.com/satori/go.uuid#UUID)
* [`kallax.ULID`](https://godoc.org/github.com/src-d/go-kallax/#ULID): this is a type kallax provides that implements a lexically sortable UUID. You can store it as `uuid` like any other UUID, but internally it's an ULID and you will be able to sort lexically by it.

If you need another type as primary key, feel free to open a pull request implementing that.

**Known limitations**

* Only one primary key can be specified and it can't be a composite key.

### Model constructors

Kallax generates a constructor for your type named `New{TypeName}`. But you can customize it by implementing a private constructor named `new{TypeName}`. The constructor generated by kallax will use the same signature your private constructor has. You can use this to provide default values or construct the model with some values.

If you implement this constructor:

```go
func newUser(username, password string, emails ...string) (*User, error) {
        if username != "" || len(emails) == 0 || password != "" {
                return errors.New("all fields are required")
        }

        return &User{Username: username, Password: password, Emails: emails}
}
```

Kallax will generate one with the following signature:

```go
func NewUser(username string, password string, emails ...string) (*User, error)
```

**IMPORTANT:** if your primary key is not auto-incrementable, you should set an ID for every model you create in your constructor. Or, at least, set it before saving it. Inserting, updating, deleting or reloading an object with no primary key set will return an error.

If you don't implement your own constructor it's ok, kallax will generate one for you just instantiating your object like this:

```go
func NewT() *T {
        return new(T)
}
```

### Model events

Events can be defined for models and they will be invoked at certain times of the model lifecycle.

* `BeforeInsert`: will be called before inserting the model.
* `BeforeUpdate`: will be called before updating the model.
* `BeforeSave`: will be called before updating or inserting the model. It's always called before `BeforeInsert` and `BeforeUpdate`.
* `BeforeDelete`: will be called before deleting the model.
* `AfterInsert`: will be called after inserting the model. The presence of this event will cause the insertion of the model to run in a transaction. If the event returns an error, it will be rolled back.
* `AfterUpdate`: will be called after updating the model. The presence of this event will cause the update of the model to run in a transaction. If the event returns an error, it will be rolled back.
* `AfterSave`: will be called after updating or inserting the model. It's always called after `AfterInsert` and `AfterUpdate`. The presence of this event will cause the operation with the model to run in a transaction. If the event returns an error, it will be rolled back.
* `AfterDelete`: will be called after deleting the model. The presence of this event will cause the deletion to run in a transaction. If the event returns an error, it will be rolled back.

To implement these events, just implement the following interfaces. You can implement as many as you want:

* [BeforeInserter](https://godoc.org/github.com/src-d/go-kallax#BeforeInserter)
* [BeforeUpdater](https://godoc.org/github.com/src-d/go-kallax#BeforeUpdater)
* [BeforeSaver](https://godoc.org/github.com/src-d/go-kallax#BeforeSaver)
* [BeforeDeleter](https://godoc.org/github.com/src-d/go-kallax#BeforeDeleter)
* [AfterInserter](https://godoc.org/github.com/src-d/go-kallax#AfterInserter)
* [AfterUpdater](https://godoc.org/github.com/src-d/go-kallax#AfterUpdater)
* [AfterSaver](https://godoc.org/github.com/src-d/go-kallax#AfterSaver)
* [AfterDeleter](https://godoc.org/github.com/src-d/go-kallax#AfterDeleter)

Example:

```go
func (u *User) BeforeSave() error {
        if u.Password == "" {
                return errors.New("cannot save user without password")
        }

        if !isCrypted(u.Password) {
                u.Password = crypt(u.Password)
        }
        return nil
}
```

## Kallax generated code

Kallax generates a bunch of code for every single model you have and saves it to a file named `kallax.go` in the same package.

For every model you have, kallax will generate the following for you:

* Internal methods for your model to make it work with kallax and satisfy the [Record](https://godoc.org/github.com/src-d/go-kallax#Record) interface.
* A store named `{TypeName}Store`: the store is the way to access the data. A store of a given type is the way to access and manipulate data of that type. You can get an instance of the type store with `New{TypeName}Store(*sql.DB)`.
* A query named `{TypeName}Query`: the query is the way you will be able to build programmatically the queries to perform on the store. A store only will accept queries of its own type. You can create a new query with `New{TypeName}Query()`.
The query will contain methods for adding criteria to your query for every field of your struct, called `FindBy`s. The query object is not immutable, that is, every condition added to it, changes the query. If you want to reuse part of a query, you can call the `Copy()` method of a query, which will return a query identical to the one used to call the method.
* A resultset named `{TypeName}ResultSet`: a resultset is the way to iterate over and obtain all elements in a resultset returned by the store. A store of a given type will always return a result set of the matching type, which will only return records of that type.
* Schema of all the models containing all the fields. That way, you can access the name of a specific field without having to use a string, that is, a typesafe way.

## Model schema

### Automatic schema generation and migrations

Automatic `CREATE TABLE` for models and migrations is not yet supported, even though it will probably come in future releases.

### Use schema

A global variable `Schema` will be created in your `kallax.go`, that contains a field with the name of every of your models. Those are the schemas of your models. Each model schema contains all the fields of that model.

So, to access the username field of the user model, it can be accessed as:

```go
Schema.User.Username
```

## Manipulate models

For all of the following sections, we will assume we have a store `store` for our model's type.

### Insert models

To insert a model we just need to use the `Insert` method of the store and pass it a model. If the primary key is not auto-incrementable and the object does not have one set, the insertion will fail.

```go
user := NewUser("fancy_username", "super_secret_password", "foo@email.me")
err := store.Insert(user)
if err != nil {
        // handle error
}
```

If our model has relationships, they will be saved (**note:** saved as in insert or update) as well. The relationships of the relationships will not, though. Relationships are only saved with one level of depth.

```go
user := NewUser("foo")
user.Posts = append(user.Posts, NewPost(user, "new post"))

err := store.Insert(user)
if err != nil {
        // handle error
}
```

If there are any relationships in the model, both the model and the relationships will be saved in a transaction and only succeed if all of them are saved correctly.

### Update models

To insert a model we just need to use the `Update` method of the store and pass it a model. It will return an error if the model was not already persisted or has not an ID.

```go
user := FindLast()
rowsUpdated, err := store.Update(user)
if err != nil {
        // handle error
}
```

By default, when a model is updated, all its fields are updated. You can also specify which fields to update passing them to update.

```go
rowsUpdated, err := store.Update(user, Schema.User.Username, Schema.User.Password)
if err != nil {
        // handle error
}
```

If our model has relationships, they will be saved (**note:** saved as in insert or update) as well. The relationships of the relationships will not, though. Relationships are only saved with one level of depth.

```go
user := FindLastPoster()
rowsUpdated, err := store.Update(user)
if err != nil {
        // handle error
}
```

If there are any relationships in the model, both the model and the relationships will be saved in a transaction and only succeed if all of them are saved correctly.

### Save models

To save a model we just need to use the `Save` method of the store and pass it a model. `Save` is just a shorthand that will call `Insert` if the model is not yet persisted and `Update` if it is.

```go
updated, err := store.Save(user)
if err != nil {
        // handle error
}

if updated {
        // it was updated, not inserted
}
```

If our model has relationships, they will be saved as well. The relationships of the relationships will not, though. Relationships are only saved with one level of depth.

```go
user := NewUser("foo")
user.Posts = append(user.Posts, NewPost(user, "new post"))

updated, err := store.Save(user)
if err != nil {
        // handle error
}
```

If there are any relationships in the model, both the model and the relationships will be saved in a transaction and only succeed if all of them are saved correctly.

### Delete models

To delete a model we just have to use the `Delete` method of the store. It will return an error if the model was not already persisted.

```go
err := store.Delete(user)
if err != nil {
        // handle error
}
```

Relationships of the model are **not** automatically removed using `Delete`.

For that, specific methods are generated in the store of the model.

For one to many relationships:

```go
// remove specific posts
err := store.RemovePosts(user, post1, post2, post3)
if err != nil {
        // handle error
}

// remove all posts
err := store.RemovePosts(user)
```

For one to one relationships:

```go
// remove the thing
err := store.RemoveThing(user)
```

## Query models

### Simple queries

To perform a query you have to do the following things: 
* Create a query
* Pass the query to `Find`, `FindOne`, `MustFind` or `MustFindOne` of the store
* Gather the results from the result set, if the used method was `Find` or `MustFind`

```go
// Create the query
q := NewUserQuery().
        Where(kallax.Like(Schema.User.Username, "joe%")).
        Order(kallax.Asc(Schema.User.Username)).
        Limit(20).
        Offset(2)

rs, err := store.Find(q)
if err != nil {
        // handle error
}

for rs.Next() {
        user, err := rs.Get()
        if err != nil {
                // handle error
        }
}
```

Next will automatically close the result set when it hits the end. If you have to prematurely exit the iteration you can close it manually with `rs.Close()`.

You can query just a single row with `FindOne`.

```go
q := NewUserQuery().
        Where(kallax.Eq(Schema.User.Username, "Joe"))

user, err := store.FindOne(q)
```

By default, all columns in a row are retrieved. To not retrieve all of them, you can specify the columns to include/exclude. Take into account that partial records retrieved from the database will not be writable. To make them writable you will need to [`Reload`](#reloading-a-model) the object.

```go
// Select only Username and password
NewUserQuery().Select(Schema.User.Username, Schema.User.Password)

// Select all but password
NewUserQuery().SelectNot(Schema.User.Password)
```

### Count results

Instead of passing the query to `Find` or `FindOne`, you can pass it to `Count` to get the number of rows in the resultset.

```go
n, err := store.Count(q)
```

### Query with relationships

By default, no relationships are retrieved unless the query specifies so.

For each of your relationships, a method in your query is created to be able to include these relationships in your query.

One to one relationships:

```go
// Select all posts including the user that posted them
q := NewPostQuery().WithPoster()
rs, err := store.Find(q)
```

One to one relationships are always included in the same query. So, if you have 4 one to one relationships and you want them all, only 1 query will be done, but everything will be retrieved.

One to many relationships:

```go
// Select all users including their posts
// NOTE: this is a really bad idea, because all posts will be loaded
// if the N side of your 1:N relationship is big, consider querying the N store
// instead of doing this
// A condition can be passed to the `With{Name}` method to filter the results.
q := NewUserQuery().WithPosts(nil)
rs, err := store.Find(q)
```

To avoid the N+1 problem with 1:N relationships, kallax performs batching in this case. 
So, a batch of users are retrieved from the database in a single query, then all the posts for those users and finally, they are merged. 
This process is repeated until there are no more rows in the result.
Because of this, retrieving 1:N relationships is really fast.

The default batch size is 50, you can change this using the `BatchSize` method all queries have.

**NOTE:** if a filter is passed to a `With{Name}` method we can no longer guarantee that all related objects are there and, therefore, the retrieved records will **not** be writable.

### Reloading a model

If, for example, you have a model that is not writable because you only selected one field you can always reload it and have the full object. When the object is reloaded, all the changes made to the object that have not been saved will be discarded and overwritten with the values in the database.

```go
err := store.Reload(user)
```

Reload will not road any relationships, just the model itself. After a `Reload` the model will **always** be writable.

### Querying JSON

You can query arbitrary JSON using the JSON operators defined in the [kallax](https://godoc.org/github.com/src-d/go-kallax) package. The schema of the JSON (if it's a struct, obviously for maps it is not) is also generated.

```go
q := NewPostQuery().Where(kallax.JSONContainsAnyKey(
        Schema.Post.Metadata,
        "foo", "bar",
))
```

## Transactions

To execute things in a transaction the `Transaction` method of the model store can be used. All the operations done using the store provided to the callback will be run in a transaction.
If the callback returns an error, the transaction will be rolled back.

```go
store.Transaction(func(s *UserStore) error {
        if err := s.Insert(user1); err != nil {
                return err
        }

        return s.Insert(user2)
})
```

The fact that a transaction receives a store with the type of the model can be a problem if you want to store several models of different types. You can, indeed, create new stores of the other types, but do so with care. Do not use the internal `*kallax.Store`, as it does not perform any type checks or some of the operations the concrete type stores do.

```go
store.Transaction(func(s *UserStore) error {
        postStore := &PostStore{s.Store}

        for _, p := range posts {
                if err := postStore.Insert(p); err != nil {
                        return err
                }
        }

        return s.Insert(user)
})
```

`Transaction` can be used inside a transaction, but it does not open a new one, reuses the existing one.

## Caveats

* It is not possible to use slices or arrays of types that are not one of these types:
  * Basic types (e.g. `[]string`, `[]int64`) (except for `rune`, `complex64` and `complex128`)
  * Types that implement `sql.Scanner` and `driver.Valuer`
  The reason why this is not possible is because kallax implements support for arrays of all basic Go types by hand and also for types implementing `sql.Scanner` and `driver.Valuer` (using reflection in this case), but without having a common interface to operate on them, arbitrary types can not be supported.
  For example, consider the following type `type Foo string`, using `[]Foo` would not be supported. Know that this will fail during the scanning of rows and not in code-generation time for now. In the future, might be moved to a warning or an error during code generation.
  Aliases of slice types are supported, though. If we have `type Strings []string`, using `Strings` would be supported, as a cast like this `([]string)(&slice)` it's supported and `[]string` is supported.
* `time.Time` and `url.URL` need to be used as is. That is, you can not use a type `Foo` being `type Foo time.Time`. `time.Time` and `url.URL` are types that are treated in a special way, if you do that, it would be the same as saying `type Foo struct { ... }` and kallax would no longer be able to identify the correct type.

## Contributing 

### Reporting bugs

Kallax is a code generation tool, so it obviously has not been tested with all possible types and cases. If you find a case where the code generation is broken, please report an issue providing a minimal snippet for us to be able to reproduce the issue and fix it.

### Suggesting features

Kallax is a very opinionated ORM that works for us, so changes that make things not work for us or add complexity via configuration will not be considered for adding.
If we decide not to implement the feature you're suggesting, just keep in mind that it might not be because it is not a good idea, but because it does not work for us or is not aligned with the direction we want kallax to be moving forward.

### Running tests

For obvious reasons, an instance of PostgreSQL is required to run the tests of this package.

By default, it assumes that an instance exists at `0.0.0.0:5432` with an user, password and database name all equal to `testing`.

If that is not the case you can set the following environment variables:

- `DBNAME`: name of the database
- `DBUSER`: database user
- `DBPASS`: database user password

License
-------

MIT, see [LICENSE](LICENSE)
