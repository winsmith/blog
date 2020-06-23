---
layout: post
title:  "Authenticate Django Services with JWT"
subtitle: "Distribute authentication tokens through your Django microservices or services"
date:   2020-06-23 17:00:00 +0100
categories: python django development services
---

<p class="lead">My client has a collection of services written in Python and Django that run inside Docker. Before, these all had their own authentication management, meaning that you would have to register a user for each service separately.</p>

<p class="lead">So we decided to add a new service, Auth, that would manage user authentication and then share a login token with the other  services. This way, a user would only have to register in the auth service, and once they were logged in they were logged in to all other services. </p>

<p class="lead">In this blog post, I will go into how we added JWT based shared authentication. In the follow up, I will show how to create a Django Authentication Middleware to consume the JWT tokens to authenticate users.</p>

----

Since we were revamping the auth system anyway, we also decided to remove the need for user names, which Django Users have by default. Instead, we decided users should identify themselves using their email address. We save a randomly generated UUID4 into the users username field. 

<p class="small">If you want to keep using usernames, you can ignore all `Form` subclasses in this article, because those overwrite and implement that behaviour. The rest should work the same.</p>

This is the road we are going to take: 

- We use **cookies** that contain **JSON Web Tokens** 
- we save them from the **auth server in Django** and 
- **read them in Django's user middleware** in the services. 

See below for an explanation of each of these parts.

This article was developed and tested using Python 3.7 and Django 3.0.3. The target audience is advanced Django programmers.

## Cookies

How does this work? 

- All the services are hosted on sub domains of the same central domain, so they are able to share cookies between them. 
- The user can log in to the auth service, which sets a cookie containing a token. 
- The token can be read by other services, and contains authentication information. 
- Since the cookie is signed (see below), the other services can trust that the information provided in the cookie is untampered and trustworthy, and do *not* need to contact the auth server for confirmation.

## JWT – JSON Web Tokens
[JSON Web Tokens](https://jwt.io) are, according to [jwt.io](https://jwt.io):

> JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. JWTs can be signed using a secret (with the HMAC algorithm) or a public/private key pair using RSA or ECDSA.

<img class="img-fluid" src="/assets/jwt-screenshot.png" alt="A screenshot of a JWT in encoded and decoded form">

JSON Web Tokens consist of three parts separated by dots (`.`), which are, header, payload and signature.

The header defines the signing algorithm used such as HMAC SHA256 or RSA and the type of the token, which is JWT. It is encoded using Base64Url.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

The payload is a dictionary that contains various fixed keys such as `iss` (issuer), `exp` (expiration time), or `sub` (subject), but can in addition also use any other key we like. We are going to use this fact. This is also encoded using Base64Url.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

To create the signature part you have to take the encoded header, the encoded payload, a secret, the algorithm specified in the header, and sign that. The signature is used to verify the message wasn't changed along the way, and, in the case of tokens signed with a private key, it can also verify that the sender of the JWT is who it says it is. 

However, JWTs are not encrypted. Anyone can read them, it's just not possible to tamper* with them.

Header, payload and signature are concatenated using dots, so a JWT typically looks like the following:

```
xxxxx.yyyyy.zzzzz
```

This is the string that we are going to stuff into a cookie.

## Saving Cookies in the Auth Service
In the auth server, we want to use Django's user authentication system to authenticate the user and then save a generated JWT into a special cookie for the other services to read.

### New Project
Let's start with creating a new Django project named `auth` for our service, and a new app inside this project named `jwtauth`:

```bash
$ django-admin startproject auth
$ cd auth/
$ ./manage.py startapp jwtauth 
```

### Setup Settings and Libraries
The next steps are to add this new project to your `INSTALLED_APPS`, and include its URLs. Please do so now.

Also install the `djangorestframework-simplejwt~=4.4` library as a dependency using Pip. We're using `djangorestframework-simplejwt`, but you could just as well use any other of the libraries listed on [jwt.io](https://jwt.io).

Let's add some configuration for SimpleJWT. Add this to your `settings.py`:

```python
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=5),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=5),
    "ALGORITHM": "HS512",
    "SIGNING_KEY": os.environ["JWT_SIGNING_KEY"],
}
```

This will increase your token' lifetimes, use a stronger signing algorithm, and get the signing key from the environment so you don't have to hardcode it into settings. The signing key needs to be the same between all of your services. It can be any string with more bits than your algorithm's key size.

### Creating the Views
We are going to use Django's regular user management, and we are going to add these two behaviours to it:

1. during login, a cookie is stored on the user's computer
2. during logout that cookie is removed. 

For this, we are going to  create new views for registration, login, and logout.

First, let's create a `TokenPayloadMixin` in `views.py`. This will encapsulate the code to set a JWT cookie derived from a user:

```python
# Give your cookies a descriptive name
ACCESS_COOKIE_NAME = "org.breakthesystem.jwt.access"
REFRESH_COOKIE_NAME = "org.breakthesystem.jwt.refresh"

class TokenPayloadMixin:
    def prepare_token_payload(self, user, access_token):
        """
        Given an access token and a user, add the user's 
        meta data to the access token's payload
        """
        access_token.payload["user_uuid"] = user.username
        access_token.payload["user_email"] = user.email
        access_token.payload["user_is_staff"] = user.is_staff
        access_token.payload["user_is_superuser"] = user.is_superuser
        access_token.payload["user_first_name"] = user.first_name
        access_token.payload["user_last_name"] = user.last_name
        return access_token

    def set_cookies(self, response, access_token, refresh_token):
        """
        Save an accesss token and refresh token as cookies
        """
        domain = settings.COOKIE_DOMAIN

        response.set_cookie(
            ACCESS_COOKIE_NAME,
            str(access_token),
            expires=datetime.now() + settings.SIMPLE_JWT["ACCESS_TOKEN_LIFETIME"],
            domain=domain,
            httponly=True,
        )
        response.set_cookie(
            REFRESH_COOKIE_NAME,
            str(refresh_token),
            expires=datetime.now() + settings.SIMPLE_JWT["REFRESH_TOKEN_LIFETIME"],
            domain=domain,
            httponly=True,
        )
```

Now we can create our views. 

#### Registration View
Registration works just as you'd normally expect in Django -- except for one difference: We decided to not support a username and email address. Instead, users identify themselves with only their email address. We save a UUID into the username field of the Django User Model. This means we are using our own `RegistrationForm` instead of the one provided by Django.

Let's create a `RegistrationForm` for registration in `forms.py`:

```python
class RegistrationForm(forms.Form):
    first_name = forms.CharField(widget=forms.TextInput(attrs={"autofocus": True}))
    last_name = forms.CharField()
    email = forms.EmailField()
    password = forms.CharField(strip=False, widget=forms.PasswordInput())
    confirm_password = forms.CharField(strip=False, widget=forms.PasswordInput())
    next = forms.CharField(required=False, widget=forms.HiddenInput())

    def clean(self):
        super().clean()

        password = self.cleaned_data.get("password")
        password2 = self.cleaned_data.get("confirm_password")

        # Check if a valid email address was entered
        if not "email" in self.cleaned_data:
            raise forms.ValidationError(
                "Please enter a valid email address.", code="email_invalid"
            )

        # Make sure that email isn't already taken
        if UserModel.objects.filter(email=self.cleaned_data["email"]).count() > 0:
            raise forms.ValidationError(
                "Sorry, that email address is already taken. Are you sure you don't want to login instead?", code="email_already_taken"
            )

        # Make sure password passes validation
        validate_password(password)

        # Make sure passwords match
        if password and password2 and password != password2:
            raise forms.ValidationError("Your passwords do not match. Please make sure you enter the same password in both fields.")
```

Now let's create a registration view in `views.py`. If the registration form gives the green light, it will create a user.

```python
class RegistrationView(TokenPayloadMixin, FormView):
    template_name = "jwtauth/register.html"
    form_class = RegistrationForm
    success_url = "/"

    def get_context_data(self, **kwargs):
        context = super().get_context_data()
        context["next"] = self.request.GET.get("next")
        return context

    def form_valid(self, form):
        email = form.cleaned_data["email"]
        password = form.cleaned_data["password"]
        first_name = form.cleaned_data["first_name"]
        last_name = form.cleaned_data["last_name"]
        user_id = uuid4()

        user = UserModel.objects.create_user(username=user_id, email=email, password=password)
        user.first_name = first_name
        user.last_name = last_name
        user.save()

        messages.success(self.request, f"Your account was successfully created. Please log in now.")

        return super().form_valid(form)
```

#### Login View
Login needs a form, so lets create a new `Form` in `forms.py`. This class started out as a copy of the `LoginForm` included with Django, and we modified it slightly to use an email address instead of a username.

```python
class LoginForm(forms.Form):
    """
    Form for authenticating users using email and password
    """

    email = forms.EmailField(widget=forms.TextInput(attrs={"autofocus": True}))
    password = forms.CharField(strip=False, widget=forms.PasswordInput(attrs={"autocomplete": "current-password"}),)
    next = forms.CharField(required=False, widget=forms.HiddenInput())

    error_messages = {
        "invalid_login": ("Please enter a correct email address and password. Note that both fields may be case-sensitive."),
        "inactive": ("This account is inactive."),
    }

    def __init__(self, request=None, *args, **kwargs):
        """
        The 'request' parameter is set for custom auth use by subclasses.
        The form data comes in via the standard 'data' kwarg.
        """
        self.request = request
        self.user_cache = None
        super().__init__(*args, **kwargs)

    def clean(self):
        email = self.cleaned_data.get("email")
        password = self.cleaned_data.get("password")

        # Check if user exists and can login
        if email is not None and password:
            try:
                user = UserModel.objects.get(email=email)
                self.user_cache = authenticate(self.request, username=user.username, password=password)
                if self.user_cache is None:
                    raise self.get_invalid_login_error()
                else:
                    self.confirm_login_allowed(self.user_cache)
            except UserModel.DoesNotExist:
                raise forms.ValidationError(
                    self.error_messages["invalid_login"], code="invalid_login", params={},
                )

        return self.cleaned_data

    def confirm_login_allowed(self, user):
        """
        Controls whether the given User may log in. This is a policy setting,
        independent of end-user authentication. This default behavior is to
        allow login by active users, and reject login by inactive users.

        If the given user cannot log in, this method should raise a
        ``forms.ValidationError``.

        If the given user may log in, this method should return None.
        """
        if not user.is_active:
            raise forms.ValidationError(
                self.error_messages["inactive"], code="inactive",
            )

    def get_user(self):
        return self.user_cache

    def get_invalid_login_error(self):
        return forms.ValidationError(self.error_messages["invalid_login"], code="invalid_login", params={},)
```

Now let's write a `LoginView` that uses our form:

First, `get_initial` and `get_context_data` are used to store the `next` parameter. Other services will supply their URL as `next` so we can forward to the URL after the user is logged in.

```python
class LoginView(TokenPayloadMixin, FormView):
    template_name = "jwtauth/login.html"
    form_class = LoginForm
    success_url = "/"

    def get_initial(self):
        # Get the "next" parameter, which stores the address
        # we should forward to after login.
        initial_data = super().get_initial()
        next = self.request.GET.get("next")
        if next is not None:
            initial_data["next"] = next
        return initial_data

    def get_context_data(self, **kwargs):
        # Store the "next" parameter in the context so we
        # can also hand it over to registration if needed
        context = super().get_context_data()
        context["next"] = self.request.GET.get("next")
        return context
```

Next, we check wether a token is present in the cookie, if its valid or refreshable. If not, we present a login form, and if it is, we log the user in properly and redirect to the `next` address. This happens in the `get` function:

```python
    def get(self, request, *args, **kwargs):
        # Retrieve the token from cookie
        refresh_token_str = request.COOKIES.get(REFRESH_COOKIE_NAME)
        
        # If no token present, show the login form
        if refresh_token_str is None:
            return super().get(request, *args, **kwargs)

        # If the token is invalid, log out
        try:
            refresh_token = RefreshToken(refresh_token_str)
        except TokenError:
            return super().get(request, *args, **kwargs)

        # If the Django auth system has lost our authentication
        # session, but we still have a JWT cookie, log the 
        # user back in
        if not self.request.user.is_authenticated:
            user_id = refresh_token.payload.get("user_id")
            user = UserModel.objects.get(id=user_id)
            login(request, user)

        # If we reach this far down, we have a valid refresh
        # token, but no access token. So let's create an 
        # access token from the refresh token
        access_token = self.prepare_token_payload(self.request.user, refresh_token.access_token)

        # We are almost done. Let's prepare a response. If 
        # a `next` parameter was supplied, redirect to that.
        # Else, just show the view again.
        next = self.request.GET.get("next")
        response = None
        if next is None:
            response = super().get(request, *args, **kwargs)
        else:
            response = HttpResponseRedirect(next)

        # With the response prepared, let's set the cookie
        # inside this response. 
        self.set_cookies(response, access_token, refresh_token)
        return response
```

Finally, the `form_valid` function is called when the form is posted and the `Form` class has found no objections. This means we have valid user data and should log the user in, and save the cookie:

```python
    def form_valid(self, form):
        # Get all data from the form
        email = form.cleaned_data["email"]
        user_instance = UserModel.objects.get(email=email)

        # Check if the user can be authenticated
        username = user_instance.username
        password = form.cleaned_data["password"]
        user = authenticate(self.request, username=username, password=password)

        # If the email/password combo is wrong, don't log the
        # user in
        if user is None:
            messages.error(self.request, "Sorry, no user exists with this combination of email and password.")
            return super().form_valid(form)

        # Otherwise, log the user in
        login(self.request, user)

        # We are almost done. Let's prepare a response. If 
        # a `next` parameter was supplied, redirect to that.
        # Else, just show the view again.
        response = super().form_valid(form)
        next = form.cleaned_data["next"]
        if next is not None:
            response = HttpResponseRedirect(next)

        # With the response prepared, let's set the cookie
        # inside this response. 
        access_token = self.prepare_token_payload(user, AccessToken.for_user(user))
        refresh_token = RefreshToken.for_user(user)
        self.set_cookies(response, access_token, refresh_token)
        return response
```


#### Logout View

Finally, logout. This is pretty straight forward, we log the user out and delete the cookies.

```python
class LogoutView(RedirectView):
    url = "/"

    def get(self, request, *args, **kwargs):
        domain = settings.COOKIE_DOMAIN

        response = super().get(request, *args, **kwargs)
        response.delete_cookie(ACCESS_COOKIE_NAME, domain=domain)
        response.delete_cookie(REFRESH_COOKIE_NAME, domain=domain)

        logout(request)

        messages.info(self.request, "You are now logged out.")
        return response
```


## Reading Cookies in Services
See the next article on how to retrieve and login users with the JWT cookies set.