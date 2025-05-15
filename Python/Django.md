

# 数据库操作

## 读写分离
在 Django 中，**同一张表的读写分离（写操作走主库，读操作走从库）可以通过自定义数据库路由实现**
> setting.py
```python
DATABASES = {
    'default': {  # 主库
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'master_db',
        'USER': 'root',
        'PASSWORD': 'password',
        'HOST': 'master_host',
        'PORT': 3306,
    },
    'slave': {  # 从库
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'slave_db',
        'USER': 'slave_user',
        'PASSWORD': 'slave_pass',
        'HOST': 'slave_host',
        'PORT': 3306,
    }
}

DATABASE_ROUTERS = ['path.to.routers.AutoReadWriteRouter']
```

> routers.py
```python
class AutoReadWriteRouter:
    def db_for_read(self, model, **hints):
        """读操作走从库"""
        return 'slave'

    def db_for_write(self, model, **hints):
        """写操作走主库"""
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        """允许跨库关联（根据需求调整）"""
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        """迁移时仅操作主库"""
        return db == 'default'
```
使用：
```
# 写入主库
User.objects.create(name="Alice")  # 使用 default 数据库
# 读取从库
users = User.objects.all()  # 使用 slave 数据库
# 强制读主库（例如需要实时数据时）
user = User.objects.using('default').get(id=1)
```

多从库负载均衡
```python
import random

class LoadBalancedRouter:
    SLAVE_DATABASES = ['slave1', 'slave2']

    def db_for_read(self, model, **hints):
        return random.choice(self.SLAVE_DATABASES)
```
按条件路由（如特定Model强制读取主库）
```python
class CriticalModelRouter:
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'payment':  # 关键业务模型
            return 'default'
        return 'slave'
```

Django 默认事务使用主库，跨库操作需手动管理：
```python
from django.db import transaction

with transaction.atomic(using='default'):
    # 主库写入
    user = User.objects.using('default').create(name="Bob")
    # 主库读取（确保一致性）
    user = User.objects.using('default').get(id=user.id)
```

迁移操作必须指定主库
```shell
python manage.py makemigrations
python manage.py migrate --database=default
```
# Mixin
