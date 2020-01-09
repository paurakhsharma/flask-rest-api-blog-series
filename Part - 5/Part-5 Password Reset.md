## Part 5: Password Reset

Howdy! In the previous [Part](https://dev.to/paurakhsharma/flask-rest-api-part-4-exception-handling-5c6a) of the series, we learned how to handle errors in Flask and send a meaningful error message to the client.

In this part, we are going to implement a password reset feature in our application.
Here is the brief diagram of how the password reset flow is gonna look like.

![Password reset flow diagram](https://thepracticaldev.s3.amazonaws.com/i/6b04olppbml2z2n6gpo4.png)
*Password reset flow diagram*

We are going to use the `flask-jwt-extended` library to generate password reset token, the good thing is we have already installed it while implementing authentication. We need to send reset token to the user through email, for that we are going to use [Flask Mail](https://pythonhosted.org/flask-mail/).

```
pipenv install flask-mail
```

Let's register this mail server in our `app.py`:

```diff
#~/movie-bag/app.py

from flask import Flask
 from flask_bcrypt import Bcrypt
 from flask_jwt_extended import JWTManager
+from flask_mail import Mail

 ...

 api = Api(app, errors=errors)
 bcrypt = Bcrypt(app)
 jwt = JWTManager(app)
+mail = Mail(app)

 app.config['MONGODB_SETTINGS'] = {
     'host': 'mongodb://localhost/movie-bag'
...
```

Now, let's create a service to send the email to the client, let's create a new folder `services` and a new file `mail_service.py` inside it. Add the following contents to the newly created file.

```bash
mkdir services
cd services
touch mail_service.py
```

```python
#~/movie-bag/services/mail_service.py

from threading import Thread
from flask_mail import Message

from app import app
from app import mail


def send_async_email(app, msg):
    with app.app_context():
        try:
            mail.send(msg)
        except ConnectionRefusedError:
            raise InternalServerError("[MAIL SERVER] not working")


def send_email(subject, sender, recipients, text_body, html_body):
    msg = Message(subject, sender=sender, recipients=recipients)
    msg.body = text_body
    msg.html = html_body
    Thread(target=send_async_email, args=(app, msg)).start()
```

Here you can see we have created a function `send_mail()` which takes `subject`, `sender`, `recipients`, `text_body` and `html_body` as arguments. It then creates a message object and runs `send_async_email()` in a separate thread, this is because while sending an email to the client we have to relay to the separate services such as Google, Outlook, etc.

Since these services can take some time to actually send the email, we are going to tell the client that their request was successful and start sending the email in a separate thread.

Now we are ready to implement the password reset. As shown in the diagram above we are going to create two different endpoints for this.

1) `/forget`: This endpoint takes the `email` of the user whose account needs to be changed. This endpoint then sends the email to the user with the link which contains reset token to reset the password.

2) `/reset`: This endpoint takes `reset_token` sent in the email and the new `password`.

Let's create a `reset_password.py` inside the `resources` folder. With the following code:

```python
#~/movie-bag/resources/reset_password.py

from flask import request, render_template
from flask_jwt_extended import create_access_token, decode_token
from database.models import User
from flask_restful import Resource
import datetime
from resources.errors import SchemaValidationError, InternalServerError, \
    EmailDoesnotExistsError, BadTokenError
from jwt.exceptions import ExpiredSignatureError, DecodeError, \
    InvalidTokenError
from services.mail_service import send_email

class ForgotPassword(Resource):
    def post(self):
        url = request.host_url + 'reset/'
        try:
            body = request.get_json()
            email = body.get('email')
            if not email:
                raise SchemaValidationError

            user = User.objects.get(email=email)
            if not user:
                raise EmailDoesnotExistsError

            expires = datetime.timedelta(hours=24)
            reset_token = create_access_token(str(user.id), expires_delta=expires)

            return send_email('[Movie-bag] Reset Your Password',
                              sender='support@movie-bag.com',
                              recipients=[user.email],
                              text_body=render_template('email/reset_password.txt',
                                                        url=url + reset_token),
                              html_body=render_template('email/reset_password.html',
                                                        url=url + reset_token))
        except SchemaValidationError:
            raise SchemaValidationError
        except EmailDoesnotExistsError:
            raise EmailDoesnotExistsError
        except Exception as e:
            raise InternalServerError


class ResetPassword(Resource):
    def post(self):
        url = request.host_url + 'reset/'
        try:
            body = request.get_json()
            reset_token = body.get('reset_token')
            password = body.get('password')

            if not reset_token or not password:
                raise SchemaValidationError

            user_id = decode_token(reset_token)['identity']

            user = User.objects.get(id=user_id)

            user.modify(password=password)
            user.hash_password()
            user.save()

            return send_email('[Movie-bag] Password reset successful',
                              sender='support@movie-bag.com',
                              recipients=[user.email],
                              text_body='Password reset was successful',
                              html_body='<p>Password reset was successful</p>')

        except SchemaValidationError:
            raise SchemaValidationError
        except ExpiredSignatureError:
            raise ExpiredTokenError
        except (DecodeError, InvalidTokenError):
            raise BadTokenError
        except Exception as e:
            raise InternalServerError
```

Here in the `ForgotPassword` resource, we first get the user based on the `email` provided by the client. We are then using `create_access_token()` to create a token based on `user.id` and this token expires in 24 hours. We are then sending the email to the client. The email contains both `HTML` and text format information.

Similarly in `ResetPassword` resource, we first get the user based on user id from the reset_token and then reset the password of the user based on the password provided by the user. Finally, a reset success email is sent to the user.

Let's create the new exceptions `EmailDoesnotExistsError` and `BadTokenError` in our `errors.py`.

```diff
#~/movie-bag/resources/errors.py

 class UnauthorizedError(Exception):
     pass

+class EmailDoesnotExistsError(Exception):
+    pass
+
+class BadTokenError(Exception):
+    pass
+
 errors = {
     "InternalServerError": {
         "message": "Something went wrong",
@@ -54,5 +60,13 @@ errors = {
      "UnauthorizedError": {
          "message": "Invalid username or password",
          "status": 401
+     },
+     "EmailDoesnotExistsError": {
+         "message": "Couldn't find the user with given email address",
+         "status": 400
+     },
+     "BadTokenError": {
+         "message": "Invalid token",
+         "status": 403
      }
 }
```

We need to create templates for HTML and text files that we need to send to the client. Let's create `templates` folder in our root directory, And inside `templates` create another folder `email` where we are creating two new files `reset_password.html` and `reset_password.txt`.

```bash
mkdir templates
cd templates
mkdir email
cd email
touch reset_password.html
touch reset_password.txt
```

In reset_password.html let's add the following:
```html
<!-- #~/movie-bag/templates/email/reset-password.html -->

<p>Dear, User</p>
<p>
    To reset your password
    <a href="{{ url }}">
        click here
    </a>.
</p>
<p>Alternatively, you can paste the following link in your browser's address bar:</p>
<p>{{ url }}</p>
<p>If you have not requested a password reset simply ignore this message.</p>
<p>Sincerely</p>
<p>Movie-bag Support Team</p>

```

Here `{{ url }}` substitutes the url we have sent earlier in the `render_template()` function.

Similarly, add the following in `reset_password.txt`:

```txt
Dear, User

To reset your password click on the following link:

{{ url }}

If you have not requested a password reset simply ignore this message.

Sincerely

Movie-bag Support Team
```

Now, we are ready to wire this `Resources` to our `routes.py`.

```diff
 from .movie import MoviesApi, MovieApi
 from .auth import SignupApi, LoginApi
+from .reset_password import ForgotPassword, ResetPassword

 def initialize_routes(api):

     ...

     api.add_resource(LoginApi, '/api/auth/login')
+
+    api.add_resource(ForgotPassword, '/api/auth/forgot')
+    api.add_resource(ResetPassword, '/api/auth/reset')

```

Now, if you try to run the application with `python app.py`

You'll see the error something like this:
```bash
ImportError: cannot import name 'initialize_routes' from 'resources.routes' (/home/paurakh/blog/flask/flask-restapi-series/movie-bag/resources/routes.py)
```

This is because of the circular dependency problem in python. In our `reset_password.py`, we import `send_mail` which is importing `app` from `app.py` whereas `app` is not yet defined on our `app.py`.

![Circular dependency](https://thepracticaldev.s3.amazonaws.com/i/3ymtwfv8cdeyc9t57bgh.png)

To solve this issue we are going to create another file `run.py` in our root directory, which will be responsible for running our app. Also, we need to initialize our routes/view functions after we have initialized our app.

```
touch run.py
```

Now, our `app.py` should look like this:

```diff
#~/movie-bag/app.py

 from database.db import initialize_db
 from flask_restful import Api
-from resources.routes import initialize_routes
 from resources.errors import errors

 app = Flask(__name__)
 app.config.from_envvar('ENV_FILE_LOCATION')
+mail = Mail(app)
+
+# imports requiring app and mail
+from resources.routes import initialize_routes

 api = Api(app, errors=errors)
 bcrypt = Bcrypt(app)
 jwt = JWTManager(app)
-mail = Mail(app)

...

 initialize_db(app)
 initialize_routes(api)
-
-app.run()
```

In our `run.py` we just run the app:

```python
#~/movie-bag/run.py

from app import app

app.run()
```

Add configuration for our `MAIL_SERVER` in `.env`
```diff

JWT_SECRET_KEY = 't1NP63m4wnBg6nyHYKfmc2TpCOGI4nss'
+MAIL_SERVER: "localhost"
+MAIL_PORT = "1025"
+MAIL_USERNAME = "support@movie-bag.com"
+MAIL_PASSWORD = ""
```

Start a SMTP server in next terminal with:
```bash
python -m smtpd -n -c DebuggingServer localhost:1025
```
This will create an SMTP server for testing our email feature.


Now run the app with
```bash
python run.py
```
*Note: remember to export `ENV_FILE_LOCATION`*

![Postmant forgot endpoint request](https://thepracticaldev.s3.amazonaws.com/i/2rrs1eu5v39t50sysbur.png)

If the email is of the existing user you can see the email in the terminal running `smtp` server as:

```html

<p>Dear, User</p>
<p>
    To reset your password
    <a href="http://localhost:3000/reset/eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzgzOTU0ODUsIm5iZiI6MTU3ODM5NTQ4NSwianRpIjoiZTEyZDg3ODgtMTkwZS00NWI1LWI0YzYtZTdkMTYzZjc5ZGZlIiwiZXhwIjoxNTc4NDgxODg1LCJpZGVudGl0eSI6IjVlMTQxNTJmOWRlNzQxZDNjNGYwYmNiYiIsImZyZXNoIjpmYWxzZSwidHlwZSI6ImFjY2VzcyJ9.dLJnhYTYMnLuLg_cHDdqi-jsXeISeMq75mb-ozaNxlw">
        click here
    </a>.
</p>
<p>Alternatively, you can paste the following link in your browser's address bar:</p>
<p>http://localhost:3000/reset/eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE1NzgzOTU0ODUsIm5iZiI6MTU3ODM5NTQ4NSwianRpIjoiZTEyZDg3ODgtMTkwZS00NWI1LWI0YzYtZTdkMTYzZjc5ZGZlIiwiZXhwIjoxNTc4NDgxODg1LCJpZGVudGl0eSI6IjVlMTQxNTJmOWRlNzQxZDNjNGYwYmNiYiIsImZyZXNoIjpmYWxzZSwidHlwZSI6ImFjY2VzcyJ9.dLJnhYTYMnLuLg_cHDdqi-jsXeISeMq75mb-ozaNxlw</p>
<p>If you have not requested a password reset simply ignore this message.</p>
<p>Sincerely</p>
<p>Movie-bag Support Team</p>
```

As you can see the URL is of format:

`http://localhost:3000/reset/<reset_token>`, you need to copy this token a send manually in your `/reset` endpoint.

*Note: We will learn how to implement to reset automatically in our front-end series but for now we need to manually copy the reset_token*

![Postman reset password request](https://thepracticaldev.s3.amazonaws.com/i/naz25gfxwa6dy6e9uo6l.png)

Congratulations your password is changed successfully. Now, you can log in with the new password.

You should also get the email stating your password was reset successfully.

```html
<p>Password reset was successful</p>
```

You can find all the code we have written till now [here](https://github.com/paurakhsharma/flask-rest-api-blog-series/tree/master/Part%20-%205)

### What we learned from this part of the series?
- How to create token for resetting user password
- How to send email to using `Flask-mail`
- How to reset user password
- How to avoid circular dependancy in flask.

In the next part of the series we are going to learn about testing our Flask REST APIs.

Until then, Happy Coding 😊