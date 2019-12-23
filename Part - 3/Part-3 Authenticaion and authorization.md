
Howdy! In the previous [Part](https://dev.to/paurakhsharma/flask-rest-api-part-2-better-structure-with-blueprint-and-flask-restful-2n93) of the series, we learned how to use `Blueprint` and `Flask-Restful` to structure our Flask REST API in a more maintainable way.

Currently, anyone can read, add, delete and update the movies in our application. Now, let's learn how we can restrict the creation of movies by any untrusted person (`Authentication`). Also, we will learn how to implement `Authorization` so that only the person who added the movie in our application can delete/modify it.


To implement these features, first of all, we must create a new document model to store the user information. So, let's do it.

Similar to how we created our `Movie` document model we are going to create a `User` document model. Let's add the following code after the `Movie` document model.

```python
#~/movie-bag/database/models.py

...

class User(db.Document):
 email = db.EmailField(required=True, unique=True)
 password = db.StringField(required=True, min_length=6)
```

Here we created this so that when the user signs up, a new user document is created with fields `email` and `password`. <br />

But saving a password in the plain `StringField` is a terrible idea. If somebody gets access to your database, all user passwords are exposed. To prevent that from happening, we are going to `hash` our password to some `cryptic` form so that nobody can find out the real password easily.

For hashing our password we are going to use a popular hashing function called `bcrypt`. You might have already guessed it, we are going to use a flask extension called [flask-bcrypt](https://flask-bcrypt.readthedocs.io/en/latest/) for this.

Let's install `flask-bcrypt`.

```
pipenv install flask-bcrypt
```

Let' initialize flask-bcrypt in our `app.py`.

```diff
#~/movie-bag/app.py

 from flask import Flask
+from flask_bcrypt import Bcrypt
 from database.db import initialize_db
 from flask_restful import Api
 from resources.routes import initialize_routes
 
 app = Flask(__name__)
 api = Api(app)
+bcrypt = Bcrypt(app)
 
 app.config['MONGODB_SETTINGS'] = {
 'host': 'mongodb://localhost/movie-bag'


```

Now we are going to create two methods: one to create a password hash `generate_password_hash()` and the other to check if the password used by the user to login generates the hash which is equal to the password saved in the database `check_password_hash()`.

Let's update our `models.py` to look like this:

```diff
#~/movie-bag/database/models.py

 from .db import db
+from flask_bcrypt import generate_password_hash, check_password_hash

...

class User(db.Document):
 email = db.EmailField(required=True, unique=True)
 password = db.StringField(required=True, min_length=6)
+ 
+ def hash_password(self):
+   self.password = generate_password_hash(self.password).decode('utf8')
+ 
+ def check_password(self, password):
+   return check_password_hash(self.password, password)
```


Now let's create an API endpoint for `signup`. Add `auth.py` inside `resources` folder with the following code.

```python
#~/movie-bag/resources/auth.py

from flask import request
from database.models import User
from flask_restful import Resource
 
class SignupApi(Resource):
 def post(self):
   body = request.get_json()
   user = User(**body)
   user.hash_password()
   user.save()
   id = user.id
   return {'id': str(id)}, 200
```

This endpoint creates a user document with `email` and `password` received from the `JSON` object sent by the user.

Let's register this endpoint in our `routes.py`.
```diff
 from .movie import MoviesApi, MovieApi
+from .auth import SignupApi
 
 def initialize_routes(api):
   api.add_resource(MoviesApi, '/api/movies')
   api.add_resource(MovieApi, '/api/movies/<id>')
+
+  api.add_resource(SignupApi, '/api/auth/signup')
```
Let's test user signup. Send `JSON` body with `email` and `password` to `http://localhost:5000/api/auth/signup`

![Postman Signup request](https://thepracticaldev.s3.amazonaws.com/i/4qmtg25bjyfrllwimgn8.png)

If we take a look at our database, we can see that our password is hashed to some random password compared to the password we sent in the API request.

![Mongo Compass database entry](https://thepracticaldev.s3.amazonaws.com/i/fa212nj1c9hj7equre4y.png)

*Note: To view the information stored in our database I used [mongo compass](https://www.mongodb.com/download-center/compass)*


Alright, we have created the functionality of creating a user through `signup`, now we need to be able to `login` as that user.

For logging users into a website, we need functionality to verify if the user is who they claim them to be. So, users can send `email` and `password` every time they need to do something on the website, which is not a good idea from a security viewpoint. So, we need functionality such that once the user is logged in into the website they can use their token to access other parts of the website.

There are many methods for working with token-based authentication, In this part, we are going to learn about `JWT` also known as [JSON Web Token](https://jwt.io/introduction/).

To use JWT, let's install another flask extension called [flask-jwt-extended](https://flask-jwt-extended.readthedocs.io/en/stable/basic_usage/) it uses a value we want to save as token (in our case it's `userid`) and combines that with the `salt` (secret key) to create a token.

```
pipenv instll flask-jwt-extended
```

Since the `secret-key` we use to create a JWT needs to be kept somewhere else from your codebase, we are going to use `.env` file to save the secret and give the location of `.env` file to our application using the environment variable.

For that let's create a new file `.env` inside the `movie-bag` folder and add the following to it.

```env
JWT_SECRET_KEY = 't1NP63m4wnBg6nyHYKfmc2TpCOGI4nss'
```

The value of `JWT_SECRET_KEY` can be anything but make that something harder to guess.

Let's update our `app.py` to use configs from `.env` file and initialize `JWT`.

```diff
#~/movie-bag/app.py

from flask import Flask
 from flask_bcrypt import Bcrypt
+from flask_jwt_extended import JWTManager
+
 from database.db import initialize_db
 from flask_restful import Api
 from resources.routes import initialize_routes
 
 app = Flask(__name__)
+app.config.from_envvar('ENV_FILE_LOCATION')
+
 api = Api(app)
 bcrypt = Bcrypt(app)
+jwt = JWTManager(app)
 
 app.config['MONGODB_SETTINGS'] = {
 'host': 'mongodb://localhost/movie-bag'
```

Here `ENV_FILE_LOCATION` is the environment variable which should store the location of `.env` file relative to `app.py`

To set this value mac/linux can run the command:
```
export ENV_FILE_LOCATION=./.env
```
and windows user can run the command:
```
set ENV_FILE_LOCATION=./.env
```

Now, we are finally ready to implement the `login` endpoint. Let's update our `auth.py` inside the `resources` folder:

```diff

-from flask import request
+from flask import Response, request
+from flask_jwt_extended import create_access_token
 from database.models import User
 from flask_restful import Resource 
+import datetime
+
 class SignupApi(Resource):
 def post(self):
 body = request.get_json()
@@ -9,4 +11,16 @@ class SignupApi(Resource):
 user.hash_password()
 user.save()
 id = user.id
 return {'id': str(id)}, 200
+
+class LoginApi(Resource):
+ def post(self):
+   body = request.get_json()
+   user = User.objects.get(email=body.get('email'))
+   authorized = user.check_password(body.get('password'))
+   if not authorized:
+     return {'error': 'Email or password invalid'}, 401
+ 
+   expires = datetime.timedelta(days=7)
+   access_token = create_access_token(identity=str(user.id), expires_delta=expires)
+   return {'token': access_token}, 200

```

Here we search for the user with the given email and check if the password sent is the same as the hashed password saved in the database.
If the password and email are correct we then create access token using `create_access_token()` which uses `user.id` as the identifier and the token expires in `7 days.` which means a user cannot access the website using this token after 7 days.

Let's register this API endpoint in our `routes.py`
```diff
 from .movie import MoviesApi, MovieApi
-from .auth import SignupApi
+from .auth import SignupApi, LoginApi
 
 def initialize_routes(api):
 api.add_resource(MoviesApi, '/api/movies')
 api.add_resource(MovieApi, '/api/movies/<id>')
 
 api.add_resource(SignupApi, '/api/auth/signup')
+ api.add_resource(LoginApi, '/api/auth/login')

```

Now, we need to restrict an unauthorized user from adding, editing and deleting the movies in our application. To do that, let's add `@jwt_required` decorator to our endpoints. This protects our endpoints form invalid or expired jwt.

Update `movie.py` as:

```diff
#~/movie-bag/resources/movie.py

 from flask import Response, request
+from flask_jwt_extended import jwt_required
 from database.models import Movie
 from flask_restful import Resource
 
class MoviesApi(Resource):
 ...
+ 
+ @jwt_required
 def post(self):
   body = request.get_json()
   movie = Movie(**body).save()
 ...


 class MovieApi(Resource):
+ @jwt_required
 def put(self, id):
   body = request.get_json()
   Movie.objects.get(id=id).update(**body)
   return '', 200
+ 
+ @jwt_required
 def delete(self, id):
   movie = Movie.objects.get(id=id).delete()
   return '', 200
```

Let's test this now.
First of all, we have to login as the user we created earlier with `signup`.

![Postman login request](https://thepracticaldev.s3.amazonaws.com/i/tggxm49c1glytejkkv35.png)

We got the token back from the server, now let's try to create a movie from the API endpoint `http://localhost:5000/api/movies`. As you can see you cannot do it and get an error, because it is protected by `jwt`. <br />
*Note: We will learn how to make error message friendly later in this series.*

Now let's use the token we got earlier from `login` in our `Authorization` header.

To use authorization header in Postman follow the steps:
1) Go to the `Authorization` tab.
2) Select the `Bearer Token` form `TYPE` dropdown.
3) Paste the token you got earlier from `/login`
4) Finally, send the request.

![Postman request with authorization header](https://thepracticaldev.s3.amazonaws.com/i/zxmx2filtj5vh8chh187.png)

<hr>

Let's add a feature such that only the user who created the movie can delete or edit the movie.

Let's update our `models.py` and create a relation between the user and the movie.

```diff
#~/movie-bag/database/models.py

class Movie(db.Document):
 name = db.StringField(required=True, unique=True)
 casts = db.ListField(db.StringField(), required=True)
 genres = db.ListField(db.StringField(), required=True)
+ added_by = db.ReferenceField('User')
 
 class User(db.Document):
   email = db.EmailField(required=True, unique=True)
   password = db.StringField(required=True, min_length=6) 
+  movies = db.ListField(db.ReferenceField('Movie', reverse_delete_rule=db.PULL))
 


+
+User.register_delete_rule(Movie, 'added_by', db.CASCADE)
```

We have created a one-many relationship between `user` and `movie`. That means a user can have one or more movies and a movie can only be created by one user. Here `reverse_delete_rule` in the movies field of `User` represents that a movie should be pulled from the user document if the movie is deleted.
Similarly, `User.register_delete_rule(Movie, 'added_by', db.CASCADE)` creates another delete rule which means if a user is deleted then the movie created by the user is also deleted. <br />
*Note: I had to register delete rule for `added_by` separately because `User` is not yet defined while defining `Movie`*


Now, let's update `movie.py` to apply the authorization.

```diff

 from flask import Response, request
-from flask_jwt_extended import jwt_required
-from database.models import Movie
+from database.models import Movie, User
+from flask_jwt_extended import jwt_required, get_jwt_identity
 from flask_restful import Resource
 
 class MoviesApi(Resource):
   def get(self):
     movies = Movie.objects().to_json()
     return Response(movies, mimetype="application/json", status=200)
 
   @jwt_required
   def post(self):
+    user_id = get_jwt_identity()
     body = request.get_json()
-    movie = Movie(**body).save()
+    user = User.objects.get(id=user_id)
+    movie = Movie(**body, added_by=user)
+    movie.save()
+    user.update(push__movies=movie)
+    user.save()
     id = movie.id
     return {'id': str(id)}, 200
 
 class MovieApi(Resource):
   @jwt_required
   def put(self, id):
+    user_id = get_jwt_identity()
+    movie = Movie.objects.get(id=id, added_by=user_id)
     body = request.get_json()
     Movie.objects.get(id=id).update(**body)
     return '', 200
 
   @jwt_required
   def delete(self, id):
-    movie = Movie.objects.get(id=id).delete()
+    user_id = get_jwt_identity()
+    movie = Movie.objects.get(id=id, added_by=user_id)
+    movie.delete()
     return '', 200
 
 def get(self, id):
...
```

Here `get_jwt_identity()` method returns the value encoded by `create_access_token()` which in our case is `user.id`. So, we only delete/update the movie which is added_by the user who is sending the request to the application.

<hr>

You can find the complete code of this part [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%203)

### What we learned from this part of the series?
- How to hash user password using `flask-bcrypt`
- How to create JSON token using `flask-jwt-extended`
- How to protect API endpoints from unauthorized access.
- How to implement authorization so that only the user who added the movie can delete/update the movie.

<hr>

Since there are a lot of unfriendly errors and exceptions in our application, in the next part we are going to learn how to handle errors and exceptions in our REST API. 

<hr>
Please let me know if you are stuck at any point so that I can guide you. Also, if there is something you want me to cover in the next parts/series don't forget to mention that below.
<hr>


Until then happy coding 😊