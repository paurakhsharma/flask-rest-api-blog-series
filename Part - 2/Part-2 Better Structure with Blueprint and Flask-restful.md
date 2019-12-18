## Part 2: Better Structure with Blueprint and Flask-restful

Howdy! In the previous [Part](https://dev.to/paurakhsharma/flask-rest-api-part-1-using-mongodb-with-flask-3g7d) of the series, we learned how to use `Mongoengine` to store our movies data into a MongoDB database. Now let's learn how you can structure your flask application in a more maintainable way.

If you are just starting from this part, you can find all the code we wrote till now [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%201).

We are going to learn two ways of structuring the flask application:
- [Blueprint](https://flask.palletsprojects.com/en/1.1.x/blueprints/): It is used to structure the Flask application into different components, making structuring the application based on different functionality.

- [Flask-restful](https://flask-restful.readthedocs.io/en/latest/): It is an extension for Flask that helps your build REST APIs quickly and following best practices.

*Note: `Blueprint` and `Flask-restful` are not a replacement for each other, they can co-exist in a single project*

### Structuring Flask App using Blueprint

Create a new folder `resources` inside `mongo-bag` and a new file `movie.py` inside `resources.`

```bash
mkdir resources
cd resources
touch movie.py
```

Now move all the route related codes from your `app.py` into `movies.py`

```python
#~/movie-bag/resources/movie.py

@app.route('/movies')
def get_movies():
 movies = Movie.objects().to_json()
 return Response(movies, mimetype="application/json", status=200)

@app.route('/movies', methods=['POST'])
def add_movie():
 body = request.get_json()
 movie = Movie(**body).save()
 id = movie.id
 return {'id': str(id)}, 200

@app.route('/movies/<id>', methods=['PUT'])
def update_movie(id):
 body = request.get_json()
 Movie.objects.get(id=id).update(**body)
 return '', 200

@app.route('/movies/<id>', methods=['DELETE'])
def delete_movie(id):
 movie = Movie.objects.get(id=id).delete()
 return '', 200

@app.route('/movies/<id>')
def get_movie(id):
 movies = Movie.objects.get(id=id).to_json()
 return Response(movies, mimetype="application/json", status=200)
```

And you app.py file should look like this

```python
#~/movie-bag/app.py

from flask import Flask, request, Response
from database.db import initialize_db
from database.models import Movie
import json

app = Flask(__name__)

app.config['MONGODB_SETTINGS'] = {
 'host': 'mongodb://localhost/movie-bag'
}

initialize_db(app)

app.run()

```
<br />

Clean right? 
Now you might be wondering how does this work? - This doesn't work. 
We must first create a blueprint in our `movie.py`

Update your `movie.py` like so,

```diff
#~/movie-bag/resources/movie.py

+from flask import Blueprint, Response, request
+from database.models import Movie
+
+movies = Blueprint('movies', __name__)

-@app.route('/movies')
+@movies.route('/movies')
def get_movies():
 movies = Movie.objects().to_json()
 return Response(movies, mimetype="application/json", status=200)

-@app.route('/movies', methods=['POST'])
+@movies.route('/movies', methods=['POST'])
def add_movie():
 body = request.get_json()
 movie = Movie(**body).save()
 id = movie.id
 return {'id': str(id)}, 200

-@app.route('/movies/<id>', methods=['PUT'])
+@movies.route('/movies/<id>', methods=['PUT'])
def update_movie(id):
 body = request.get_json()
 Movie.objects.get(id=id).update(**body)
 return '', 200

-@app.route('/movies/<id>', methods=['DELETE'])
+@movies.route('/movies/<id>', methods=['DELETE'])
def delete_movie(id):
 movie = Movie.objects.get(id=id).delete()
 return '', 200

-@app.route('/movies/<id>')
+@movies.route('/movies/<id>')
def get_movie(id):
 movies = Movie.objects.get(id=id).to_json()
 return Response(movies, mimetype="application/json", status=200)
```

So, we have created a new `Blueprint` using

```diff
+movies = Blueprint('movies', __name__)
```
with arguments `name` and `import_name`. Usually, import_name will just be `__name__`, which is a special Python variable containing the name of the current module.

Now we can replace every instance of `app` inside this `Blueprint` with `movies`.

So, let's update our `app.py` to register the `Blueprint` we created.

```diff
#~/movie-bag/app.py

-from flask import Flask, request, Response
+from flask import Flask
from database.db import initialize_db
-from database.models import Movie
-import json
+from resources.movie import movies

app = Flask(__name__)

app.config['MONGODB_SETTINGS'] = {
 'host': 'mongodb://localhost/movie-bag'
 }

initialize_db(app)
+app.register_blueprint(movies)

app.run()
```

That's it, you have used `Blueprint` to structure your `Flask` application. And it looks way cleaner than it was before.

### Structuring Flask REST API using Flask-restful

Now, let's get to the main topic we have been waiting for from the beginning.

Installing flask-restful

```bash
pipenv install flask-restful
```

Now, let's update our `movie.py` to use `flask-restful`

```diff
#~movie-bag/resources/movie.py

-from flask import Blueprint, Response, request
+from flask import Response, request
 from database.models import Movie
+from flask_restful import Resource
+
-movies = Blueprint('movies', __name__)
+class MoviesApi(Resource):
+ def get(self):
+ movies = Movie.objects().to_json()
+ return Response(movies, mimetype="application/json", status=200)
+
+ def post(self):
+ body = request.get_json()
+ movie = Movie(**body).save()
+ id = movie.id
+ return {'id': str(id)}, 200
+ 
+class MovieApi(Resource):
+ def put(self, id):
+ body = request.get_json()
+ Movie.objects.get(id=id).update(**body)
+ return '', 200
+ 
+ def delete(self, id):
+ movie = Movie.objects.get(id=id).delete()
+ return '', 200
+ def get(self, id):
+ movies = Movie.objects.get(id=id).to_json()
+ return Response(movies, mimetype="application/json", status=200)
+
-@movies.route('/')
-def get_movies():
- movies = Movie.objects().to_json()
- return Response(movies, mimetype="application/json", status=200)
-
-@movies.route('/', methods=['POST'])
-def add_movie():
- body = request.get_json()
- movie = Movie(**body).save()
- id = movie.id
- return {'id': str(id)}, 200
-
-@movies.route('/<id>', methods=['PUT'])
-def update_movie(id):
- body = request.get_json()
- Movie.objects.get(id=id).update(**body)
- return '', 200
-
-@movies.route('/<id>', methods=['DELETE'])
-def delete_movie(id):
- movie = Movie.objects.get(id=id).delete()
- return '', 200
-
-@movies.route('/<id>')
-def get_movie(id):
- movies = Movie.objects.get(id=id).to_json()
- return Response(movies, mimetype="application/json", status=200)

```

As we can see `flask-restful` uses a Class-based syntex so, if we want to define a resource (i.e API) we can just define a class which extends `flask-restful`'s `Resource`
i.e
```diff
+class MoviesApi(Resource):
+ def get(self):
+ movies = Movie.objects().to_json()
+ return Response(movies, mimetype="application/json", status=200)
```
This creates an endpoint which accepts `GET` request.

Now let's register these endpoints that we just created.
Let's create a new file `routes.py` inside `resources` directory and add the following to it.

```python
#~movie-bag/resources/routes.py

from .movie import MoviesApi, MovieApi

def initialize_routes(api):
 api.add_resource(MoviesApi, '/movies')
 api.add_resource(MovieApi, '/movies/<id>')
```

We have defined the function to initialize the routes. Let's call this function from our `app.py`

```diff
#~/movie-bag/app.py

-from resources.movie import movies
+from flask_restful import Api
+from resources.routes import initialize_routes
 
 app = Flask(__name__)
+api = Api(app)
 
 app.config['MONGODB_SETTINGS'] = {
 'host': 'mongodb://localhost/movie-bag'
 }
 
 initialize_db(app)
-app.register_blueprint(movies)
-
+initialize_routes(api)
 
 app.run()

```
Here we first created an `Api` instance with `app = Api(app)` and then initialized the API routes with `initialize_routes(api)`

Wow! we did it y'all. Now we can access our movies at `http://localhost:5000/movies`. As we can see just by looking at the URL we cannot know if this is an API. So, let's update our `routes.py` to add `api/` in front of our API routes.


```diff
#~movie-bag/resources/routes.py
- api.add_resource(MoviesApi, '/movies')
- api.add_resource(MovieApi, '/movies/<id>')
+ api.add_resource(MoviesApi, '/api/movies')
+ api.add_resource(MovieApi, '/api/movies/<id>')
```

Now we can access our movies at `htpp://localhost:5000/api/movies`.

You can find the complete code of this part [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%202)


### What we learned from this part of the series?
- How to structure our Flask App using `Blueprint`
- How to create REST APIs using `Flask-restful`

I guess that's it for this [Flask Rest API - Zero to Yoda](https://dev.to/paurakhsharma/flask-rest-api-part-0-setup-basic-crud-api-4650) series. I hope you learned something of value from this series. I had lots of fun while making this series. I hope you had a mutual feeling.

<hr>

Please let me know if you want to learn how to create a frontend for this REST APIs using modern JavaScript frameworks like [Vue.js](https://vuejs.org/), [Svelte](https://svelte.dev/) and [React.js](https://reactjs.org/)

<hr>

Also, comment down if you want to learn other concepts in Flask like authentication, internalization, sending mail, testing, deploying, etc.
<hr>


Until then happy coding 😊