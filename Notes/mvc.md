# Model-View-Controller Pattern

## What is the MVC Pattern?

The MVC pattern is a software design pattern commonly used for developing web servers by handling different aspects of the application with different functions.

- **Controller**: Handles the client request. Using the information contained on the request (path, params, queries and body), it will invoke the model, which will interact with the data source, and then send a response to the client with the appropriate data.
- **Model**: Handles the fetching, updating, creating and deleting of data, and returns requested data to the controller.
- **View**: Displays data to the user in an easy-to-understand format based on the user’s actions. (_We won't be dealing with views_)

## Why not write an entire server all in one function / file?

As in the examples we have seen so far, our functions can become complex and deeply nested when building a web server. In order to be able to clearly identify what is going on in the server and help debug errors, we can instead have all the logic abstracted away into separate functions. To do this consistently, we will be using the MVC (Model-View-Controller) pattern, splitting our code up into separate functions that are responsible for different things.

Consider the following simple app:

```py
app = FastAPI()


@app.get("/api/dogs")
def get_dogs():
    dogs = db_conn.run('SELECT * FROM dogs;')
    return {"dogs": dogs}


@app.get("/api/dogs/{dog_id}")
def get_dog_by_id(dog_id):
    dogs = db_conn.run(
      'SELECT * FROM dogs WHERE dog_id = :dog_id', dog_id=dog_id
    )
    return {"dog": dogs[0]}
```

As this application grows, there will need to be several more endpoints that interact with the database. Several of which may need to run queries against the dogs table.

To make our code more maintainable, we can create new modules (files) to contain controllers, which handle the request/response, and models that can be used by whichever controllers need them.

```py
# main.py - Some imports like FastAPI and db_conn omitted
from controllers.dogs import get_dogs, get_dog_by_id
app = FastAPI()

# passing controllers to the app directly instead of using
# decorator syntax
app.get("/api/dogs")(get_dogs)
app.get("/api/dogs/{dog_id}")(get_dog_by_id)


# controllers.dogs.py
import models.dogs as dog_models


def get_dogs():
    dogs = dog_models.find()
    return {"dogs": dogs}


def get_dog_by_id(dog_id):
    dog = dog_models.find_by_id(dog_id)
    return {"dog": dog}


# models.dogs.py
def find():
    dogs = db_conn.run('SELECT * FROM dogs;')
    return dogs


def find_by_id(dog_id):
    dogs = db_conn.run(
      'SELECT * FROM dogs WHERE dog_id = :dog_id', dog_id=dog_id
    )
    return dogs[0]
```

## Project Structure

Now that we have an increasing number of functions to deal with, we need a maintainable directory structure to store these files. Keeping each category of function (`controllers`/`models`) separate keeps them distinct, and makes it easier to add more to these directories later.

We have increased the number of functions we have in a simple example, but as the project grows there is a clear structure that several people can contribute to without running into conflicts.

![Example file structure separating out models and controllers](https://assets.northcoders.com/mvc_files.png '380px 480px MVC file structure')

## MVC Flow

When dealing with multiple files, it can be helpful to visualise the flow of data through our servers. Below is an illustration:

![Illustration of the data flow using MVC](https://assets.northcoders.com/mvc_flow_diagram.png '2970px 1088px MVC flow diagram')

**Note:** In the example of a client making a request to our `/api/dogs/:dog_id` endpoint, the controller invokes a single model, but in more complicated endpoints the controller can make use of multiple models from different files in order the process the request.

## Alternative Backend Design Patterns

While the MVC pattern is commonly used, and the one we teach at Northcoders, there are multiple alternative design patterns out there.

Below is a list of other backend design patterns you may encounter, followed by links to some additional reading.

### MVC Alternatives

See [this article on MVC and its variants](https://herbertograca.com/2017/08/17/mvc-and-its-variants/) for more detail.

- HMVC - Hierarchical-Model-View-Controller
- MVVM - Model-View-ViewModel
- MVPVM - Model-View-Presenter-ViewModel

Here is [an in-depth article on MVC MVP, and HMVC(PAC) patterns](https://lostechies.com/derekgreer/2007/08/25/interactive-application-architecture/). [This helpful Stack Overflow post](https://stackoverflow.com/questions/141912/alternatives-to-the-mvc) points towards a myriad of other good examples for further reading.

### Further Reading

- [This Refactoring Guru article](https://refactoring.guru/design-patterns/python) contains some examples of design patterns with Python examples, some of which you'll see on the course.
- [This freeCodeCamp article](https://www.freecodecamp.org/news/design-pattern-for-modern-backend-development-and-use-cases/#real-world-examples-of-design-patterns-in-backend-development) outlines several general backend design patterns that may be useful.
