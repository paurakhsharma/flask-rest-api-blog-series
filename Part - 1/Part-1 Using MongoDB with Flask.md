## Part 1: Using MongoDB with Flask

Howdy! In the last [Part](https://dev.to/paurakhsharma/flask-rest-api-part-0-setup-basic-crud-api-4650) of the series, we learned how to create a basic `CRUD` REST API functionality using python `list`. But that's not how the real-world applications are built, because if your server is restarted or god forbids crashes then you are gonna lose all the information stored in your server. To solve those problems (and many others) database is used. So, that's what we are gonna do. We are going to use [MongoDB](https://docs.mongodb.com/manual/) as our database.

If you are just starting from this part, you can find all the code we wrote till now [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%200).

Before we start make sure you have installed MongoDB in your system. If you haven't already you can install for [Linux](https://docs.mongodb.com/manual/administration/install-on-linux/), [Windown](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-windows/) and [macOS](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-os-x/).

There are mainly to popular libraries which makes working with MongoDB easier:

1) [Pymongo](https://api.mongodb.com/python/current/) is a low-level Python wrapper around MongoDB, working with `Pymongo` is similar to writing the MongoDB query directly.
Here is the simple example of updating the name of a movie whose `id` matches the given `id` using `Pymongo`.
```python
db['movies'].update({'_id': id},
                    {'$set': {'name': 'My new title'}})
```
`Pymongo` doesn't use any predefined schema so it can make full use of Schemaless nature of MongoDB.

2) [MongoEngine](http://docs.mongoengine.org/) is an Object-Document Mapper, which uses a document schema that makes working with MongoDB clear and easier.
Here is the same example using `mongoengine`.
```python
Movies.objects(id=id).update(name='My new title')
```
`Mongoengine` uses predefined schema for the fields in the database which restricts it from using the Schemaless nature of MongoDB.

As we can see both sides have their advantages and disadvantages. So, choose the one that fits your project well. In this series we are going to learn about `Mongoengine`, please do let me know in the comment section below if you want me to cover `Pymongo` as well.

To work better with `Mongoengine` in our `Flask` application there is a great `Flask` extension called [Flask-Mongengine](http://docs.mongoengine.org/projects/flask-mongoengine/en/latest/).

So, let's get started by installing `flask-mongoengine`.
```
pipenv install flask-mongoengine
```
*Note: Since `flask-mongoengine` is built on top of `mongoengine` it gets installed automatically while installing flask-mongoengine, also `mongoengine` is build on top of `pymongo` so, it also gets installed*


Now, let's create a new folder inside `movie-bag`. I am gonna call it `database`. Inside `database` folder create a file named `db.py`. Also, create another file and name it `models.py`

Let's see how files/folder looks like now.

```bash
movie-bag
│   app.py
|   Pipfile
|   Pipfile.lock   
└───database
    │   db.py
    └───models.py
```

Now, let's dive into the interesting part.
First of all, let's initialize our database by adding the following code to our `db.py`

```python
#~movie-bag/database/db.py

from flask_mongoengine import MongoEngine

db = MongoEngine()

def initialize_db(app):
    db.init_app(app)
```
Here we have imported `MongoEngine` and created the `db` object and we have defined a function `initialize_db()` which we are gonna call from our `app.py` to initialize the database.


Let's write the following code in our `movie.py` inside `models` directory

```python
#~movie-bag/database/models.py
from .db import db

class Movie(db.Document):
    name = db.StringField(required=True, unique=True)
    casts = db.ListField(db.StringField(), required=True)
    genres = db.ListField(db.StringField(), required=True)
```
What we just created is a document for our database. So, that the users cannot add other fields then what are defined here.
Here we can see the `Movie` document has three fields:
1) `name`: is a field of type `String`, we also have two constraints in this field.
    - `required` which means the user cannot create a new movie without giving its title. 
    - `unique` which means the movie name must be unique and cannot be repeated.

2) `casts`: is a field of type `list` which contains the values of type `String`

3) `genres`: same as `casts`

Finally, we can initialize the database in our `app.py` and change our `view` functions (functions handling our API request) to use the `Movie` document we defined earlier.

```diff
#~movie-bag/app.py

-from flask import Flask, jsonify, request
+from flask import Flask, request, Response
+from database.db import initialize_db
+from database.models import Movie
 
 app = Flask(__name__)
 
-movies = [
-    {
-        "name": "The Shawshank Redemption",
-        "casts": ["Tim Robbins", "Morgan Freeman", "Bob Gunton", "William Sadler"],
-        "genres": ["Drama"]
-    },
-    {
-       "name": "The Godfather ",
-       "casts": ["Marlon Brando", "Al Pacino", "James Caan", "Diane Keaton"],
-       "genres": ["Crime", "Drama"]
-    }
-]
+app.config['MONGODB_SETTINGS'] = {
+    'host': 'mongodb://localhost/movie-bag'
+}
+
+initialize_db(app)
 
-@app.route('/movies')
-def hello():
-    return jsonify(movies)

+@app.route('/movies')
+def get_movies():
+    movies = Movie.objects().to_json()
+    return Response(movies, mimetype="application/json", status=200)

-@app.route('/movies', methods=['POST'])
-def add_movie():
-    movie = request.get_json()
-    movies.append(movie)
-    return {'id': len(movies)}, 200

+@app.route('/movies', methods=['POST'])
+    body = request.get_json()
+    movie = Movie(**body).save()
+    id = movie.id
+    return {'id': str(id)}, 200

-@app.route('/movies/<int:index>', methods=['PUT'])
-def update_movie(index):
-    movie = request.get_json()
-    movies[index] = movie
-    return jsonify(movies[index]), 200

+@app.route('/movies/<id>', methods=['PUT'])
+def update_movie(id):
+    body = request.get_json()
+    Movie.objects.get(id=id).update(**body)
+    return '', 200

-@app.route('/movies/<int:index>', methods=['DELETE'])
-def delete_movie(index):
-    movies.pop(index)
-    return 'None', 200

+@app.route('/movies/<id>', methods=['DELETE'])
+def delete_movie(id):
+    Movie.objects.get(id=id).delete()
+    return '', 200
 
 app.run()
```

Wow! that a lot of changes, let's go step by step with the changes.

```diff
-from flask import Flask, jsonify, request
+from flask import Flask, request, Response
+from database.db import initialize_db
+from database.models.movie import Movie
```

Here we removed `jsonify` as we no longer need and added `Response` which we use to set the type of response. Then we import `initialize_db` form `db.py` which we defined earlier to initialize our database. And lastly, we imported the `Movie` document form `movie.py`

```diff
+app.config['MONGODB_SETTINGS'] = {
+    'host': 'mongodb://localhost/movie-bag'
+}
+
+db = initialize_db(app)
```

Here we set the configuration for our mongodb database. Here the host is in the format `<host-url>/<database-name>`. Since we have installed mongodb locally so we can access it from `mongodb://localhost/` and we are gonna name our database `movie-bag`.
And at the last, we initialize our database.

```diff
+@app.route('/movies')
+def get_movies():
+    movies = Movie.objects().to_json()
+    return Response(movies, mimetype="application/json", status=200)
+
```
Here we get all the objects from `Movie` document using `Movies.objects()` and convert them to `JSON` using `to_json()`. At last, we return a `Response` object, where we defined our response type to `application/json`.


```diff
+@app.route('/movies', methods=['POST'])
+    body = request.get_json()
+    movie = Movie(**body).save()
+    id = movie.id
+    return {'id': str(id)}, 200
```

In the `POST` request we first get the `JSON` that we send and a request. And then we load the `Movie` document with the fields from our request with `Movie(**body)`. Here `**` is called the spread operator which is written as `...` in JavaScript (if you are familiar with it.). What it does is like the name suggests, spreads the `dict` object. <br />
So, that `Movie(**body)` becomes 
```python
Movie(name="Name of the movie",
    casts=["a caste"],
    genres=["a genre"])
```
At last, we save the document and get its `id` which we return as a response.

```diff
+@app.route('/movies/<id>', methods=['PUT'])
+def update_movie(id):
+    body = request.get_json()
+    Movie.objects.get(id=id).update(**body)
+    return '', 200
```

Here we first find the Movie document matching the `id` sent in the request and then update it. Here also we have applied the spread operator to pass the values to the `update()` function.


```diff
+@app.route('/movies/<id>', methods=['DELETE'])
+def delete_movie(id):
+    Movie.objects.get(id=id).delete()
+    return '', 200
```
Similar to the `update_movie()` here we get the Movie document matching given `id` and delete it from the database.

Oh, **I just remembered** that we haven't added the API endpoint to `GET` only one document from our server. 
Let's add it:
Add the following code right above `app.run()`

```python
@app.route('/movies/<id>')
def get_movie(id):
    movies = Movie.objects.get(id=id).to_json()
    return Response(movies, mimetype="application/json", status=200)
```
Now you can get the single movie from API endpoint `/movies/<valid_id>`.

To run the server make sure you are at `movie-bag` directory.

Then run 
```bash
pipenv shell
python app.py
```

To activate the virtual environment in your terminal and start the server.

Wow! Congratulations on making this far. To test the APIs, use `Postman` as we used in the [previous]((https://dev.to/paurakhsharma/flask-rest-api-part-0-setup-basic-crud-api-4650)) part of this series.

You might have noticed that if we send invalid data to our endpoint e.g: without a name, or other fields we get an unfriendly error in the form of `HTML`. If we try to get the movie document with `id` that doesn't exist in the database then also we get an unfriendly error in the form of `HTML` response. Which is not an excepted behavior of a nicely build API. We will learn how we can handle such errors in the later parts of the series.

### What we learned from this part of the series?
 - Difference between `Pymongo` and `Mongoengine`.
 - How to create Document schema using `Mongoengine`.
 - How to perform `CRUD` operation using `Mongoengine`.
 - Python spread operator.

You can find the complete code of this part [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%201)

In the next part, we are going to learn how to better structure your flask application using `Blueprint`. And also how to create REST APIs faster, following best practices with the minimal setup using `flask-restful`

Until then happy coding 😊