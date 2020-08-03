# Rails: Create Multiple Records For a Single Model in a Single Request
Creating multiple entries in a database from a single HTTP request using Rails is easier than it seems. We can even ensure that if one fails, our database gets rolled back to its previous state! Ooh la la.

Note that this article talks about creating many records in a single table (model), not many tables (models). Also, this isn't a batch SQL insert: if you're interested in that look into `insert_all`. If you read the whole thing, expect to learn quite a bit about `create(!)` and parameters. If you just want to see the code that changed, scroll down to the Summary at the end.

## What Do We Want to Do?
Let's say we have a table storing Band data, and we want the ability to add several new Bands to this table from the frontend (JavaScript) using a fetch request. Our initial idea might be to loop through all of the new Band data on the frontend and send one fetch request per Band. However, this results in making more requests than necessary. Instead, we can actually add all of the new Bands using a single fetch request, which reduces the number of requests our server must handle.

The fetch request we're aiming for might look something like this:
```
const options = {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ bands }) // bands is an array of Objects
}

// fetch here
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
Adding model validations for the name and year columns changes the behavior of `create`. First let's put the invalid record at the start of the Array:
```
# RADIOHEAD IS INSERTED
Band.create([{ name: "The Blah Blahs" }, { name: "Radiohead", year: 1985 }])

# RADIOHEAD AND TESTING ARE INSERTED
Band.create([{ name: "Radiohead", year: 1985 }, { name: "The Blah Blahs" }, { name: "Testing", year: 2222 }])
```
Unlike before, this actually resulted in all valid records being inserted into the database regardless of their location in the Array.

But what about `create!`? Is there a difference?
```
# NOTHING IS INSERTED, ABORTED ON FIRST INVALID RECORD
Band.create!([{ name: "The Blah Blahs" }, { name: "Radiohead", year: 1985 }])

# RADIOHEAD IS INSERTED, ABORTED ON FIRST INVALID RECORD
Band.create!([{ name: "Radiohead", year: 1985 }, { name: "The Blah Blahs" }, { name: "Testing", year: 2222 }])
```
This is actually similar to before. As soon as an invalid record was encountered, the process was aborted. The Blah Blahs, sadly, are still floating in space waiting for their time to shine.

**Return values and exceptions**
When only column modifiers are present, both `create` and `create!` raise exceptions when any invalid records are present in the Array, and return an Array of records when all records are valid. As a result, if an invalid record is present, attempting to store the result in a variable will cause the variable to equal nil or its previous value, if it had one:

```
a = Band.create([{ name: 'Bandy McBandFace', year: 3000 }, { name: 'not happening' }])
# a => nil

a = 'dance magic dance'
a = Band.create([{ name: 'Bandy McBandFace', year: 3000 }, { name: 'not happening' }])
# a => 'dance magic dance'
```

When model validations are present and only valid records exist in the Array, `create` will return an Array of valid objects representing the entries that were just created, including their IDs. When creating a combination of valid and invalid records `create` returns all of those records, but the invalid records will have an ID of nil. If all of the records are invalid, `create` will still return an Array of those objects with IDs set to nil.

`create!` operates differently. If all of the records are valid, it will return them in an Array with their IDs set. If one record is invalid, regardless of where it's located in the Array, it will raise an exception. If we try to set a variable to the return value of `create!` when it raises an exception, the value stored in that variable will either be nil or the variable's previous value. `create!`, in other words, behaves just as it does when only column modifiers are present.

**Summary: create vs create!**
- Both take a Hash or Array of Hashes as an argument.
- Both return an Array of objects when all records being created are valid.
- When only column modifers are present, both raise exceptions upon encountering an invalid record and stop inserting any further records. No records are returned.
- When model validations are present, `create` inserts all valid records and returns an Array of objects. Objects representing invalid records will have IDs set to nil in the returned Array.
- When model validations are present, `create!` behaves just as it does when no model validations are present.

> Note: the specific exception that's raised differs when handling a failed database-level validations vs failed model validations!

## Accepting Arrays from HTTP Requests
Now that we know the differences between the two creates (it's important, I promise!), we can move on to our BandController. We're probably pretty used to creating a single record where our parameters are expecting to a simple Hash of data:

```
def create
  band = Band.new(band_params)

  if band.save
    render json: band
  else
    render json: { errors: band.errors.full_messages }
  end
end

private

def band_params
  params.require(:band).permit(:name, :year)
end
```

The code above is pretty straightforward: `band_params` parses the parameters from the request and returns a Hash-like object, which `Band.new` uses to create a new Band object. That data can either be saved to the database or not depending on its validity. So how do we change this to accept an Array of Bands instead of a single Band.

**Updating the strong params**
First, our strong parameters need to be updated to accept an Array of data. Instead of going straight to updating the controller, let's play with `Parameters` in the Rails console. We'll create a new `Parameters` object using the same data format we expect to receive from the frontend:

```
bands = { bands: [{ name: "Radiohead", year: 1985 }, { name: "Refused", year: 1991 }] }
params = ActionController::Parameters.new(bands)
# => <ActionController::Parameters {"bands"=>[{"name"=>"Radiohead", "year"=>1985}, {"name"=>"Refused", "year"=>1991}]} permitted: false>
```

Now let's try running these parameters through our regular strong parameters method that expects to receive a Hash...just for fun (because there's no chance it's going to work! Look at me knowing what fun is):

```
params.require(:band).permit(:name, :year)
# => ActionController::ParameterMissing (param is missing or the value is empty: band)

params.require(:bands).permit(:name, :year)
# => NoMethodError (undefined method `permit' for #<Array:0x00007fc530ea8880>)
```

In both cases, exceptions were raised. First, it was because the required parameter/key `band` was missing. In the second case, we updated the requirement to `bands`, but we received an error stating that `permit` can't be called on an Array. This means that calling `require` returned an Array, and `permit` just wasn't into any of that business. 

This is actually important to keep in mind. When `require` is called on the parameters in this example, it returns the value associated with the `bands` key, which happens to be an Array. In other words, `require` returns values, and not key-value pairs.

So how can we fix this? Well, what if we just get rid of `require` and tell `permit` to accept an Array of data:
```
params.permit(bands: [:name, :year])
# => <ActionController::Parameters {"bands"=>[<ActionController::Parameters {"name"=>"Radiohead", "year"=>1985} permitted: true>, <ActionController::Parameters {"name"=>"Refused", "year"=>1991} permitted: true>]} permitted: true> 
```

We didn't get an error, but this still isn't quite what we want. We now have a `Parameters` object that has our Band data attributed with a `bands` key in a Hash. Both `create(!)`s are expecting an Array. Let's give it a whirl to check our hypothesis:

```
Band.create(params.permit(bands: [:name, :year]))
# => ActiveModel::UnknownAttributeError (unknown attribute 'bands' for Band.)
```

Our good ol' friend the exception is telling us that our hypothesis was right. The `create` method received a Hash-like object with an unknown attribute, and not an Array. So what can we do to fix this? `require` will return the value associated with the `bands` key, so let's chain that on and see what happens:

```
params.permit(bands: [:name, :year]).require(:bands)
# => [<ActionController::Parameters {"name"=>"Radiohead", "year"=>1985} permitted: true>, <ActionController::Parameters {"name"=>"Refused", "year"=>1991} permitted: true>] 

Band.create(params.permit(bands: [:name, :year]).require(:bands))
# => Created the records and returned them in an Array
```

Wow! That worked!! We called `permit` first, so that we could accept an Array of data. That returned a `Parameters` object containing more `Parameters` structured as a Hash. To get the Array associated with the `bands` attribute from there, we just had to call `require`. Not too shabby.

We can now update our strong parameters method like so:
```
# changed the method's name, FYI
def bands_params
  params.permit(bands: [:name, :year]).require(:bands)
end
```

> Note: For this example, I'm assuming all requests will pass an Array of Bands. If we wanted to accept both Arrays and Hashes, we could set up a separate route or use additional logic in the `create` action to handle both types of data.

**Updating the create action**
Our `create` action is currently expecting to add a single record to the database. We'll need to change this so that it creates multiple records. Here is our code so far with the updated strong params method:

```
def create
  band = Band.new(band_params)

  if band.save
    render json: band
  else
    render json: { errors: band.errors.full_messages }
  end
end

private

def bands_params
  params.permit(bands: [:name, :year]).require(:bands)
end
```

This most certainly is not going to work...at all.

At the moment, our Rails project has database-level validations (column modifiers) and model validations. This means that any invalid requests will fail on the model validations, since they directly mirror the column modifiers. We need to keep this in mind as we move forward, since it affects how we handle requests.

First, let's try making the simplest change we can think of to our `create` action:
```
def create
  bands = Band.create(bands_params)
  render json: bands
end
```

First we'll see what happens when we send only valid data:
```
options = {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ 
    bands: [ 
      { name: 'oh yeah', year: 3000 }, 
      { name: 'wee bitz', year: 1980 } 
    ] 
  })
}

fetch('http://localhost:3000/bands', options)
  .then(res => res.json())
  .then(console.log)
  .catch(console.log)
// => Array of Objects representing Band objects with IDs set
```

Sending only valid data, resulted in all of our records being created and an Array of data being sent back in response. But what will happen if we send a mix of valid and invalid data?:

```
options = {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ 
    bands: [ 
      { name: 'i am first', year: 1 }, 
      { name: 'i feel so alone' } 
    ] 
  })
}

fetch('http://localhost:3000/bands', options)
  .then(res => res.json())
  .then(console.log)
  .catch(console.log)
// => Array of Objects representing Band objects with one ID set and one set to null
```

Both records were returned regardless, but only one was created in the database. On the frontend, we could choose to pinpoint the bad records, fix them, and then resend our request, but I'd prefer to go a different route: let's handle our requests in an all-or-nothing manner. All records must be valid for any insertions to persist. 

## Creating Records via Transactions
We're about to make a new friend: a class method called `transaction`. Transactions are protective blocks that help protect the integrity of a database. Any SQL statements executed inside a transaction will persist only if all of the SQL statements were executed successfully. A single failure will cause the database to roll back to its previous state. In other words, it's all or nothing!

For the transaction to do its job, we have to raise an exception when an insertion fails (The API documentation is really beautiful when it comes to transactions). This means we have to say goodbye to our friend `create` and make `create!` our new bestie. Let's modify our controller code to utilize transactions:

```
def create
  Band.transaction do
    # using an instance variable to make it available to render
    # otherwise we'd have to declare bands in the outer scope as nil
    @bands = Band.create!(bands_params)
  end
    
  render json: @bands
end
```

After sending a fetch request with only valid data, we receive the response we expect: an Array of Hashes containing valid Band records. But what happens if there's invalid data in the request? First of all, no new records are created, the database is restored to its pre-HTTP-request state, so yay. In the web page's console, we get an error message (the angry red one) stating "422 (Unprocessable Entity)", and from the second `then()` we log an Object that looks like this:

```
{status: 422, error: "Unprocessable Entity", exception: "#<ActiveRecord::RecordInvalid: Validation failed: Year can't be blank>", traces: {…}}
```

It doesn't have to be this way. We could handle the exception that's raised in the controller if we prefer to send a different response.

**Handling the exception**
To handle the exception, we'll make yet another friend, the `begin-end` block, because that's what it does! Luckily, this requires only a small change to the `create` action:

```
def create
  begin
    Band.transaction do
      @bands = Band.create!(bands_params)
    end
  rescue ActiveRecord::RecordInvalid => exception
    # omitting the exception type rescues all StandardErrors
    @bands = {
      error: {
        status: 422,
        message: exception
      }
    }
  end
  
  render json: @bands
end
```

Now if a record is invalid, our custom response will be sent instead of the one generated by Rails:
```
{
  error: {
    message: "Validation failed: Year can't be blank",
    status: 422
  }
}
```

We can now post multiple Bands at once to the database and send custom error messages when things go wrong!

## Summary
Here's a quick rundown of how to post multiple records embedded in an HTTP request with a single request:

1. Update the strong params (or declare a new method) in the controller to accept an Array of data
2. Update the controller action (or declare a new one) to create records using `create` or `create!`
3. If you want an all-or-nothing outcome, use a transaction and `create!`
4. If you also want to respond with a custom error message, handle the exception created by the transaction

And here's the BandController at the end of all this wordy madness:
```
class BandsController < ApplicationController
  def create
    begin
      Band.transaction do
        @bands = Band.create!(bands_params)
      end
    rescue ActiveRecord::RecordInvalid => exception
      @bands = {
        error: {
          status: 422,
          message: exception
        }
      }
    end
  
    render json: @bands
  end
  
  private
  
  def bands_params
    params.permit(bands: [:name, :year]).require(:bands)
  end
end
```
[GitHub Repo](https://github.com/helloRupa/mass_create_example)

Why so many words for so little code? I'm a deep sea diver who operates on land. Sometimes I can't help myself.