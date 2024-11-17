# Djoser
Djoser is a Python library that simplifies the implementation of user authentication and account management for Django projects. It provides a set of reusable views, serializers, and endpoints to handle common authentication workflows, such as user registration, login, logout, password reset, account activation, and more.


## Why Use Djoser?
Djoser streamlines user authentication by handling many repetitive tasks, allowing developers to focus on building their application's core functionality. It works seamlessly with Django's default authentication system and is compatible with third-party authentication packages like Django Rest Framework (DRF).

## Key Features
1. Token-Based Authentication:

- Supports JWT, TokenAuthentication, or other authentication backends supported by Django Rest Framework.
- Easily integrates with libraries like djangorestframework-simplejwt or django-rest-framework-jwt.

2. Out-of-the-Box Endpoints:

- Provides pre-built RESTful API endpoints for:
- User registration
- Account activation
- Password reset and change
- Login and logout
- Token management
- User profile retrieval and update

3. Highly Customizable:

- Developers can override its default behavior (e.g., views, serializers, email templates) to meet specific project requirements.
- Supports customization of email templates for account activation, password reset, etc.

4. Email-Based Workflows:

- Supports email-based verification for actions like account activation or password reset.
- Automatically generates secure tokens for user-related operations.

5. Integration with Django's Built-In Authentication:

- Works with Django's AbstractUser and AbstractBaseUser.
- Leverages Django's User model, making it easy to extend for custom user models.

6. Multi-Platform Support:

- API-driven, making it ideal for applications that need a backend for mobile apps, single-page applications (SPAs), or any client-side framework (e.g., React, Angular).


## Installation
1. Install Djoser:
```pip install djoser```

2. Add to `INSTALLED_APPS`:
```
  INSTALLED_APPS = [
    ...
    'djoser',
    'rest_framework',
    'rest_framework.authtoken',  # If using token-based auth
]
  ```

3. Configure URLs: Add Djoser's routes to your `urls.py`:
```
from django.urls import path, include

urlpatterns = [
    path('auth/', include('djoser.urls')),  # Includes default endpoints
    path('auth/', include('djoser.urls.jwt')),  # Includes JWT endpoints (optional)
]
```

4. Configure Settings (Optional): Customize behavior in `settings.py`:
```
DJOSER = {
    'LOGIN_FIELD': 'email',  # Use email as the login field
    'SEND_ACTIVATION_EMAIL': True,  # Enable activation email
    'ACTIVATION_URL': 'activate/{uid}/{token}/',  # URL for account activation
    'PASSWORD_RESET_CONFIRM_URL': 'password-reset/{uid}/{token}/',  # URL for password reset
    'SERIALIZERS': {
        'user_create': 'myapp.serializers.CustomUserCreateSerializer',
        'user': 'myapp.serializers.CustomUserSerializer',
    },
}
```


## How Djoser Works
Djoser provides a set of RESTful endpoints that map to specific user authentication actions. For example:

```
Endpoint	                          Purpose
/auth/users/	                      Register a new user
/auth/users/activation/	            Activate a user account via email
/auth/jwt/create/	                  Obtain a JSON Web Token (JWT)
/auth/jwt/refresh/	                Refresh an existing JWT
/auth/jwt/verify/	                  Verify the validity of a JWT
/auth/password/reset/	              Request a password reset
/auth/password/reset/confirm/	      Confirm the password reset
/auth/users/me/	                    Retrieve or update the authenticated user's data
/auth/logout/	                      Log the user out
```










