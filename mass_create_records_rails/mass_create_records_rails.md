# Rails: Create Multiple Records For a Single Model in a Single Request
Creating multiple entries in a database using Rails is easier than it seems. We can even ensure that if one fails, our database gets rolled back to its previous state! Ooh la la. And one of the most beautiful parts is that we don't have to iterate over anything!! Note that this article talks about creating many records in a single table (model), not many tables (models). Also, this isn't a batch SQL insert: I'll include a note about that at the end.

## What Do We Want to Do?
Let's say we have a table storing Band data, and we want the ability to add several new Bands to this table from the frontend (JavaScript) using a fetch request. Our initial idea might be to loop through all of the new Band data on the frontend and send one fetch request per Band. However, this results in sending more fetch requests than necessary. Instead, we can actually add all of the new Bands using a single fetch request, which reduces the load on our server.

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
# Band.create!(name: "Nite Fields", year: 2012)
```

However, we can also pass an Array of Hashes to create several records at once:
```
Band.create([{ name: "Nite Fields", year: 2012 }, { name: "Refused", year: 1991 }])
# Band.create!([{ name: "Nite Fields", year: 2012 }, { name: "Refused", year: 1991 }])
```

But the question is: what happens if some of the records in the Array are valid and others are not? Will the valid records be created anyway? Luckily, this is easy to test. We can attempt to create several valid records, and then attempt to create a mix of invalid and valid records all at once:
```
# VALID RECORDS ONLY
Band.create([{ name: "Nite Fields", year: 2012 }, { name: "Refused", year: 1991 }])
# ONE INVALID RECORD
Band.create([{ name: "The Blah Blahs" }, { name: "Radiohead", year: 1985 }])
```
When each Hash contains valid data (passes validations and any database-level checks), the records are created as expected: they are inserted one-at-a-time into the database. When creating a mix of valid and invalid records, we actually get mixed results depending on where the validation checks are occurring.

**Database-level validations using column modifiers only**
When we use column modifiers in our migrations, such as `null: false`, but we don't add validations to our models, `create` will only create records if all of the records are valid. A single invalid record will cause a rollback of the entire transaction. In the code above, records would be created for Nite Fields and Refused, but NOT for The Blah Blahs or Radiohead, because my migration required that neither name nor year could be null.

**Model validations**
Adding model validations

