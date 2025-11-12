# Anna’s challenge 1: python over the internet

Please note there is folder called [Notes](Notes) which contains written teaching material about most things covered in this challenge. Look through it at any point.

## LOs:

- To understand how python can connect to psql databases and run queries using PG8000.
- Create a RESTful API using python to serve data from a backend, using FastAPI.
- To demonstrate unit testing and integration testing for both happy and sad path scenarios
- To build a API that can be used with either test or production data depending on environment variables
- To seed a database using a reusable Python function.
- To understand how FastAPI handles errors by default and custom.

## Tech choices

The frameworks and third-party libraries I normally pointed people to in my previous role were:

- [FastAPI](https://fastapi.tiangolo.com/) - a web framework for building APIs with Python.
- [PG8000](https://pypi.org/project/pg8000/) - a PSQL driver, essentially allows the world of PSQL to interact with the world of python, letting us write psql queries in our python code.
- [pytest](https://docs.pytest.org/en/stable/) - a go-to framework for writing tests in python.
- [python-dotenv](https://pypi.org/project/python-dotenv/) - allowing us to read .env values to setup environment variables.

With an optional extra of [pydantic](https://docs.pydantic.dev/latest/) if we're feeling spicy 🌶️ and want more validation in our endpoints.

BUT YOU CAN FREESTYLE

Want to try a different framework to FastAPI, like Django or Flask then feel free! Anything you feel like playing around with let's try it but I will probably be learning alongside you.

## The brief

Computing Young Fellows (CYF) is a apprenticeship programme in the UK and they want a API created that can serve student/course data through HTTP requests. Let's ignore the security implications of this for now and think about the MVP.

## MVP

### Required endpoints are the following

GET /courses

GET /students

GET /students/{student_id}/feedback

### The following endpoints are nice-to-have

POST /students/{student_id}

DELETE /feedback/{feedback_id}

## Endpoint details

Here are more details about how each endpoint should behave.

[I used generative AI (GPT-5) to populate the specific details about these endpoints]

### GET /courses

    Method: GET

    URL: '/courses'

    Query params (optional):
    - limit (int, default 50)
    - q (string, search text against title)

    Response 200 (application/json):
    Body:
            {
              "total": "int",
              "limit": "int",
              "courses": [
                {
                  "id": "int",
                  "code": "string",
                  "title": "string",
                  "description": "string (nullable)",
                  "created_at": "ISO8601 timestamp",
                  "students_enrolled": "int"
                }
              ]
            }


    Example success:
            {
              "total": 12,
              "limit": 20,
              "data": [
                {
                  "id": 1,
                  "code": "CYF-101",
                  "title": "Intro to Python",
                  "description": "Basics",
                  "created_at": "2025-11-01T09:00:00Z",
                  "students_enrolled": 20
                }
              ]
            }


    Possible errors: What could go wrong here?

### GET /students

    Method: GET

    URL: '/students'

    Query params (optional):
    - limit (int, default 50)
    - course_id (int) — filter by course
    - q (string) — search by name or email

    Response 200 (application/json):
    Body:
            {
              "total": "int",
              "limit": "int",
              "students": [
                {
                  "id": "int",
                  "name": "string",
                  "email": "string",
                  "course_id": "int",
                  "enrolled_at": "ISO8601"
                }
              ]
            }

    Example success:
            {
              "total": 2,
              "limit": 50,
              "courses": [
                {
                  "id": 7,
                  "name": "Aisha Khan",
                  "email": "aisha@example.org",
                  "course_id": 1,
                  "enrolled_at": "2025-10-12T14:00:00Z"
                }
              ]
            }

    Errors: what could go wrong - queries start playing a bigger role here!

### GET /students/{student_id}/feedback

    Method: GET

    URL: '/students/{student_id}/feedback'

    Path params:
    - student_id (int)

    Query params (optional):
    - limit
    - sort (e.g. created_at, desc)

    Response 200:
    Body:
            {
              "student_id": "int",
              "total": "int",
              "limit": "int",
              "feedback": [
                {
                  "id": "int",
                  "author": "string",
                  "status": "string",
                  "comment": "string",
                  "created_at": "ISO8601"
                }
              ]
            }

    Example success:
            {
              "student_id": 7,
              "total": 3,
              "limit": 20,
              "feedback": [
                {
                  "id": 21,
                  "author": "Joe Bloggs",
                  "status": "On Track",
                  "comment": "Great progress",
                  "created_at": "2025-10-25T10:00:00Z"
                }
              ]
            }

    Errors: How could error messages start informing users about what has gone wrong?

### POST /students/{student_id}

      Method: POST

      URL: '/students/{student_id}'

      Path params: student_id (int)

      Request Content-Type: application/json
      Request body (JSON):
              {
                "author": "string", // required
                "status": "string", // required
                "comment": "string" // optional
              }

      Validation rules:
      - author must be present and non-empty (max length 255).
      - status must be a string of either "Ahead", "On-track", "Behind", or "At-Risk"
      - comment optional (max length e.g. 2000).
      - Reject empty body or unknown top-level fields with 400.

      Response: 201 Created, body returns created feedback object:
              {
                feedback: {
                  "id": "int",
                  "student_id": "int",
                  "author": "string",
                  "status": "string",
                  "comment": "string",
                  "created_at": "timestamp"
                }

              }

      Errors: a lot more validation could go wrong here - remember to use logical feedback for the user to know what's gone wrong.

### DELETE /feedback/{feedback_id}

    Method: DELETE

    URL: '/feedback/{feedback_id}'

    Path params: feedback_id (int)

    Response: 204 No Content — successful deletion

    Errors: what could go wrong here?

## Endpoint requirements

You can build these endpoints using the [FastAPI](https://fastapi.tiangolo.com/) framework.

You could choose to add [pydantic](https://docs.pydantic.dev/latest/) into these endpoints to check the incoming data looks correct.

Prioritise the following:

- Clean endpoints, functions extracted into utils where needed, consistent naming.
- Well structured code base, file structure that makes sense.
- Testing throughout, unit testing on any utility functions. High test coverage.

The HTTP endpoints must have integration testing, this can be done using [pytest](https://docs.pytest.org/en/stable/) and FastAPI's TestClient object. There are notes on this.

## Database

This API is reliant on a database existing that it can serve data from. This can be a local psql database that you then connect to using pg8000's Connection object. Notes on [environments](Notes/environments.md) will help shed light on how to connect to a db using pg8000.

To help you create the database, there is some dummy data in the [data](db/data) folder. This data was generated using gpt-5.

We might need to use a 'seed' function to programmatically populate the database with data. This is because as we test endpoints on the database we build more and more ambiguity into the database (e.g. if we test a POST endpoint multiple times, suddenly the database will be clogged with dummy data that stops other test outputs from being predictable - predictability being how we are able to make assertions on what the api will return).

My suggestion is to build a seed function that can re-populate a local database with tables and data pulled from the original data files. A reset button essentially.

For example:

1. Generate a connection using pg8000 to your local psql database.
2. Create a seed function that uses that connection, drops + adds tables, inserts appropriate data.
3. Run this seed function at some point when you run your integration tests so the data stays predictable as we do POSTs and DELETEs on it.

## Environments

This is really a nice to have. We could imagine a scenario where we want a test and production database. And we need to dynamically connect to the test or production database using PG8000 under certain circumstances.

E.g. if I run my tests, `I want to connect to the test database. But if I run the 'seed' script myself I want it to interact with my prod data.

For this kind of "context-switching" we can use environment variables to help us.

I don't want to get into too much detail here, but we can definitely discuss this further. There are notes in this repo for that too.

## Success Markers

- I can make requests to MVP endpoints on this API using insomnia
- The repo has high test coverage which i can check through [coverage](https://coverage.readthedocs.io/en/7.11.3/)
- Repo structure can be justified and makes sense for other developers.
