# Rails: Create Multiple Records For a Single Model in a Single Request
Creating multiple entries in a database using Rails is easier than it seems. We can even ensure that if one fails, our database gets rolled back to its previous state! Ooh la la. And one of the most beautiful parts is that we don't have to iterate over anything!! Note that this article talks about creating many records in a single table (model), not many tables (models). Also, this isn't a batch SQL insert: I'll include a note about that at the end.

## What Do We Want to Do?
Let's say we have a table storing Band data, and we want the ability to add several new Bands to this table from the frontend (JavaScript) using a fetch request. Our initial idea might be to loop through all of the new Band data on the frontend and send one fetch request per Band. However, this results in sending more fetch requests than necessary. Instead, we can actually add all of the new Bands using a single fetch request, which reduces the number of requests our server must handle.

The fetch request we're aiming for might look something like this:
```
const options = {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ band: bands }) // bands is an array of hashes
}
```

## Hello My Old Friend, ActiveRecord create(!)
To do this easily, we first need to take a look at our old class method friends `create` and `create!`. Commonly, we use these methods to create a single entry in a database by passing a Hash as an argument:
```
Band.create(name: "Nite Fields", year: 2012)
Band.create!(name: "Nite Fields", year: 2012)
```

However, we can also pass an Array of Hashes to create several records at once:
```
Band.create([{ name: "Nite Fields", year: 2012 }, { name: "Refused", year: 1991 }])
Band.create!([{ name: "Nite Fields", year: 2012 }, { name: "Refused", year: 1991 }])
```

But the question is: what happens if some of the records in the Array are valid and others are not? Will the valid records be created anyway? Luckily, this is easy to test. We can attempt to create several valid records, and then attempt to create a mix of invalid and valid records all at once:
```
# VALID RECORDS ONLY
Band.create([{ name: "Nite Fields", year: 2012 }, { name: "Refused", year: 1991 }])

# ONE INVALID RECORD
Band.create([{ name: "Radiohead", year: 1985 }, { name: "The Blah Blahs" }, { name: "Testing", year: 2222 }])
```
When each Hash contains valid data (passes validations and any database-level checks), the records are created as expected: they are inserted one-at-a-time into the database. When creating a mix of valid and invalid records, we actually get mixed results depending on where the invalid data is located in the Array.

**Database-level validations using column modifiers only**
When we use column modifiers in our migrations, such as `null: false`, but we don't add validations to our models, `create` will abort as soon as an invalid Hash of attributes is encountered in the Array. All records preceding the invalid one will be inserted into the database. For example, if the first Hash in the Array contains invalid data, no records will be inserted. If the middle Hash contains invalid data, every record preceding it will be inserted:
```
# NO RECORDS WILL BE INSERTED, FIRST HASH IS INVALID
Band.create([{ name: "The Blah Blahs" }, { name: "Radiohead", year: 1985 }])

# ONLY RADIOHEAD WILL BE INSERTED
Band.create([{ name: "Radiohead", year: 1985 }, { name: "The Blah Blahs" }, { name: "Testing", year: 2222 }])
```

But what if we use `create!` instead. Is the behavior any different? Let's find out:
```
# NO RECORDS WILL BE INSERTED, FIRST HASH IS INVALID
Band.create!([{ name: "The Blah Blahs" }, { name: "Radiohead", year: 1985 }])

# ONLY RADIOHEAD WILL BE INSERTED
Band.create!([{ name: "Radiohead", year: 1985 }, { name: "The Blah Blahs" }, { name: "Testing", year: 2222  }])
```
It's the same behavior! Well, that makes things a little easier on our brains.

**Model validations**
Adding model validations changes the behavior of `create`. I added validation checks for both the name and year fields to check for their presence. First let's put the invalid record at the start of the Array:
```
# RADIOHEAD IS INSERTED
Band.create([{ name: "The Blah Blahs" }, { name: "Radiohead", year: 1985 }])

# RADIOHEAD AND TESTING ARE INSERTED
Band.create([{ name: "Radiohead", year: 1985 }, { name: "The Blah Blahs" }, { name: "Testing", year: 2222 }])
```
Unlike before, this actually resulted in all valid records being inserted into the database.

But what about `create!`? Is there a difference?
```
# NOTHING IS INSERTED, ABORTED ON FIRST INVALID RECORD
Band.create!([{ name: "The Blah Blahs" }, { name: "Radiohead", year: 1985 }])

# RADIOHEAD IS INSERTED, ABORTED ON FIRST INVALID RECORD
Band.create!([{ name: "Radiohead", year: 1985 }, { name: "The Blah Blahs" }, { name: "Testing", year: 2222 }])
```
This is actually similar to before. As soon as an invalid record was encountered, the process was aborted. The Blah Blahs, sadly, are still floating in space waiting for their time to shine.

**Return values and exceptions**
When creating only valid records `create` will return an Array of valid objects representing the entries that were just created, including their IDs. When creating a combination of valid and invalid records `create` returns all of those records, but the invalid records will have an ID of nil. If all of the records are invalid, `create` will still return an Array of those objects with IDs set to nil.

`create!` operates differently. If all of the records are valid, it will return them in an Array with their IDs set. If one record is invalid, regardless of where it's located in the Array, it will raise an exception. If we try to set a variable to the return value of `create!` when it raises an exception, the value stored in that variable will either be nil or its previous value:
```
a = Band.create!([{ name: 'Bandy McBandFace', year: 3000 }, { name: 'not happening' }])
# a => nil

a = 'dance magic dance'
a = Band.create!([{ name: 'Bandy McBandFace', year: 3000 }, { name: 'not happening' }])
# a => 'dance magic dance'
```
