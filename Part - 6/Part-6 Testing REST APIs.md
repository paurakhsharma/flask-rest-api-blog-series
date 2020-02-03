## Part 6: Testing REST APIs

Howdy! In the previous [Part](https://dev.to/paurakhsharma/flask-rest-api-part-5-password-reset-2f2e) of the series, we learned how to perform
password reset in our REST API.

In this part, we are going to learn how to test our REST API endpoints.

### Why should we spend time writing tests?
- To make sure our application doesn't break while making changes/refactoring
- To automate repetitive manual tests reducing human errors
- Be able to release to the production on Fridays ;)
- Testing provides a better CI/ CD workflow.

I hope you are convinced that we should write tests. Let's get started with testing our Flask application.

When it comes to testing, there are two most popular tools to test Python applications.
1) [unittest](https://docs.python.org/3/library/unittest.html): `unittest` is a python [standard library](https://docs.python.org/3/library/) which means it is distributed with Python. `unittest` provides tons of tools for constructing and running tests.

2) [pytest](https://docs.pytest.org/en/latest/): `pytest` is a python library which I like to call is the superset of `unittest` which means you can run tests written in `unittest` with `pytest`. It makes writing tests easier and faster.

In this tutorial, we are going to learn how to write tests using `unittest`, because it enables us to write our tests using [OOP](https://docs.python.org/3/tutorial/classes.html).

Before we start, remove the line below from `app.py`

```py
app.config['MONGODB_SETTINGS'] = {
    'host': 'mongodb://localhost/movie-bag'
}
```
and add
```bash
MONGODB_SETTINGS = {
    'host': 'mongodb://localhost/movie-bag'
}
```
to our `.env`, this step is required because we want to use a different database for developing our application and running the tests.

First of all, let's create an `env` file to store our test-related configurations, we should separate our test configs from our development and production configs.

In the root directory create a file `.env.test` and add the following configs to it.

```bash
touch .env.test
```

```bash
#~/movie-bag/.env.test

JWT_SECRET_KEY = 'super-secret'
MAIL_SERVER: "localhost"
MAIL_PORT = "1025"
MAIL_USERNAME = "support@movie-bag.com"
MAIL_PASSWORD = ""
MONGODB_SETTINGS = {
    'host': 'mongodb://localhost/movie-bag-test'
}
```
>*Notice that we have used different database for our test config, this is done because our tests and we want our tests and development database to be separated. We also want our test database to be empty before running the tests.*

Now, let's create a new folder `tests` inside our root directory. Create a new file `__init__.py` inside the `tests` folder, also, create a new file `test_signup.py`.

```bash
mkdir tests
cd tests
touch __init__.py
touch test_signup.py
```

```python
#~/movie-bag/tests/test_signup.py

import unittest
import json

from app import app
from database.db import db


class SignupTest(unittest.TestCase):

    def setUp(self):
        self.app = app.test_client()
        self.db = db.get_db()

    def test_successful_signup(self):
        # Given
        payload = json.dumps({
            "email": "paurakh011@gmail.com",
            "password": "mycoolpassword"
        })

        # When
        response = self.app.post('/api/auth/signup', headers={"Content-Type": "application/json"}, data=payload)

        # Then
        self.assertEqual(str, type(response.json['id']))
        self.assertEqual(200, response.status_code)

    def tearDown(self):
        # Delete Database collections after the test is complete
        for collection in self.db.list_collection_names():
            self.db.drop_collection(collection)
```


Let's go step by step to understand what is actually going on.

First of all, we define `SignupTest` class which extends `unittest.TestCase`. `TestCase` provides us with useful methods such as `setUp` and `tearDown` and also the assertation methods.

`setUp()` method runs each time before running each method defined on the `SignupTest` class. `setUp()` as the name suggests is used to set up our test infrastructure before running the tests.<br>
Here you can see we define `this.app` and `this.db` in this method. We use `app.test_client()` instead of `app` because it makes testing our flask application easier. Also, we get our `Database` instance with `db.get_db()` and set it to `this.db`.


Similarly, `test_successful_signup()` is the method that is actually testing the `Signup` feature. Here we have defined a payload which should be a `JSON` value. And we send a `POST` request to `/api/auth/signup`.

The response from the request is used to finally assert that our `Signup` feature actually sent the user id and successful status code which is `200`.

Finally, after each test methods the `tearDown()` method runs each time. This method is responsible for clearing our infrastructure setup. This includes deleting our database collection for `test isolation`.

### Test Isolation
Test isolation is one of the most important concepts in testing. Usually, when we are writing tests, we test one business logic. The idea of test isolation is that one of your tests should not in any way affect another test.<br>
Suppose that you have created a user in one test and you are testing login on another test. To follow test isolation you cannot depend on the user-created in a user creation test, but should create the user right in the test where you are going to test login. Why? Because your login test might run before your user creation test this makes your test fail.

Also, if we do not delete our user which we created on the previous test run, and we try to run the test again, our test fails because the user is already there.
So, we should always test a feature from an empty state and for that easiest way is to delete all the collections in our database.


Before running our first test make sure to export environment variable `ENV_FILE_LOCATION` with the location to the test env file.

To set this value mac/linux can run the command:
```
export ENV_FILE_LOCATION=./.env.test
```

and windows user can run the command:
```
set ENV_FILE_LOCATION=./.env.test
```

Make sure you have activated your virtual environment with `pipenv shell`.

To run the test enter this command in your terminal.
```
python -m unittest tests/test_signup.py
```

You should be able to see the output like this:
```
.
----------------------------------------------------------------------
Ran 1 test in 1.023s

OK
```
This means our test run successfully.

**If you run into any error feel free to comment down, I am always ready to help you out**

As you can see we are going to need this `setUp()` and `tearDown()` in our ever TestCase. So, let's move this logic to a new file, let's call it `BaseCase.py`.

```python
#~/movie-bag/tests/BaseCase.py

import unittest

from app import app
from database.db import db


class BaseCase(unittest.TestCase):

    def setUp(self):
        self.app = app.test_client()
        self.db = db.get_db()


    def tearDown(self):
        # Delete Database collections after the test is complete
        for collection in self.db.list_collection_names():
            self.db.drop_collection(collection)
```

Now update your `test_signup.py` to look like this:

```diff

 import json

-from app import app
-from database.db import db
+from tests.BaseCase import BaseCase

-
class SignupTest(unittest.TestCase):
-
-    def setUp(self):
-        self.app = app.test_client()
-        self.db = db.get_db()

     def test_successful_signup(self):
         # Given
...
-
-    def tearDown(self):
-        # Delete Database collections after the test is complete
-        for collection in self.db.list_collection_names():
-            self.db.drop_collection(collection)
```

Now let's add test for our `Login` feature, create a new file `test_login.py` inside `tests` folder with the following code.

```python
#~/movie-bag/tests/test_login.py

import json

from tests.BaseCase import BaseCase

class TestUserLogin(BaseCase):

    def test_successful_login(self):
        # Given
        email = "paurakh011@gmail.com"
        password = "mycoolpassword"
        payload = json.dumps({
            "email": email,
            "password": password
        })
        response = self.app.post('/api/auth/signup', headers={"Content-Type": "application/json"}, data=payload)

        # When
        response = self.app.post('/api/auth/login', headers={"Content-Type": "application/json"}, data=payload)

        # Then
        self.assertEqual(str, type(response.json['token']))
        self.assertEqual(200, response.status_code)
```


Here we first created the user with `/api/auth/signup` endpoint and login using the same email and password and assert that the `/api/auth/login` endpoint returns the token.

Now, let's add tests to check the creation of the movie.
Create `test_create_movie.py` with the code below.

```python
#movie-bag/tests/test_create_movie.py

import json

from tests.BaseCase import BasicTest

class TestUserLogin(BasicTest):

    def test_successful_login(self):
        # Given
        email = "paurakh011@gmail.com"
        password = "mycoolpassword"
        user_payload = json.dumps({
            "email": email,
            "password": password
        })

        self.app.post('/api/auth/signup', headers={"Content-Type": "application/json"}, data=user_payload)
        response = self.app.post('/api/auth/login', headers={"Content-Type": "application/json"}, data=user_payload)
        login_token = response.json['token']

        movie_payload = {
            "name": "Star Wars: The Rise of Skywalker",
            "casts": ["Daisy Ridley", "Adam Driver"],
            "genres": ["Fantasy", "Sci-fi"]
        }
        # When
        response = self.app.post('/api/movies',
            headers={"Content-Type": "application/json", "Authorization": f"Bearer {login_token}"},
            data=json.dumps(movie_payload))

        # Then
        self.assertEqual(str, type(response.json['id']))
        self.assertEqual(200, response.status_code)
```

To run all the tests at once use the command:

```
python -m unittest --buffer
```

Here `--buffer` or `-b` is used to discard the output on a successful test run.


Here we first signup for the user account, log in as the user to get the login token and then use the login token to create a movie. Finally, we check to see if the movie creating endpoint returns the `id` to the created movie.

You might have noticed in this test we only check if the movie creation works but do not check if the user creation worked or user login worked. This is because we already have separate tests that are testing these things so, we don't have to repeat the same tests.


>*We have only created happy path tests but it is crucial for us to test that our application response is expected even in the case when the user enters invalid input. For instance, the user doesn't send the password while signing up or sends an invalid format email.*

**I have not included these tests in the tutorial itself, but I will be sure to include them in the Github repo.**

You can find all the code we have written till now and **more tests** [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%206)

### What we learned from this part of the series?
- Why we should write tests for our application
- What test isolation is and why we should isolate our tests cases
- How to test Flask REST APIs with `unittest`


Until then, Happy Coding 😊
