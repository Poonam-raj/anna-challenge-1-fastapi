# API Error Handling

## Errors in Context

From the [FastAPI docs](https://fastapi.tiangolo.com/tutorial/handling-errors/):

> There are many situations in which you need to notify an error to a client that is using your API.
>
> This client could be a browser with a frontend, a code from someone else, an IoT device, etc.
>
> You could need to tell the client that:
>
> - The client doesn't have enough privileges for that operation.
> - The client doesn't have access to that resource.
> - The item the client was trying to access doesn't exist.
> - etc.
>
> In these cases, you would normally return an HTTP status code in the range of 400 (from 400 to 499).
>
> This is similar to the 200 HTTP status codes (from 200 to 299). Those "200" status codes mean that somehow there was a "success" in the request.
>
> The status codes in the 400 range mean that there was an error from the client.
>
> Remember all those "404 Not Found" errors (and jokes)?

## Standard HTTP Error Responses

This is _not_ a definitive list, but common error responses include:

### GET All

- `/notARoute` -> route that does not exist: **404 Not Found**

### GET by ID

- `/api/resource/999999999` -> resource that does not exist: **404 Not Found**
- `/api/resource/notAnId` -> invalid ID: **400 Bad Request**

### POST

- `/api/resource` body: `{}` -> malformed body / missing required fields: **400 Bad Request**
- `/api/resource` body: `{ rating_out_of_five: 6 }` -> failing schema validation: **400 Bad Request**

### DELETE by ID

- `/api/resource/999999999` -> resource that does not exist: **404 Not Found**
- `/api/resource/notAnId` -> invalid ID: **400 Bad Request**

### PATCH/PUT by ID

- `/api/resource/{id}` body: `{}` -> malformed body / missing required fields: **400 Bad Request**
- `/api/resource/{id}` body: `{ increase_votes_by: "word" }` -> incorrect type: **400 Bad Request**

([The HTTP Cats site](https://http.cat/), however, _is_ the definitive list.)

## HTTPException

Within most API frameworks, handling errors using `print` or `raise` will not provide the user/client with any information as to why they have not received the expected response. In the case of a runtime error, the usual default response is "500 Internal Server Error", which is less than helpful, and is certainly not an example of human-centred design.

Fast API's solution to this is to provide an `Exception` class - `HTTPException` - with specific behaviours and integration into the API's [**middleware**](https://www.ibm.com/topics/middleware).

```python
from fastapi import FastAPI, HTTPException
```

In a FastAPI, raising an `HTTPException` within any function called by a **path operation function** will stop its execution and send an error response to the client.

### Raising an HTTPException

Below is a very simple example of how this might work:

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

items = {"foo": "The Foo Wrestlers"}


@app.get("/api/resource/{resource_id}")
def read_item(item_id: str):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item": items[item_id]}
```

Instead of sending back a `500 Internal Server Error`, a standard 404 response is sent back, accurately communicating the result of their request.

## FastAPI with pg8000

Let's look at the behaviour of, and interaction between, our database and API and translate that into a meaningful response for the client.

The following endpoint is defined in our API:

```python
@app.get("/api/things/{thing_id}")
def get_thing_by_id(thing_id):
    '''Returns a single thing.'''
    db = Connection('postgres')
    res = db.run(
        f'SELECT * FROM things WHERE thing_id={literal(thing_id)}')
    return {"thing": res}
```

The client makes a request to the following URL:

```http
METHOD: GET
URL: /api/things/999999
```

They receive the following response:

```python
{
  "thing": []
}
```

### A better solution

The following would be a more informative response for the client:

```json
// Status 404
{
  "detail": "No thing found with that ID"
}
```

This is how that would be achieved by raising an `HTTPException`:

```python
@app.get("/api/things/{thing_id}")
def get_thing_by_id(thing_id):
    '''Returns a single thing.'''
    db = Connection('postgres')
    res = db.run(
        f'SELECT * FROM things WHERE thing_id={literal(thing_id)}')
    if not res: # Remember an empty list in Python is falsy
        raise HTTPException(status_code=404, detail="No thing found with that ID")
    return {"thing": res}
```

### PG8000.exceptions

When communicating with a PostgreSQL database using pg8000, there are some common exceptions raised by pg8000 that can be handled explicitly.

Taking the same endpoint above, a client sending a GET request to `/api/things/thingy_mcthing_face` would force FastAPI to send a `500 Internal Server Error`. This is because pg8000 will have raised a`DatabaseError` exception at runtime, and FastAPI has not explicitly handled that exception.

```
pg8000.exceptions.DatabaseError: {
'S': 'ERROR',
'V': 'ERROR',
'C': '22P02',
'M': 'invalid input syntax for type integer: "thingy_mcthing_face"',
'P': '43',
'F': 'numutils.c',
'L': '235',
'R': 'pg_strtoint32'
}
```

We can, however, `except` that error.

> It's important to note here that - _with any API_ - when sending error responses to a client, the response should be **standardised** and **sanitised**.
>
> In other words: the response should be in **standard HTTP error codes** with corresponding messages, and that **no details of the database should be revealed** that could open it up to exploitation.

With this in mind, it would be bad practice to just send the raw pg8000 exception object to the client. This could open up your database for exploitation, but more likely would become cumbersome for the client to interpret.

The following is a good example of a standardised and sanitised exception being passed to the client:

```python
@app.get("/api/things/{thing_id}")
def get_thing_by_id(thing_id):
    '''Returns a single thing.'''
    db = Connection('postgres')
    try:
        res = db.run(
            f'SELECT * FROM treasures WHERE treasure_id={literal(treasure_id)}')
    except DatabaseError as db_err:
# Access the error response from the database directly
        err = db_err.args[0]
        code = err['C']
        message = err['M']
# Make the behaviour specific to this type of error.
        if code == '22P02':
            raise HTTPException(
                status_code=400,
# Give a contextual response to the client, sending any other sanitised information that could be useful
                detail=f"Treasure Id's should be a number - {message}")
    if not res:
        raise HTTPException(status_code=404, detail="Treasure not found")
    return {"thing": res}
```

Note that, above:

- The `as` keyword must be used here, as the DatabaseError needs to be instantiated within the context of the except statement before we are able to interact with it.
- The error object has a method, `.args`, from which we can access the arguments that were passed into the class on invocation of the constructor. This will contain the response from the database itself.
- `.args` returns a tuple, where the first element is the dictionary that the Postgres database responded with when the error occurred.

This error has propagated from the database and has been passed all the way through our API's **middleware** to the client. There are times when this is not necessary, and it would be preferable to handle the error more efficiently, as you'll see in the next section.

##  Getting pedantic with Pydantic

In some cases, such as the one above, Pydantic types can enhance FastAPI's behaviour in a way that '[defines errors out of existence](https://milkov.tech/assets/psd.pdf#page=84)'. This means that `FastAPI` will step in before we even need to handle an exception, reducing code complexity and the load on you as a developer.

```python
@app.get("/api/things/{thing_id}")
def get_thing_by_id(thing_id: int): # <<<<< This is all you might need.
    '''Returns a single thing.'''
    db = Connection('postgres')
    if not res:
        raise HTTPException(status_code=404, detail="Treasure not found")
    return {"thing": res}
```

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "thing_id"],
      "msg": "Input should be a valid integer, unable to parse string as an integer",
      "input": "thingy_mcthing_face",
      "url": "https://errors.pydantic.dev/2.5/v/int_parsing"
    }
  ]
}
```

## Custom Exception Handlers

So that the **path operation functions** of your endpoints don't become bloated with code, it will become necessary to extract some of the error-handling logic away from the controller functions.

This can be achieved with custom exception handlers using the `@app.exception_handler(<Exception>)` decorator.

As always, [FastAPI's documentation](https://fastapi.tiangolo.com/tutorial/handling-errors/#install-custom-exception-handlers) is a great source for learning more about this.

##  Conclusion

When handling exceptions, it all comes down to your knowledge of the frameworks you are using, their behaviour, and whether or not that behaviour is meaningful for the client. The bottom line is that standard HTTP error codes and corresponding messages should be what reach the client in the event of any runtime error.
