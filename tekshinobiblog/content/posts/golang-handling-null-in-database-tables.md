---
title: "Golang: Handling Null in Database Tables"
date: 2023-07-11T01:52:51+03:00
draft: false 
categories: ["golang", "go"]
tags: ["golang", "go", "null"]
---

Consider this table schema (Postgres):
```sql
CREATE TABLE IF NOT EXISTS mytable (
id BIGSERIAL PRIMARY KEY,
title TEXT NULL,
version INTEGER NULL 
);
```

if we write its Go model, `title` is `string` and `version` is `int`. But here we have a problem. Zero value of string in Go is `""` not `nil`. Same for `int` whose zero value is `0` and not `nil`.

If we tried to write the above model in Go like this:
```go
type MyTable struct {
	ID      int64
	Title  string 
	Version int
}
```

there is a chance that when MyTable row was inserted in database, Title and Version are both `nil` since the database schema allows null values for them. Then when we tried to read that value into MyTable like so:
```go
	query := `
	select id, title, version 
	from mytable 
	where id = $1
	`
	var mt MyTable

	err := n.DB.QueryRow(query, id).Scan(&mt.ID, &mt.Title, &mt.Version)
```
we will get a panic in the last line, scanning nil into Title or Version. Since those types don't allow nil.

So, clearly the `mytable` cannot be expressed in Go using its native data types.

There are two ways of handling this. Both have their merits and demerits. 

## Approach 1: using sql.Null*
Let's write the corresponding Go model using this approach:
```go
type MyTable struct {
	ID      int64
	Title   sql.NullString
	Version sql.NullInt32
}
```

Here, `database/sql` provides `sql.Null*` types that already have a scanner interface implemented and in that implementation, they handle the mapping of sql null to Go native zero values. Using this approach, there will be no panic. 

But we have a problem here. Now, if we wrote our Get method like so:
```go
func (n NullValModels) Get(id int64) (*MyTable, error) {
	query := `
	select id, title, version 
	from mytable 
	where id = $1
	`
	var mt MyTable

	err := n.DB.QueryRow(query, id).Scan(&mt.ID, &mt.Title, &mt.Version)
	return mt, err
}
```
the native sql.Null* implementation is exposed to the rest of the application, when it should be isolated to the repository layer.

If we tried to print out the returned row, that have nulls in Title and Version, it will look like this:
```terminal
&{ID:1 Title:{String: Valid:false} Version:{Int32:0 Valid:false}}
```

This is not ideal if we are following clean architecture principles, and we should be. The service layer method that called this method should not be exposed to this internal `database/sql` implementation of handling nulls. 

Thus we would abstract this internal detail like so:
```go
type NullValModels struct {
	DB *sql.DB
}

type MyTable struct {
	ID      int64
	Title   sql.NullString
	Version sql.NullInt32
}

type MyTableService struct {
	ID      int64
	Title   string
	Version int
}

func (n NullValModels) Get(id int64) (*MyTableService, error) {
	query := `
	select id, title, version 
	from mytable 
	where id = $1
	`
	var mt MyTable

	err := n.DB.QueryRow(query, id).Scan(&mt.ID, &mt.Title, &mt.Version)
	if err != nil {
		return nil, err
	}
	mts := &MyTableService{
		ID:      mt.ID,
		Title:   mt.Title.String,
		Version: int(mt.Version.Int32),
	}

	return mts, err
}
```

This is a valid approach, the downside is the duplicated MyTable and MyTableService models. The upside is that in service layer we can use this data as is. What "as is" means will become clear when contrasted with the next approach. 

## Approach 2: using pointers in sql model
```go
type MyTable struct {
	ID      int64
	Title   *string 
	Version *int
}

func (n NullValModels) Get(id int64) (*MyTable, error) {
	query := `
	select id, title, version 
	from mytable 
	where id = $1
	`
	var mt MyTable

	err := n.DB.QueryRow(query, id).Scan(&mt.ID, &mt.Title, &mt.Version)
	return &mt, err
}
```
This approach will output a result like so:
```terminal
&{ID:1 Title:<nil> Version:<nil>}
```
Notice the nils in the result. These will now be leaked into the service layer and will have to be handled there. So the upside is that there is no duplication but downside is that nils need to handled in service layer.

## Conclusion

Both are valid and idiomatic ways of handling this situation. [Here](https://groups.google.com/g/golang-nuts/c/vOTFu2SMNeA/m/GB5v3JPSsicJ) is what Russ Cox, one of the co-creators of Go, opines about this:
> There's no effective difference. We thought people might want to use NullString because it is so common and perhaps expresses the intent more clearly than *string. But either will work.
>
> Russ