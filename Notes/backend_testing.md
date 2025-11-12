# Testing Backend Applications

## Integration Testing

Up to this point, the vast majority of our tests have been **unit tests** - testing individual functions and methods on a granular level.

Another type of testing is **integration testing**. Unlike unit testing, which tests individual functions, integration testing is oriented towards checking that multiple parts of a project work cohesively together to successfully achieve their goal.

This is useful when developing a server, where each endpoint will require the controllers, models, and database to work together to respond with the correct data. An **integration test** here could make a request to the server and test that the response is correct, meaning that all the distinct moving parts have worked in the intended way.

> Because testing an endpoint involves testing the flow from a user request all the way to the database, it could be argued that there is overlap here with **end-to-end testing**. Also known as 'E2E Testing', this refers to tests that check an entire user journey.

### Unit tests for servers

> **"Tests must ALWAYS be written first"** - _most good back-end developers, ever_

Let's suppose we have a small number of movies in a data set:

```python
movies = [
      {'id': 1, 'title': 'Endless Love', 'rating': 3},
      {'id': 2, 'title': 'Love Story', 'rating': 2},
      {'id': 3, 'title': 'The Texas Chainsaw Massacre', 'rating': 16}
]
```

We intend to access these with a simple API with a single endpoint of the form `/movies/{movie_id}`.

The app looks like this:

```python
from fastapi import FastAPI, HTTPException
from typing import Union

app = FastAPI()

def find_movie_in_data(id: int, movie_data: list[dict]) -> Union[dict, None]:
    return next((m for m in movie_data if m['id'] == id), None)

@app.get("/movies/{movie_id}")
def get_movie(movie_id: int) -> dict:
    result = find_movie_in_data(movie_id, movies)
    if not result:
        raise HTTPException(status_code=404, detail=f'No movie with id {movie_id}')
    return result
```

The two functions are tested like this:

```python
from api import get_movie, find_movie_in_data
from fastapi import HTTPException
import pytest


def test_find_in_data():
    movies = [
      {'id': 1, 'title': 'Endless Love', 'rating': 3},
      {'id': 2, 'title': 'Love Story', 'rating': 2},
      {'id': 3, 'title': 'The Texas Chainsaw Massacre', 'rating': 156}
    ]
    assert find_movie_in_data(1, movies) == {'id': 1, 'title': 'Endless Love', 'rating': 3}
    assert find_movie_in_data(23, movies) is None


def test_get_movie():
    assert get_movie(2) == {'id': 2, 'title': 'Love Story', 'rating': 2}


def test_get_movie_error():
    with pytest.raises(HTTPException):
        get_movie(56)
```

The app is correctly _unit tested_ because each of the component functions has been separately tested for its own correct individual behaviour. However, there is no overall test that a call to an endpoint like `/movies/2` will produce the correct result, or that a call to `/movies/73` will return the correct error message and status code.

### Creating integration tests

FastAPI provides a straightforward interface for creating tests for server-side route handling. This allows us to test APIs as they will actually be used - by specifying calls to the various endpoints and checking the responses. The key component is called `TestClient`.

We can create new tests to use this client.

```python
from api import app
from fastapi.testclient import TestClient

client = TestClient(app)

# Integration tests...

def test_movies_200():
    expected_json = {'id': 2, 'title': 'Love Story', 'rating': 2}
    response = client.get('/movies/2')
    assert response.status_code == 200
    assert response.json() == expected_json


def test_movies_404():
    expected_json = {'detail': 'No movie with id 888'}
    response = client.get('/movies/888')
    assert response.status_code == 404
    assert response.json() == expected_json
```

The error-handling behaviour of your API should be fully tested. Test all 'happy path' behaviour (i.e. all the code `200`s). Then, begin to write tests for each error response in turn. Follow the normal test-driven development cycle (red, green, refactor) when error handling.

## Before and After Tests

Integration tests might make use of test infrastructure such as a dummy database. If so, we may need to think carefully about the
"Arrange" part of our "Arrange, Act, Assert" procedure for testing.

Consider the consequences of testing a GET /items endpoint and then a POST /items endpoint. As the POST request would actually be
adding data to the database, the next time the test suite is run and the GET /items endpoint is tested, there will be more items in the database than were originally seeded. Each test that is run will interact with the test database and change the data that is stored.

Ideally, we would set up the database in an identical fashion before each test run and then take it down when finished.

In pytest, the recommended way to do this is via a fixture. Let's suppose we have written two scripts to `seed` and `tear_down` the
database. We can write a fixture like this:

```python
import pytest
from pg8000.native import Connection

@pytest.fixture(scope=function)
def database():
    creds = #... get test database server credentials
    con = Connection(**creds)
    seed(con) # run seeding function
    yield con # yields the database connection to the requesting test
    tear_down(con) # clears the database
    con.close()

def test_some_database_thing(database):
    # test code here

```

We are making use of the `yield` statement to pause and resume execution. The fixture runs the seeding procedure and provides
the required connection to the database before pausing. When the test is finished, execution resumes, allowing us to run the code
that clears the database and closes the connection. By choosing `scope=function`, we are making this happen for every test
individually. We could also, for example, choose the scope to be the module so that all the tests in the module are run with the same fixture.

Further detail on using `yield` as a method for setting up/tearing down integration test fixtures can be found in the
[pytest docs](https://docs.pytest.org/en/6.2.x/fixture.html#teardown-cleanup-aka-fixture-finalization).
