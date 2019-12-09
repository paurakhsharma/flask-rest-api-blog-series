## Part 0: Setup & Basic CRUD API

Howdy! Welcome to the Flask Rest API - Zero to Yoda, tutorial series. We will go through building a Movie database where a user can (Add, Edit, Update & delete) `Genre`, `Movie`, and `Casts`. First of all, we will start with a basic APIs structure with just `Flask` and then learn about integrating `MongoDB` with the application, finally, we will learn about structuring our API following the best practices with the minimal setup using `Flask-RESTful`.

What we are going to learn in this series?
 - [Flask](https://palletsprojects.com/p/flask/) - For our web server.
 - [flask-restful](https://flask-restful.readthedocs.io/en/latest/installation.html) - For building cool Rest-APIs.
 - [Pipenv](https://pipenv.readthedocs.io/en/latest/) For managing python virtual environments.
 - [mongoengine](http://docs.mongoengine.org/projects/flask-mongoengine/en/latest/) - For managing our database.
 - [flask-marshmallow](https://flask-marshmallow.readthedocs.io/en/latest/) - For serializing user requests.
 - [Postman](https://www.getpostman.com/downloads/) - For testing our APIs


Why flask?
 - Easy to get started 😊
 - Great for building Rest APIs and microservices.
 - Used by big companies like Netflix, Reddit, Lyft, Et cetera.
 - Great for building APIS for machine learning applications.
 - Force is strong with this one 😉

Who is this series for?
 - Beginners with basic Python knowledge who wants to build cool apps.
 - Experienced flask developers who have been working with flask `SSR` (Server Side Rendering).

### So, let's get started.
First of all, create a new directory and browse to the newly created directory, I'm gonna call it `movie-bag`.

```
mkdir movie-bag
cd movie-bag
```

First of all install `pipenv`
using command

```
pip install --user pipenv
```

`Pipenv` is used to create a `virtual environment` which isolates the python packages you used in this project from other system python packages. So, that you can have a different version of same python packages for different projects.

Now, let's install `flask` using `pipenv`
```
pipenv install flask
```
This will create a new virtual environment and install flask. This command will create two files `Pipfile` and `Pipfile.lock`.
```
#~ movie-bag/Pipfile

[[source]]
name = "pypi"
url = "https://pypi.org/simple"
verify_ssl = true

[dev-packages]

[packages]
flask = "*"

[requires]
python_version = "3.7"
```

Pipfile contains the packages that are required for your application. As you can see `flask` is added to `[packages]` list. This means when someone downloads your code and runs `pipenv install`, `flask` gets installed in their system. Great, right?

If you are familiar with `requirements.txt`, think `Pipfile` as the `requirements.txt` on steroids.


Flask is so simple that you can create an API using a single file. (But you don't have to 😅)

Create a new file called `app.py` where we are gonna write our Flask Hello World API. Write the following code in `app.py`

```python
#~movie-bag/app.py

from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return {'hello': 'world'}


app.run()
```

Let's understand what we just wrote. First of all, we imported the `Flask` class from `flask` package.

Then we define a root endpoint with `@app.route('/')`,  `@app.route()` is called a `decorator` which basically takes function `hello()` extends it's behavior so that it is invoked when `/` endpoint is requested. And `hello()` returns `{'hello': 'world'}`, finally `Flask` server is started with `app.run()`

Wanna learn more about decorators? Read this great article {% link https://dev.to/sonnk/python-decorator-and-flask-4c16 %}

There you go, you have made yourself your very first Flask API (Pat on your back).

To run the app, first, enable the virtual environment that you created earlier while installing `Flask` with

```
pipenv shell
```
Now run the app using,
```
python app.py
```

The output looks like this:
```
* Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

As you can see the app is running on `http://127.0.0.1:5000/`. Type the URL in your browser of choice and see the `JSON` response from the server.

*Note: Alias for 127.0.0.1 is localhost, So you can access your API at http:localhost:5000 aswell*

Now let's update our `app.py` to have more fun 😉

```diff
#~/movie-bag/app.py

-from flask import Flask
+from flask import Flask, jsonify
 
 app = Flask(__name__)
 
+movies = [
+    {
+        "name": "The Shawshank Redemption",
+        "casts": ["Tim Robbins", "Morgan Freeman", "Bob Gunton", "William Sadler"],
+        "genres": ["Drama"]
+    },
+    {
+       "name": "The Godfather ",
+       "casts": ["Marlon Brando", "Al Pacino", "James Caan", "Diane Keaton"],
+       "genres": ["Crime", "Drama"]
+    }
+]
+
-@app.route('/')
+@app.route('/movies')
def hello():
-    return {'hello': 'world'}
+    return jsonify(movies)
 
 
 app.run()
```

*Note:*  *In the code snippet above, `-`(`red`) represents the part of the previous code that was removed and `+`(`green`) represents the code that replaced it. So, if you are coding along with me. Only copy the `green` codes excluding `+` sign.*

Here we imported `jsonify` from `flask` which is used to convert our `movies` list into proper `JSON` value.
Notice we also renamed our API endpoint from `/` to `/movies`.
So, now our API should be accessable at `http://localhost:5000/movies`

To see the changes, restart your `Flask` server.


To work with our APIs, we are going to use `Postman`. Postman is used for testing the APIs with different HTTP methods. Such as sending data to our web server with `POST` request, updating the data in our server with `PUT` request, getting the data from the server with `GET` and deleting them with `DELETE` request.

Learn more about [REST API HTTP methods here.](https://restfulapi.net/http-methods/)

Install [Postman](https://www.getpostman.com/downloads/) to test your API endpoint easily.

After installing `Postman`, open a new tab and send the `GET` request to your server (http://localhost:5000).

![Postman get request response](https://thepracticaldev.s3.amazonaws.com/i/fzj8uyk1fec894r28xu0.png)
*GET request response using Postman*


Ok now we are ready to add new API endpoints for `CRUD` (Create, Update, Retrieve & Delete)

Add the following code in `app.py` before `app.run()`

```diff
#~movie-bag/app.py

-from flask import Flask, jsonify
+from flask import Flask, jsonify, request
 
 app = Flask(__name__)
 
@@ -19,5 +19,21 @@ movies = [
 def hello():
     return jsonify(movies)
 
+@app.route('/movies', methods=['POST'])
+def add_movie():
+    movie = request.get_json()
+    movies.append(movie)
+    return {'id': len(movies)}, 200
+
+@app.route('/movies/<int:index>', methods=['PUT'])
+def update_movie(index):
+    movie = request.get_json()
+    movies[index] = movie
+    return jsonify(movies[index]), 200
+
+@app.route('/movies/<int:index>', methods=['DELETE'])
+def delete_movie(index):
+    movies.pop(index)
+    return 'None', 200
 
 app.run()
```

As you can see `@app.route()` can take one more argument in addition to the API endpoint. Which is `methods` for API endpoint.

What we have just added are: 
1) `@app.route('/movies', methods=['POST'])`
endpoint for adding a new movie to our `movies` list. This endpoint accepts the `POST` request. With `POST` request you can send a new movie for the list. </br>
*Use postman to send a movie via POST request*
  - Select `POST` from the dropdown
  - Click on the `Body` tab and then click `raw`.
  - Select `JSON` from the drop down (indicating we are sending the data to our server in JSON format.)
  - Enter the following movie details and hit `SEND`
```json
  {
    "name": "The Dark Knight",
    "casts": ["Christian Bale", "Heath Ledger", "Aaron Eckhart", "Michael Caine"],
    "genres": ["Action", "Crime", "Drama"]
  }
```
![Postman POST request response](https://thepracticaldev.s3.amazonaws.com/i/6zll28s9eli18nhc91j8.png)
*POST request response using Postman*
<br />
In the response you get the id of the recently added movie
```json
{"id": 2}
```
- Now again send `GET` request and see a list of `3` movies is responsed by the server 😊

2) `@app.route('/movies/<int:index>', methods=['PUT'])` endpoint for editing the movie which is already existing on the list based on it's `index` as suggested by `<int:index>`. Similarly for `PUT` request also you have to include the `JSON` body for the movie you want to update. <br />
  
![Postman PUT request response](https://thepracticaldev.s3.amazonaws.com/i/4abhnnqiqszksbg88m8q.png)

*PUT request response using Postman*
- Now send a `GET` request to see the movie you updated actually getting updated on the list of movies.
  
3) `@app.route('/movies/<int:index>', methods=['DELETE'])` API endpoint for deleting the movie from the given index of movies list.
![Postman DELETE request response](https://thepracticaldev.s3.amazonaws.com/i/bpbmmfh3p12vppax6s81.png)
*DELETE request response using Postman*
- Now send a `GET` request to `/movies` and see that the movie is already removed from the `movies` list.

### What we learned from this part of the series?
 - Install `Flask` in a new virtual environment using `pipenv`
 - Create a simple hello world flask application
 - Create API endpoints with `CRUD` functionality.


That's it for this part of the series y'all.
You can find the code for this part [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%200).

In this part, we only learned how to store movies in a python list but in the next part of the series, we are going to learn how we can use `Mongoengine` to store our movies in the real `MongoDB` database.

Until then happy coding 😊