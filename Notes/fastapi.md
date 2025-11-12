# FastAPI

## Introducing FastAPI

FastAPI describes itself as a:

> ...modern, fast (high-performance), web framework for building APIs with Python 3.8+ based on standard Python type hints.

In simple terms, it does everything that native Python HTTP Servers do, but it's a lot nicer to work with.

To install FastAPI, run `pip install "fastapi[standard]"` within your virtual environment.

We can use the following code to setup our FastAPI app, which will be an instance of the `FastAPI` class.

```python
/// main.py
from fastapi  import FastAPI

app = FastAPI()
```

It's important that the entry point to your app, above, is saved in a file named `main.py` at the root of your application directory.

Notice that we're not passing a handler to the server. That is part of what FastAPI does for us by default. It will handle requests and responses through its middleware layer.

We also haven't specified a port to listen on. This is because FastAPI uses [Uvicorn](https://www.uvicorn.org/), an ASGI web server implementation, which runs the server on a default port of `8000`.

We can run the server by using the CLI command below, referring to the `main.py` file as the entry point - where `app`, our FastAPI instance, is defined:

```bash
fastapi dev main.py

╭────────── FastAPI CLI - Development mode ───────────╮
│                                                     │
│  Serving at: http://127.0.0.1:8000                  │
│                                                     │
│  API docs: http://127.0.0.1:8000/docs               │
│                                                     │
│  Running in development mode, for production use:   │
│                                                     │
│  fastapi run                                        │
│                                                     │
╰─────────────────────────────────────────────────────╯

```

And that's it: Your server is running with 2 lines of code and a CLI command!

### Basic API Endpoint

The FastAPI has methods, used as decorators, which take the **path** of the endpoint as their first argument.

```python
from fastapi  import FastAPI

app = FastAPI()


@app.get("/")
def root():
    return {"message": "Hello World"}
```

The first argument passed to the decorator is the path `'/'`. Notice that these parameters are passed directly to the path operation decorator, not to your path operation function.

Any time we get a `GET` request on the `'/'` route, FastAPI will invoke the `root` function.

### Basic Routing

Previously, using `http.server`, we would need to build the response manually with `json.dumps()` before sending it with `self.wfile.write(data.encode("utf-8"))`.

In FastAPI, we can be far more succinct and achieve the same with the following:

```python
from fastapi  import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello World"}

@app.get('/api/dogs')
def dogs():
    return {"dogs": [{"name": "fido", "good_doggo": True}, {"name": "brute", "good_doggo": False}]}

```

Any time we get a `GET` request on the `'/api/dogs'` route, the Python `dict` is returned to the client as JSON.

The above method will update the headers in the request objects in order to inform the browser that it is being sent JSON. The status code is updated to 200 for a successful response, and the actual response is turned from a Python dictionary with nested Python lists to a JSON object and nested array before being sent back to the client.

It also handles the `404: "path not found"` error for you, sending the following for any invalid endpoints:

```json
{
  "detail": "Not Found"
}
```

This behaviour can also be further [customised](https://fastapi.tiangolo.com/tutorial/handling-errors/).

### Parametric Endpoints and Variables

FastAPI allows us to pass in a variable name into our path within curly braces as illustrated in [the FastAPI documentation](https://fastapi.tiangolo.com/tutorial/path-params/). The name will be passed into your path operation function as an argument of the same name, with its value being that of the section of the path that FastAPI actually received in the request.

```python
 from fastapi  import FastAPI

app = FastAPI()

dogs = [{"name": "fido", "good_doggo": True}, {"name": "brute", "good_doggo": False}]

@app.get('/api/dogs/{dog_name}')
def dogs(dog_name: str):
    dog = filter(lambda dog : dog["name"] == dog_name, dogs)
    return {"dog": dog}

```

It's worth noting here that when creating variable endpoints, [the order which you declare your path operations matters](https://fastapi.tiangolo.com/tutorial/path-params/#order-matters) - especially where the variable's value could plausibly be the same as an _existing path_. For example, where `/api/dogs/mydog` is used as an endpoint for a dashboard of your own dogs, but some other user has called their dog 'mydog'.

### Queries

Similarly to **parametric endpoints**, FastAPI makes handling **queries** much easier than when using `http.server`'s `BaseHTTPRequestHandler` class.

```python
#...
dogs = [{"name": "fido", "good_doggo": True}, {"name": "brute", "good_doggo": False}]

@app.get('/api/dogs')
def dogs(good: bool | None = None):
    if good is not None:
        return {"dogs": filter(lambda dog : dog["good_doggo"] is good)}
    else:
        return {"dogs": dogs}

```

With the above example, the URL `/api/dogs?good=True` would respond with:

```json
// Headers:
// code           200
// content-type   application/json

{
  "dogs": [
    {
      "name": "fido",
      "good_doggo": "True"
    }
  ]
}
```

More information about queries can be found in the [FastAPI docs](https://fastapi.tiangolo.com/tutorial/query-params/).

## Pydantic Models

FastAPI is based on a data handling and validation library called [Pydantic](https://docs.pydantic.dev/latest/). As such, it handles a huge amount of the grunt-work of the request/response cycle for you.

To fully take advantage of this, and use FastAPI to its greatest effect, you should include Python type declarations. We can, and should, go further if we want to build a robust and secure API.

The following is an example of a POST endpoint, where the request body must conform to a [**data model**](https://fastapi.tiangolo.com/tutorial/body/#request-body), declared as the class `NewDog`:

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel


class NewDog(BaseModel):
    name: str
    good_doggo: bool


app = FastAPI()

@app.post("/api/dogs")
async def add_dog(new_dog: NewDog):
    return {
                "message": "Congratulations! We've added your dog!",
                "dog": new_dog
          }
```

> For more information about FastAPI, be sure to check out their [documentation](https://fastapi.tiangolo.com/)!
