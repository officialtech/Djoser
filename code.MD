# Code



## settings
```
DJOSER = {
    'LOGIN_FIELD': 'email',
    'USER_CREATE_PASSWORD_RETYPE': True,
    'USERNAME_CHANGED_EMAIL_CONFIRMATION': True,
    'PASSWORD_CHANGED_EMAIL_CONFIRMATION': True,
    'SEND_CONFIRMATION_EMAIL': True,
    'SET_USERNAME_RETYPE': True,
    'SET_PASSWORD_RETYPE': True,
    'SEND_ACTIVATION_EMAIL': True,
    'PASSWORD_RESET_CONFIRM_URL': 'password/reset/confirm/{uid}/{token}',
    'USERNAME_RESET_CONFIRM_URL': 'email/reset/confirm/{uid}/{token}',
    'ACTIVATION_URL': 'auth/activate/{uid}/{token}',
    'SERIALIZERS': {
        'user_create': 'myapp.serializers.CustomUserCreateSerializer',
    },
    'EMAIL': {
        'activation': 'myapp.emails.CustomActivationEmail',
        'password_changed_confirmation': 'myapp.emails.CustomPasswordChangedConfirmationEmail',
    },
}
```



## serializer

```
from djoser.serializers import UserCreateSerializer, UidAndTokenSerializer
from djoser.conf import settings as djoser_settings
from rest_framework import serializers, exceptions

class CustomUserCreateSerializer(UserCreateSerializer):
    def validate(self, attrs):
        attrs.update(self.initial_data)
        if attrs["password"] != attrs["re_password"]:
            raise serializers.ValidationError({"password": """Password fields didn't match."""})
        attrs.pop("re_password", None)
        attrs = super().validate(attrs)
        return attrs

    def create(self, validated_data):
        print(f"""Creating {validated_data.get("username")} user...""")
        user = super().create(validated_data)
        # add more logics below for extra
        return user



class CustomActivationSerializer(UidAndTokenSerializer):
    default_error_messages = {
        "stale_token": djoser_settings.CONSTANTS.messages.STALE_TOKEN_ERROR
    }

    def validate(self, attrs):
        attrs = super().validate(attrs)
        if not self.user.is_active:
            return attrs
        raise exceptions.PermissionDenied(self.error_messages["stale_token"])
```





## view

```
from djoser.views import (
    UserViewSet as DjoserViewSet,
)
from djoser import signals
from djoser.conf import settings as djoser_settings
from djoser.utils import decode_uid
from djoser.compat import get_user_email

from django.http import HttpResponse
from django.contrib.auth import get_user_model
from django.contrib.auth.tokens import default_token_generator

from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
)
from rest_framework.views import APIView
from rest_framework.generics import GenericAPIView
from rest_framework.response import Response
from rest_framework import status

# internal module
from .serializers import (
    CustomUserCreateSerializer,
    CustomActivationSerializer,
)



class CustomUserViewSet(DjoserViewSet):

    serializer_class = CustomUserCreateSerializer
    permission_classes = [AllowAny]

    def create(self, request, *args, **kwargs):
        serializer = self.serializer_class(data=request.data)

        if serializer.is_valid():
            self.perform_create(serializer)
            # Access the serializer data after saving the instance
            serializer_data = serializer.data
            headers = self.get_success_headers(serializer_data)
            return Response({
                "status": 201,
                "data": serializer_data,
                "message": "Your account has been created successfully, please check your email to proceed further"
            }, status=status.HTTP_201_CREATED, headers=headers)

        else:
            serializer_data = serializer.errors
            serializer_data["success"] = False
            if serializer_data.get("email"):
                serializer_data["message"] = "User account with this email address is already exists"
            return Response({
                "status": 400,
                "data": serializer_data,
            }, status=status.HTTP_400_BAD_REQUEST)

    def perform_create(self, serializer, *args, **kwargs):
        user = serializer.save(*args, **kwargs)

        context = {"user": user}
        to = [user.email]
        if djoser_settings.SEND_ACTIVATION_EMAIL:
            djoser_settings.EMAIL.activation(self.request, context).send(to)
        elif djoser_settings.SEND_CONFIRMATION_EMAIL:
            djoser_settings.EMAIL.confirmation(self.request, context).send(to)
        return user


# -----------------------------------------------
# Custom account activation view
# -----------------------------------------------
class CustomDjoserActivationView(GenericAPIView):
    """Inheriting the Djoser's `UserViewSet` class
    to use `activation` method

    Args:
        UserViewSet (viewset): ModelViewSet
    """
    permission_classes = [AllowAny]
    # ALERT: check settings includes carefully
    serializer_class = CustomActivationSerializer

    def get(self, request, uid=None, token=None, *args, **kwargs):
        try:
            print("Activating the account...")
            User = get_user_model()
            uid = decode_uid(uid)
            user = User.objects.get(pk=uid)  # AUTH_USER_MODEL
            if user.is_active:
                return HttpResponse(
                    """
                        <div style="text-align: center; padding: 20px;">
                            <h2 style="color: orange;">Account Already Activated!</h2>
                        </div>
                    """,
                    status=status.HTTP_403_FORBIDDEN,
                )

            if default_token_generator.check_token(user, token):
                user.is_active = True
                user.save()

                # Send the user_activated signal
                signals.user_activated.send(
                    sender=self.__class__, user=user, request=self.request)

                if djoser_settings.SEND_CONFIRMATION_EMAIL:
                    context = {"user": user}
                    to = [get_user_email(user)]
                    djoser_settings.EMAIL.confirmation(
                        self.request, context).send(to)

                print("Account activated successfully!")

                return HttpResponse(
                    """
                        <div style="text-align: center; padding: 20px;">
                            <h2 style="color: green;">Account Activated Successfully!</h2>
                        </div>
                    """,
                    status=status.HTTP_201_CREATED,
                )
            else:
                return HttpResponse(
                    """
                    <div style="text-align: center; padding: 20px;">
                        <h2 style="color: red;">Account Activated Failed!</h2>
                    </div>
                """,
                    status=status.HTTP_401_UNAUTHORIZED,
                )

        except Exception as acex:
            print(f"{acex}")
            return HttpResponse(
                f"""
                    <div style="text-align: center; padding: 20px;">
                        <h2 style="color: red;">{acex}</h2>
                    </div>
                """,
                status=status.HTTP_400_BAD_REQUEST,
            )

```


## email

```
from datetime import datetime

from django.templatetags.static import static
from django.template.loader import render_to_string
from django.core.mail import EmailMultiAlternatives
from django.contrib.auth.tokens import default_token_generator

from djoser import email
from djoser.conf import settings


class CustomActivationEmail(email.ActivationEmail):
    template_name = 'CustomActivationEmail.html'

    def get_context_data(self):
        context = super().get_context_data()
        user = context.get("user")
        context["token"] = default_token_generator.make_token(user)
        context["url"] = settings.ACTIVATION_URL.format(**context)
        context["logo_url"] = static('email_template_panomiq_static/logo.png')

        context["template_name"] = self.template_name
        user_object = context.get('user')
        user_email = user_object.email
        context["user_email"] = user_email
        current_year = datetime.today().year
        activation_url = f"""{context.get("protocol")}//{context.get("site_name")}/{context.get("url")}/"""
        return context
    
    # No need this default behaviour is enough, you can comment below code
    def send(self, context, *args, **kwargs):
        """ this is only required if you want to override default email sending behaviour
        """
        # Prevent automatic email sending
        user_email = context["user_email"]
        send_email_confirmation(
            context=context,
            activation_url=context["activation_url"],
            current_year=context["current_year"],
            html_template=self.template_name, 
            to_email_list=[user_email, ],
        )

class CustomPasswordChangedConfirmationEmail(email.PasswordChangedConfirmationEmail):
    """Overriding the djoser's `CustomPasswordChangedConfirmationEmail` class 
    and inheriting the `BaseEmailMessage` base class from `templated_mail`
    """
    template_name = "registration/templates/password_changed_confirmation.html"

    def get_context_data(self, *args, **kwargs):
        context = super().get_context_data()
        user_object = context.get('user')
        user_email = user_object.email
        user_name = user_object.name
        otp_secret = user_object.otp_secret
        return context
```


## app urls
```
from django.urls import path
from . import views


urlpatterns = [
    path(
        "auth/users/",
        views.CustomUserViewSet.as_view({"post": "create"}), 
        name="user-create",
    ),
    path(
        "auth/activate/<str:uid>/<str:token>/",
        views.CustomDjoserActivationView.as_view(),
        name="activation",
    ),
]
```


## project urls

```
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("yourapp.urls"), ),
    path("auth/", include("djoser.urls")),
    path("auth/", include("djoser.urls.jwt")),
]
```




