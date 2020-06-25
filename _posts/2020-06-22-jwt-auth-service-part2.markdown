---
layout: post
title:  "Authenticate Django Services with JWT — Part 2"
subtitle: "Consume Authentication Tokens and Log Users In"
date:   2020-06-24 15:00:00 +0100
categories: python django development services
---

<p class="lead">This is part 2 of a two-part series. If you haven't done so, I recommend you read <a href="/python/django/development/services/2020/06/23/jwt-auth-service-part1.html">Part 1</a> first.</p>

<p class="lead">In the first part, we created a new service, Auth, that would authenticate a user and then store information about them in cookies using JSON Web Tokens. This post is about authenticating users in our other services, by replacing the default Django Authentication Middleware.</p>

This article is about how to pick up Django users from the JWT data in the previous post, so you can use Django to render templates, manage users, etc. If you don't want or need the Django Authentication system and, for example, only want to authenticate API requests, check out the [SimpleJWT documentation](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/) — you might find all you need already there.

<p>
<div class="alert alert-info" role="alert">
This article was developed and tested using Python 3.7 and Django 3.0.3. The target audience is advanced Django programmers.
</div>
</p>
----

## Creating a Project

Let's go to one of our services and add a new app, called `jwtauthmiddleware`. (Later, I recommend you put that app into its own repository and install it via pip, but this is a good way of stepping through the code.)

```bash
$ cd some-service/
$ ./manage.py startapp jwtauthmiddleware 
```

## Middleware

Let's write a middleware! Django middlewares, at their easiest, consist of a class that contains one method, `process_request`, which takes a `request` object and modifies it somehow. In our case we are going to instantiate a user from the JWT data and set it as `request.user`, very much like the normal `AuthenticationMiddleware` might do.

Inside the `jwtauthmiddleware` package, create a new class for our middleware, for example in `__init__.py`:

```python

class JWTAuthenticationMiddleware(AuthenticationMiddleware):
    def get_user(self, request):
        # Retrieve the token from cookie
        access = request.COOKIES.get("org.breakthesystem.jwt.access")
        refresh = request.COOKIES.get("org.breakthesystem.jwt.refresh")

        # Check for invalid or expired token
        try:
            token = AccessToken(access)
        except TokenError:
            return AnonymousUser()

        # Retrieve Token Payload Data
        user_uuid = token.payload.get("user_uuid")
        user_email = token.payload.get("user_email")
        user_is_staff = token.payload.get("user_is_staff")
        user_is_superuser = token.payload.get("user_is_superuser")

        user_first_name = token.payload.get("user_first_name")
        user_last_name = token.payload.get("user_last_name")

        # Make sure the the payload data is actually present
        for variable in [access, refresh, user_uuid, user_email, user_is_staff, user_is_superuser]:
            if variable is None:
                return AnonymousUser()

        # Create a new user. There's no need to set a 
        # password because it is not used to log in directly
        user, created = User.objects.get_or_create(username=user_uuid)

        # Update the user's meta information, the access 
        # token payload is the canonical source of truth
        user.email = user_email
        user.is_staff = user_is_staff
        user.is_superuser = user_is_superuser
        user.first_name = user_first_name
        user.last_name = user_last_name
        user.save()

        return user or AnonymousUser()

    def process_request(self, request):
        request.user = SimpleLazyObject(lambda: self.get_user(request))
```

In the previous article, we saved the user's `username` into a dictionary field called `user_uuid`, because we have replaced usernames with randomly generated IDs. Here we take the `user_uuid` field from the dictionary and save it as `username` again. 

This helps us here: We don't have to care about database IDs and still are able to have unique usernames. If you want to display a user's identifying information, you should use their first name, last name, or email address. 

If in the previous post you decided to use actual usernames, this code should work just as well. You might want to consider renaming the `user_uuid` dictionary key though, as it is misleading in this case.

## Installing the Middleware
In your project's `settings.py`, change the `MIDDLEWARE` setting. Remove the entry `'django.contrib.auth.middleware.AuthenticationMiddleware'`, and instead add `'jwtauthmiddleware.JWTAuthenticationMiddleware'`.

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    
    # Remove this
    # 'django.contrib.auth.middleware.AuthenticationMiddleware',
    
    # Replace by this
    "jwtauthmiddleware.JWTAuthenticationMiddleware",
    
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

Also, we need to set a `LOGIN_URL` to the URL where our Auth service lives. This will redirect users who are not logged in to the Auth service URL. 

We also define another setting, `OWN_URL`, for reasons that will become clear in a minute.

```python
LOGIN_URL = "https://auth.breakthesystem.org/login"
OWN_URL = "https://someservice.breakhtesystem.org"
```

## Redirecting to Login
The middleware is all we *really* need at this point. Now, if your user is logged in to your Auth service, things will work exactly as expected. 

If your user is *not* logged in, they will be redirected to the Auth service URL. However, right now, they won't get redirected back after login. Why is that?

By default, when Django redirects you to the `LOGIN_URL`, it appends the current path as a parameter called `next`. The URL looks like this:


`https://auth.breakthesystem.org/login?next=/home/user/daniel`

If authentication lives inside the same service, this `next` path is enough. However, we are changing URLs here, so the Auth service has not enough information to redirect our users back to where they need to be. What we'd like instead is that the `next` parameter contains the whole URL, including host, of the current service.

So, let's do that! We are going to use [monkey patching](https://stackoverflow.com/a/5626250) to replace the next implementation to include our service's host.

```python
from django.contrib.auth import views as auth_views

REDIRECT_FIELD_NAME = "next"

def custom_redirect_to_login(next, login_url=None, redirect_field_name=REDIRECT_FIELD_NAME):
    """
    Redirect the user to the login page, passing the given 'next' page.
    """
    resolved_url = settings.LOGIN_URL

    login_url_parts = list(urlparse(resolved_url))
    if redirect_field_name:
        querystring = QueryDict(login_url_parts[4], mutable=True)
        querystring[redirect_field_name] = settings.DOMAIN + next
        login_url_parts[4] = querystring.urlencode(safe="/")

    redirect_url = urlunparse(login_url_parts)
    return HttpResponseRedirect(redirect_url)

auth_views.redirect_to_login = custom_redirect_to_login
```

<p>
<div class="alert alert-warning" role="alert">
It goes without saying that monkey patching should be done with care. If a future version of Django changes the method signature of the method we just patched, your app will crash.
This is one of the many reasons why you should pin your major and minor versions of dependencies in your requirements.txt file. This way, you'll get to try out new versions explicitly, and can deal with any API changes.
</div>
</p>

## And We're Done
By now you should have a working JWT Auth service that saves all necessary user information in cookies. You also created a Django Authentication Middleware replacement which will extract a User object from the stored cookie and saves it into `request.user`. 

If you have questions or comments about this article, please [contact me on twitter](https://twitter.com/breakthesystem/). 
