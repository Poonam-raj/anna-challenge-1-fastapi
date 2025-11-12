# Environments

## What is an Environment?

A **software environment** refers to the combination of hardware, software, and data configurations that determine how a computer program behaves. It encompasses various elements that influence the execution of a software application.

Key components of a **software environment** include:

- Operating system (OS)
- Programming language runtimes
- Libraries and frameworks
- Development tools
- Database systems
- Networking protocols
- Version control systems
- Configuration management
- Deployment infrastructure
- Security measures

## Different Environments

In software development, environments are separated to serve specific purposes in the **software development lifecycle** (SDLC).

### Development (Dev) Environment

The development environment is where developers write and test new code. It's a controlled environment where programmers can experiment, troubleshoot and collaborate on building new features or fixing bugs.

- Typically isolated from the production environment to prevent accidental disruption.
- Contains tools and configurations that aid development and debugging.
- May have simulated or partial data to mimic the production environment.
- Frequent changes and updates occur as developers work on code.

###  Testing Environment

The testing environment is where software is tested comprehensively before it is deployed to the production environment. It helps ensure that the application behaves as expected, is free of critical bugs and meets the specified requirements.

- Represents a more realistic environment than the development environment but is still separate from production to avoid affecting end users.
- Contains a copy of the production database or a subset of it.
- Various types of testing, such as unit testing, integration testing and user acceptance testing, are performed here.
- Changes are less frequent compared to the development environment.

### Production (Prod) Environment

The production environment is the live environment where the actual end users interact with the software. It hosts the final, fully-tested version of the application that has passed all quality assurance checks.

- Represents the real-world scenario where users access and use the software.
- Highly stable and secure, with configurations optimized for performance and reliability.
- Data in the production environment is real and not simulated.
- Changes are carefully planned, scheduled and monitored to minimize disruptions to users.

### Why are they necessary?

- **Risk Mitigation:** Separating development, testing and production environments helps mitigate the risk of introducing bugs or issues to the live application. Changes are thoroughly tested before reaching production.

- **Isolation and Control:** Each environment serves a specific purpose, providing isolation and control. Developers can experiment freely in the development environment without impacting users, and testing is conducted in a controlled setting.

- **Quality Assurance:** Testing environments allow for comprehensive testing, including functional, performance and security testing, ensuring that the software meets the required standards before being deployed to production.

- **Stability and Reliability:** The production environment is optimized for stability and reliability to ensure uninterrupted service for end users. It undergoes careful planning and monitoring to minimize downtime and disruptions.

## Environment Variables in Python

An **environment variable** is a dynamic-named value that can affect the way processes behave on a computer. These variables are part of the environment in which a process runs, and they provide a way for software applications and the operating system to share configuration settings.

In Python, environment variables be accessed by Python scripts during runtime.

### Accessing Environment Variables

The native Python `os` module is commonly used to interact with the operating system, including accessing environment variables.

The `os.environ` object is a dictionary-like object that represents the current environment variables.

### Setting Environment Variables

Environment variables can be set within a Python script using the `os.environ` object.

For example, to set a variable named `MY_VARIABLE` with the value `"Hello"`:

```python
    import os
    os.environ["MY_VARIABLE"] = "Hello"
```

### Retrieving Environment Variables

To retrieve the value of an environment variable, you can use the `os.environ.get()` method.

```python
    import os

    database_url = os.environ.get("DATABASE_URL")
    if database_url:
        print(f"Connected to database at {database_url}")
    else:
        print("DATABASE_URL not set.")
```

> **Security considerations**: We must however be cautious when using environment variables for sensitive information, as they may be accessible to other processes or users on the system.

## Separating environments programmatically

Until now, we have set the name of the database manually, on initialisation of the `pg8000.native.Connection` class.

This, however, becomes cumbersome when developing software within a **continuous integration/continuous development** workflow, and hard-coding connection credentials into code or environment variables is the antithesis of secure coding practice. Anyone with access to the source code of your application could easily connect to your production database...

> ```sql
> DROP DATABASE IF EXISTS stupidly_insecure_database
> ```

Bear in mind that there are no set conventions on how to configure separated environments, so this is but one example. Always refer to the docs of the various frameworks you are using and, in the case of developing within an already well-established codebase, keep to the patterns of the code upon which you are building.

### Using python-dotenv

`python-dotenv` is a `pip` [package](https://pypi.org/project/python-dotenv/) that handles the configuration of environment variables.

Once it is installed, we can create a file called `.env` and store our variables in there.

```txt
PG_USER=postgres
PG_HOST=localhost
PG_DATABASE=database_mcdataface
PG_PORT=5432
PG_PASSWORD=postgres
```

An important benefit of this is that we can `gitignore` this file so that we do not push any sensitive information about our database and/or postgres passwords up to GitHub.

In a dedicated module at the root of our application, we can set up our environment configuration by invoking the [`load_dotenv()`](https://saurabh-kumar.com/python-dotenv/reference/#dotenv.load_dotenv) method.

This sets all of the environment variables from the `.env` file to the `os.environ` object.

In the example below, we have then set them to globally scoped variables, making them available in other modules via one interface.

```python
#./config.py
import os
from dotenv import load_dotenv

load_dotenv()

user = os.getenv('PG_USER')
host = os.getenv('PG_HOST')
database = os.getenv('PG_DATABASE')
port = os.getenv('PG_PORT')
password = os.getenv('PG_PASSWORD')
```

Now, wherever we require these variables, we can access them via the module - `config` - within which they were declared using dot notation.

```python
# main.py
import config
from os import getenv
from pg8000.native import Connection
from dotenv import load_dotenv

connection = Connection(
    config.user,
    host=config.host,
    database=config.database,
    port=config.port,
    password=config.password
)
```

## Separate test and development databases

Running tests can cause updates to the data within a database, which can make the state of the data in the database unpredictable across tests.

An added benefit of this is that the test data can be curated by the developers to be a smaller, more manageable subset of the development data.

This means that test data can be re-seeded quicker and is easier to update and write tests for.

### Connecting to different PostgreSQL databases

In order to use pg8000 to connect to environment-specific databases, the `config.py` file needs to be configured to programmatically change which database we are connecting to based on the environment.

A common way to achieve this is with multiple `.env` files.

In `.env.test`:

```txt
...
PGDATABASE=test_database_name
...

```

In `.env.development`:

```txt
...
PGDATABASE=development_database_name
...
```

In the config module, we need to configure our code so that it can define values to the relevant environment variables based on the environment.

This is made easier by the fact that `load_dotenv()` accepts a file path for the specific `.env` file you would like to load.

In order to take advantage of this, we can explicitly set an environment variable when running our tests in the command line:

```bash
PYTHON_PATH=$(pwd) TESTING=True pytest
```

This environment variable will have the value of `None` in development as, within that environment, it is not set at all.

Therefore, we can take advantage of that using conditional logic:

```python
# config.py

import os
from dotenv import load_dotenv

TESTING = os.getenv('TESTING')

if TESTING == 'True':
    load_dotenv('.env.testing')
else:
    load_dotenv('.env.development')

PG_USER = os.getenv('PG_USER')
PG_HOST = os.getenv('PG_HOST')
PG_DATABASE = os.getenv('PG_DATABASE')
PG_PORT = os.getenv('PG_PORT')
PG_PASSWORD = os.getenv('PG_PASSWORD')
env = os.getenv('env')
```

## Keeping the Test Database Testable

Re-seeding the database before running the test suite allows the data used for the tests to be more predictable and consistent.

If the same database is used for user interaction with the application (for example using a browser or Insomnia), the user would experience unexpected changes to the data when the test database was re-seeded, or if the tests made changes to the data. This is why it is beneficial to have a separate database for running test scripts that can be regularly re-seeded, and another for manual use of the application where any changes made by a user can persist.

### Fixtures

If you've been clever, and written your `seed` as a reusable function, fixtures provide a way for you to invoke that function for each test.

The seed should ensure that it cleans out any data, before inserting it again, with appropriate `DROP TABLE IF EXISTS` clauses.

> "Software test fixtures initialize test functions. They provide a fixed baseline so that tests execute reliably and produce consistent, repeatable, results. Initialization may setup services, state, or other operating environments. These are accessed by test functions through arguments; for each fixture used by a test function there is typically a parameter (named after the fixture) in the test function’s definition." - [pytest docs](https://docs.pytest.org/en/6.2.x/fixture.html#)

The following is an example of how a fixture can be passed into your test functions in order to seed the database on each test:

```python
from fastapi.testclient import TestClient
from main import app
import db
import pytest


client = TestClient(app)


@pytest.fixture(scope='function')
def seed_db():
    db.seed()


def test_get_things(seed_db):


    expected_response = {
        "things": [
            {"name": "Thing 1", "appearance": "Like Thing 2"},
            {"name": "Thing 2", "appearance": "Like Thing 1"}
        ]
    }

    response = client.get("/api/things").content.decode('utf-8')

    assert response.json() == expected_response
```
