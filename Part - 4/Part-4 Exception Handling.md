## Part 4: Exception Handling

Howdy! In the previous [Part](https://dev.to/paurakhsharma/flask-rest-api-part-3-authentication-and-authorization-5935) of the series, we learned how we can
add `authentication` and `authorization`. In this part, we are going to learn how we can add make
our flask application more resilient to errors, and how to send a proper error message to the client.

When something goes wrong in a computer program, the program throws a specific `Exception` giving some hint to the user what went wrong. In our application when something goes wrong e.g user tries to create another account with the already used email address, they get an `Internal Server Error` and the client has no idea what they did wrong. So, to solve such issues we are going to use `Exception Handling` to catch such exceptions and send a proper error message to the client indicating what went wrong.

We are going to use a really useful feature of `flask-restful` which lets us define [Custom Error Messages](https://flask-restful.readthedocs.io/en/0.3.5/extending.html#custom-error-handlers). 

Let's create a new file `errors.py` inside the `resources` folder and add the following code:

```bash
cd resources
touch errors.py
```

```python
#~/movie-bag/resources/errors.py

class InternalServerError(Exception):
    pass

class SchemaValidationError(Exception):
    pass

class MovieAlreadyExistsError(Exception):
    pass

class UpdatingMovieError(Exception):
    pass

class DeletingMovieError(Exception):
    pass

class MovieNotExistsError(Exception):
    pass

class EmailAlreadyExistsError(Exception):
    pass

class UnauthorizedError(Exception):
    pass

errors = {
    "InternalServerError": {
        "message": "Something went wrong",
        "status": 500
    },
     "SchemaValidationError": {
         "message": "Request is missing required fields",
         "status": 400
     },
     "MovieAlreadyExistsError": {
         "message": "Movie with given name already exists",
         "status": 400
     },
     "UpdatingMovieError": {
         "message": "Updating movie added by other is forbidden",
         "status": 403
     },
     "DeletingMovieError": {
         "message": "Deleting movie added by other is forbidden",
         "status": 403
     },
     "MovieNotExistsError": {
         "message": "Movie with given id doesn't exists",
         "status": 400
     },
     "EmailAlreadyExistsError": {
         "message": "User with given email address already exists",
         "status": 400
     },
     "UnauthorizedError": {
         "message": "Invalid username or password",
         "status": 401
     }
}
```

As you can see firs we have extended the `Exception` class to create different custom exceptions and then we created an `errors` dictionary, which contains the error message and status codes for each exception. Now, we need to add these errors to the `flask-restful` `Api` class.

Update `app.py` to import recently created `errors` dictionary and add this as a parameter to `Api` class.

```diff
#~/movie-bag/app.py

from database.db import initialize_db
 from flask_restful import Api
 from resources.routes import initialize_routes
+from resources.errors import errors
 
 app = Flask(__name__)
 app.config.from_envvar('ENV_FILE_LOCATION')
 
-api = Api(app)
+api = Api(app, errors=errors)
 bcrypt = Bcrypt(app)
 jwt = JWTManager(app)

```

Finally, we are ready to perform some exception handling in our application. Update `movie.py` view functions according to the following:

```diff
#~/movie-bag/resources/movie.py
 from flask import Response, request
 from database.models import Movie, User
 from flask_jwt_extended import jwt_required, get_jwt_identity
 from flask_restful import Resource
+
+from mongoengine.errors import FieldDoesNotExist, NotUniqueError, DoesNotExist, ValidationError, InvalidQueryError

+from resources.errors import SchemaValidationError, MovieAlreadyExistsError, InternalServerError, \
+UpdatingMovieError, DeletingMovieError, MovieNotExistsError
+
 
 class MoviesApi(Resource):
     def get(self):
@@ -11,32 +15,57 @@ class MoviesApi(Resource):
 
     @jwt_required
     def post(self):
-        user_id = get_jwt_identity()
-        body = request.get_json()
-        user = User.objects.get(id=user_id)
-        movie =  Movie(**body, added_by=user)
-        movie.save()
-        user.update(push__movies=movie)
-        user.save()
-        id = movie.id
-        return {'id': str(id)}, 200
-        
+        try:
+            user_id = get_jwt_identity()
+            body = request.get_json()
+            user = User.objects.get(id=user_id)
+            movie =  Movie(**body, added_by=user)
+            movie.save()
+            user.update(push__movies=movie)
+            user.save()
+            id = movie.id
+            return {'id': str(id)}, 200
+        except (FieldDoesNotExist, ValidationError):
+            raise SchemaValidationError
+        except NotUniqueError:
+            raise MovieAlreadyExistsError
+        except Exception as e:
+            raise InternalServerError
+
+
 class MovieApi(Resource):
     @jwt_required
     def put(self, id):
-        user_id = get_jwt_identity()
-        movie = Movie.objects.get(id=id, added_by=user_id)
-        body = request.get_json()
-        Movie.objects.get(id=id).update(**body)
-        return '', 200
+        try:
+            user_id = get_jwt_identity()
+            movie = Movie.objects.get(id=id, added_by=user_id)
+            body = request.get_json()
+            Movie.objects.get(id=id).update(**body)
+            return '', 200
+        except InvalidQueryError:
+            raise SchemaValidationError
+        except DoesNotExist:
+            raise UpdatingMovieError
+        except Exception:
+            raise InternalServerError       
     
     @jwt_required
     def delete(self, id):
-        user_id = get_jwt_identity()
-        movie = Movie.objects.get(id=id, added_by=user_id)
-        movie.delete()
-        return '', 200
+        try:
+            user_id = get_jwt_identity()
+            movie = Movie.objects.get(id=id, added_by=user_id)
+            movie.delete()
+            return '', 200
+        except DoesNotExist:
+            raise DeletingMovieError
+        except Exception:
+            raise InternalServerError
 
     def get(self, id):
-        movies = Movie.objects.get(id=id).to_json()
-        return Response(movies, mimetype="application/json", status=200)
+        try:
+            movies = Movie.objects.get(id=id).to_json()
+            return Response(movies, mimetype="application/json", status=200)
+        except DoesNotExist:
+            raise MovieNotExistsError
+        except Exception:
+            raise InternalServerError


```

Let's see the example of `post` method in `MoviesApi` class:

```python
def post(self):
  try:
      user_id = get_jwt_identity()
      body = request.get_json()
      user = User.objects.get(id=user_id)
      movie =  Movie(**body, added_by=user)
      movie.save()
      user.update(push__movies=movie)
      user.save()
      id = movie.id
      return {'id': str(id)}, 200
  except (FieldDoesNotExist, ValidationError):
      raise SchemaValidationError
  except NotUniqueError:
      raise MovieAlreadyExistsError
  except Exception as e:
      raise InternalServerError
```
Here you can see we have wrapped the whole view opetations in `try...except` block. We have performed an exception chaining so, that when we get some excetion we throw the exceptions which we have defined in `errors.py` and `flask-restful` generates a response based on the values we defined in `errors` dictionary.

When there is `FieldDoesNotExist` exception or `ValidationError` from `mongoengine` we raise `SchemaValidationError` exception which tells the client that their request JSON is invalid. Similarly, when a user tries to a movie with the name which already exists `mongoengine` throws `NotUniqueError` exception and by catching that exception we raise `MovieAlreadyExistsError` which tells the user that the movie name already exists.

And lastly, we get an exception that we have not expected then we throw an `InternalServerError`. 

Let's add similar exception handling to our `auth.py`

```diff
#~/movie-bag/resources/auth.py

 from database.models import User
 from flask_restful import Resource
 import datetime
+from mongoengine.errors import FieldDoesNotExist, NotUniqueError, DoesNotExist
+from resources.errors import SchemaValidationError, EmailAlreadyExistsError, UnauthorizedError, \
+InternalServerError
 
 class SignupApi(Resource):
     def post(self):
-        body = request.get_json()
-        user =  User(**body)
-        user.hash_password()
-        user.save()
-        id = user.id
-        return {'id': str(id)}, 200
+        try:
+            body = request.get_json()
+            user =  User(**body)
+            user.hash_password()
+            user.save()
+            id = user.id
+            return {'id': str(id)}, 200
+        except FieldDoesNotExist:
+            raise SchemaValidationError
+        except NotUniqueError:
+            raise EmailAlreadyExistsError
+        except Exception as e:
+            raise InternalServerError
 
 class LoginApi(Resource):
     def post(self):
-        body = request.get_json()
-        user = User.objects.get(email=body.get('email'))
-        authorized = user.check_password(body.get('password'))
-        if not authorized:
-            return {'error': 'Email or password invalid'}, 401
-        expires = datetime.timedelta(days=7)
-        access_token = create_access_token(identity=str(user.id), expires_delta=expires)
-        return {'token': access_token}, 200
+        try:
+            body = request.get_json()
+            user = User.objects.get(email=body.get('email'))
+            authorized = user.check_password(body.get('password'))
+            if not authorized:
+                raise UnauthorizedError
+ 
+            expires = datetime.timedelta(days=7)
+            access_token = create_access_token(identity=str(user.id), expires_delta=expires)
+            return {'token': access_token}, 200
+        except (UnauthorizedError, DoesNotExist):
+            raise UnauthorizedError
+        except Exception as e:
+            raise InternalServerError
```


That's it, people. Now, when there is an error in our application we get a proper error message with relevant status code.

Let's try creating a user with `/api/auth/signup` with some email address let's say `testemail@gmail.com`. Now again try to create another user with the same email address. We get a response like this:

```json
{
    "message": "User with given email address already exists",
    "status": 400
}
```
Now, the user of our application can easily know what went wrong.

You can find the complete code of this part [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%204)

### What we learned from this part of the series?
- How to handle exceptions in our flask application using exception chaining.
- How to send the error message and status code based on the exception occurred.

<hr>
In the next part of the series, we are going to learn how to perform a password reset in our application.
<hr>

Until then happy coding 😊