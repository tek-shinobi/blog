---
title: "MongoDB: Cheatsheet"
date: 2021-11-14T10:25:50+02:00
draft: false 
categories: ["mongodb"]
tags: ["mongodb", "docker"]
---

create and run mongodb docker container:
```shell
docker run --name mongodb -d -p 27017:27017 mongo:7 
```

login to mongodb container:
```shell
docker exec -it mongodb sh
```

run mongosh to enter mongo shell
```shell
> mongosh
```
it will automatically show a `test>` prompt which is a default test database

to clear screen:
```shell
cls
```

to ompletely exit out of mongosh and docker shell terminal:
```shell
exit
```

this command will show all dbs:
```shell
show dbs
```

to create a new database:
```shell
use appdb
```
here `appdb` is the name of the new database. It will create the database and then switch to that database. (it might be that at this stage, if you do `show dbs`, you cannot see `appdb`. This is because that db will get created when data is added to it)

to delete a database, first do `use <dbname>` to switch to that database and then:
```shell
db.dropDatabase()
```
note that above command will still keep you at `appdb>` prompt even if that database does not exist. It is because in mongodb, if a database does not exist and you try to add data or collections to it, it will automatically get created.

to view all collections in the database:
```shell
show collections
```

to access a current database (after running `use appdb` .. where `appdb` is the name of database I am using):
```shell
db
```
it will return the name of the current database

`db` actually has a bunch of methods on it.

to insert record into a collection named `users`:
```shell
appdb> db.users.insertOne({name: "Dingo"})
```

to see the above inserted record:
```shell
appdb> db.users.find()
```
the above method will return all records in the collection `users`

insert another record:
```shell
appdb> db.users.insertOne({name: "Sally", age: 19, address: {street: "987 North St"}, hobbies:["Running"]})
```
As seen above, with mongodb, we don't need to have all documents within a collection share the same fields. Also, we can nest objects within objects, like the address object and hobbies array in above record.

to insert multiple documents:
```shell
appdb> db.users.insertMany([{name: "Jill"}, {name: "Mike"}])
appdb> db.users.insertMany([{name: "Kyle", age: 26, hobbies:["Weight lifting", "Bowling"], address:{street: "123 Main St", city: "New York City"}}, {name: "Billy", age: 41, hobbies:["Swimming", "Bowling"], address:{street: "442 South St", city: "New York City"}}])
```

to limit the results to 2:
```shell
appdb> db.users.find().limit(2)
```

to sort alphabetically on `name`, `1` for ascending, `-1` for descending:
```shell
appdb> db.users.find().sort({name: 1}).limit(2)
```

to sort with multiple fields: we will first sort in order on `age` and then in reverse order on `name` :
```shell
appdb> db.users.find().sort({age: 1, name: 1}).limit(2)
```

if you want to skip some results:
```shell
appdb> db.users.find().sort({age: 1, name: 1}).skip(1).limit(2)
```
this will skip first result

to return only a select few fields:
```shell
appdb> db.users.find({name: "Kyle"}, {name: 1, age: 1, _id: 0})
```
here we pass a second object within `find()` that has the fields that I want set to 1. Exception is `_id` which is also returned by default. To not have it, pass 0 for it.

Also, to send all the fields except for `age` in the result:
```shell
appdb> db.users.find({name: "Kyle"}, {age: 0})
```

#### more complex find queries
```shell
appdb> db.users.find({name: {$ne: "Kyle"}})
```
here it will return all records where name is not Kyle.

```shell
appdb> db.users.find({age: {$gt: 13}})
```
here it will return all records where age is greater than 13.
- `$gte` is for greater than or equal to
- `$lte` is for less than or equal to
- `$lt` is for less than

a query where return all records where `name` is within a list of values:
```shell
appdb> db.users.find({name: {$in: ["Kyle", "Sally"]}})
```
this will return all records where name is either Kyle or Sally.
to achieve the opposite of this, use `$nin` which is `not in`

to return all records where `age` property is set:
```shell
appdb> db.users.find({age: {$exists: true}})
```

to return all records where `age` property is not set:
```shell
appdb> db.users.find({age: {$exists: false}})
```

note that `$exists` only checks if the key exists. For example, in above examples, it will check if key `age` exists.

If however the key exists but its value is set to null, `$exists` will still evaluate to true. So it only checks is key exists and not if the value also exists. 

to return all records within a range:
```shell
appdb> db.users.find({age: {$gte: 20, $lte: 40}})
```

to do an `AND` style query where we return all records within a range AND with name Kyle:
```shell
appdb> db.users.find({age: {$gte: 20, $lte: 40}, name: "Kyle"})
```
there is another way to do the same `AND` query:
```shell
appdb> db.users.find({$and: [{age: {$gte: 20, $lte: 40}}, {name: "Kyle"}]})
```

generally you don't need to do AND style queries with `$and` because all other ways also support and. But where you do need this dollar style syntax is for OR queries.
```shell
appdb> db.users.find({$or: [{age: {$gte: 20, $lte: 40}}, {name: "Kyle"}]})
```

let's also do a `$not` query:
```shell
appdb> db.users.find({age: {$not: {$lte: 20}}})
```
this will return all users whose age is not less than or equal to 20, as well as those users who do not have age key defined.

if we tried to do this:
```shell
appdb> db.users.find({age: {$gt: 20}})
```
we would think that this is equivalent of the one with `$not` but this one will only return records where key age is defined and whose age is greter than 20. This is one of the places where `$not` based queries are helpful.

Let's insert this data now:
```shell
appdb> db.users.insertMany([{name: "Tom", balance: 100, debt: 200},{name: "Kristin", balance: 20, debt: 0}])
```
Let's now write a query for users whose debt is greater than their balance (essentially users who are in debt). So here will need to compare two different properties on an object. We use `$expr` for this.
```shell
appdb> db.users.find({$expr: {$gt: ["$debt", "$balance"]}})
```
notice the `$` before `debt` and `balance`. This is special way to indicate that we want to access these columns. If there was `$` missing before theses two, we would not get proper results.

if we wanted to include nested fields in our query, like stree within address:
```shell
appdb> db.users.find({"address.street": "123 Main St"})
```
so here `address.street` is just like any other column name.

we can also do a `findOne` to return the very first result:
```shell
appdb> db.users.findOne({age: {$lte: 40}})
```

We can also return a count:
```shell
appdb> db.users.countDocuments({age: {$lte: 40}})
```

#### Update data
the very first object passed to `updateOne` is the object for filtering, same as for find queries. The second parameter passed is the atomic operator that contains the updated field and values. This has a special syntax as this operation operation needs to be atomic
```shell
appdb> db.users.updateOne({age: 26}, {$set:{age: 27}})
```
here `$set` is the atomic operator and `{age: 27}` is the update. So this query will replace all age=26 entries to age=27. And each of these update operations will be atomic.

when doing based on `_id`:
```shell
appdb> db.users.updateOne({_id: ObjectId("1232132323123ec")}, {$set:{age: 27}})
```

to increment a value by 3:
```shell
appdb> db.users.updateOne({_id: ObjectId("1232132323123ec")}, {$inc:{age: 3}})
```

to rename a field:
```shell
appdb> db.users.updateOne({_id: ObjectId("1232132323123ec")}, {$rename:{name: "firstName"}})
```
this will rename "name" column to "firstName"

to completely remove a field from an object, use `unset`
```shell
appdb> db.users.updateOne({_id: ObjectId("1232132323123ec")}, {$unset:{age: ""}})
```
here, the `age` property will be removed from the document at id 1232132323123ec. Moreover, `{age: ""}` here the empty value part is just a placeholder and does not matter as this property will be removed entirely.


to add values to an array:
```shell
appdb> db.users.updateOne({_id: ObjectId("1232132323123ec")}, {$push: {hobbies: "Swimming"}})
```
this query will append "Swimming" to the contents of array `hobbies`

to do the opposite, remove an entry from an array:
```shell
appdb> db.users.updateOne({_id: ObjectId("1232132323123ec")}, {$pull: {hobbies: "Bowling"}})
```
will remove Bowling from hobbies.

to update many records at once, for example to remove address property from all documents that have it:
```shell
appdb> db.users.updateMany({address: {$exists: true}}, {$unset: {address: ""}})
```

we also have replace, that can replace an object entirely:
```shell
appdb> db.users.replaceOne({age: 30}, {name: "John"})
```
here, the first arguement is the filter and the second arguement is the replacement object.

#### delete
```shell
appdb> db.users.deleteOne({name: "John"})
```
just give the filter. Like here, it will delete the user John that we inserted in previous step.

to do multiple deletes:
```shell
appdb> db.users.deleteMany({age: {$exists: false}})
```
this will delete all documents that dont have age property.



