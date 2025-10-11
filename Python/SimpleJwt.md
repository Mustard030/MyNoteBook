安装

```bash
pip install django djangorestframework djangorestframework-simplejwt
```

配置

```python
INSTALLED_APPS = [
    'rest_framework',
    'rest_framework_simplejwt',
    ...
]

# DRF 全局默认认证类
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',  # 设置为通过JWT验证
    ),
}
```

路由

```python
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
    # JWT 内置 3 个视图
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

自定义令牌

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        # 添加自定义 payload
        token['username'] = user.username
        token['uid'] = user.id
        return token

    # 额外往 response 里塞数据
    def validate(self, attrs):
        data = super().validate(attrs)
        data.update({'uid': self.user.id, 'username': self.user.username})
        return data
        
# 配置方式一：在配置项中配置
# 默认配置可以去 site_packages/rest_framework_simplejwt/settings.py中查看
# https://django-rest-framework-simplejwt.readthedocs.io/en/latest/settings.html
SIMPLE_JWT = {
	"TOKEN_OBTAIN_SERIALIZER": "path.to.MyTokenObtainPairSerializer"
}
# 配置方式二：直接自定义TokenObtainPairView
class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer

# 并更换路由
from api.views import MyTokenObtainPairView
path('api/token/', MyTokenObtainPairView.as_view(), ...)
```
