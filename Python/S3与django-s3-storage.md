## **ARN 的通用格式**

`arn:partition:service:region:account-id:resource`

- `arn`：固定前缀
- `partition`：AWS 分区，例如 `aws`（标准 AWS 区域）、`aws-cn`（中国）、`aws-us-gov`（政府云）
- `service`：资源所属服务，例如 `s3`, `iam`, `lambda`
- `region`：区域，例如 `us-east-1`（有些服务忽略）
- `account-id`：AWS 账号 ID（有些服务忽略）
- `resource`：具体资源，可以是名称、路径或 ID



# django-s3-storage

```
[pip]
pip install django-s3-storage

[uv]
uv add django-s3-storage
```

在`INSTALLED_APPS`中添加`django_s3_storage`

下面有两种配置方式，如果所有的文件都放S3，则可以在配置文件中直接配置，否则，如果有多种S3配置，则自定义`S3Storage`实例，并在文件字段自行配置
```python
DEFAULT_FILE_STORAGE = "django_s3_storage.storage.S3Storage"
```

## 提供的配置项

**认证和连接配置**
这些设置用于建立与 AWS 的连接和会话。

| **配置项**                     | **默认值**       | **解释**                                              |
| --------------------------- | ------------- | --------------------------------------------------- |
| **`AWS_REGION`**            | `"us-east-1"` | S3 存储桶所在的 AWS 区域名称。                                 |
| **`AWS_ACCESS_KEY_ID`**     | `""`          | AWS 访问密钥 ID。如果为空，`boto3` 将尝试使用 IAM 角色或本地配置文件进行身份验证。 |
| **`AWS_SECRET_ACCESS_KEY`** | `""`          | AWS 秘密访问密钥。                                         |
| **`AWS_SESSION_TOKEN`**     | `""`          | 如果使用临时安全凭证（如通过 IAM 角色或 MFA 获取），则需要提供此令牌。            |

**S3 操作和行为配置**
这些设置控制文件如何存储、访问以及 S3 客户端的行为。

| **配置项**                        | **默认值**  | **解释**                                                            |
| ------------------------------ | -------- | ----------------------------------------------------------------- |
| **`AWS_S3_BUCKET_NAME`**       | `""`     | **必需项。** 存储文件的 S3 存储桶名称。                                          |
| **`AWS_S3_ENDPOINT_URL`**      | `""`     | 如果使用兼容 S3 的服务（如 MinIO、Ceph）或 AWS 私有 Link，则指定自定义的 S3 **端点 URL**。   |
| **`AWS_S3_KEY_PREFIX`**        | `""`     | 文件在 S3 存储桶中存储的**前缀**（相当于一个虚拟目录路径）。所有文件名都会以此前缀开始。                  |
| **`AWS_S3_ADDRESSING_STYLE`**  | `"auto"` | S3 客户端用于访问存储桶的寻址方式（如 `virtual` 或 `path` 样式）。`auto` 会让客户端自动选择最佳方式。 |
| **`AWS_S3_SIGNATURE_VERSION`** | `"s3v4"` | 用于对 S3 请求进行签名（身份验证）的版本。`s3v4` 是目前推荐的版本。                           |


**访问和 URL**

| **配置项**                      | **默认值**       | **解释**                                                                                                                                          |
| ---------------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **`AWS_S3_BUCKET_AUTH`**     | `True`        | 是否使用**预签名 URL**（Presigned URLs）进行访问。<br>* 如果为 `True`，文件默认为 **`private`**，URL 将包含签名参数和过期时间。<br>* 如果为 `False`，文件默认为 **`public-read`**，URL 将不包含签名。 |
| **`AWS_S3_PUBLIC_URL`**      | `""`          | 一个公共的基础 URL。如果设置，`url()` 方法将直接使用此 URL 加上文件名来构造访问路径，并且 **不使用** 预签名。这常用于通过 CloudFront 或其他 CDN 访问文件。                                               |
| **`AWS_S3_MAX_AGE_SECONDS`** | `3600` (1 小时) | 用于设置 HTTP 响应中的 **`Cache-Control`** 头部，控制浏览器和 CDN 缓存文件的时间（单位：秒）。也用于设置预签名 URL 的过期时间。                                                              |
**存储和元数据**

| **配置项**                          | **默认值** | **解释**                                                                                    |
| -------------------------------- | ------- | ----------------------------------------------------------------------------------------- |
| **`AWS_S3_REDUCED_REDUNDANCY`**  | `False` | 是否使用 S3 的 **`REDUCED_REDUNDANCY`** 存储类别（现在通常推荐使用标准或 IA 等）。如果为 `False`，则使用 **`STANDARD`**。 |
| **`AWS_S3_METADATA`**            | `{}`    | 一个字典，用于指定在文件上传时添加到 S3 对象的自定义用户元数据 (`x-amz-meta-*`)。                                       |
| **`AWS_S3_CONTENT_DISPOSITION`** | `""`    | 设置 HTTP 响应的 **`Content-Disposition`** 头部。可用于强制浏览器下载文件或指定下载文件名。                            |
| **`AWS_S3_CONTENT_LANGUAGE`**    | `""`    | 设置 HTTP 响应的 **`Content-Language`** 头部。                                                    |

**性能和安全**

|**配置项**|**默认值**|**解释**|
|---|---|---|
|**`AWS_S3_GZIP`**|`True`|是否对可压缩文件（如文本、CSS、JS、HTML、JSON 等）进行 **Gzip 压缩**并上传。压缩后会设置 **`Content-Encoding: gzip`** 头部。|
|**`AWS_S3_ENCRYPT_KEY`**|`False`|是否对 S3 上的文件进行**服务器端加密**（Server-Side Encryption）。<br><br>  <br><br>* 如果为 `True`，默认使用 `AES256`。<br><br>  <br><br>* 如果传入字符串（例如 `"aws:kms"`），则使用指定加密类型。|
|**`AWS_S3_KMS_ENCRYPTION_KEY_ID`**|`""`|如果 `AWS_S3_ENCRYPT_KEY` 设置为 KMS 加密，则指定要使用的 **KMS 密钥 ID**。|
|**`AWS_S3_FILE_OVERWRITE`**|`False`|当上传同名文件时，是否允许 **覆盖** 现有文件。如果为 `False`，存储后端将使用 Django 的机制生成一个新的、可用的文件名。|
|**`AWS_S3_USE_THREADS`**|`True`|在使用 `boto3` 上传和下载大文件时，是否启用**多线程**以提高性能。|


**静态文件存储特有配置**

| **配置项**                             | **默认值**          | **解释**                                                                                                       |
| ----------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------ |
| **`AWS_S3_MAX_AGE_SECONDS_CACHED`** | `31536000` (1 年) | 针对**带哈希值**（即经过 Django ManifestFilesMixin 处理）的静态文件设置的 **`Cache-Control`** 最大过期时间。因为这些文件的内容永远不会改变，所以可以设置为长久缓存。 |


通常只需要配置：
```python
AWS_ACCESS_KEY_ID = secrets.get("AWS_S3_ACCESS_KEY_ID")
AWS_SECRET_ACCESS_KEY = secrets.get("AWS_S3_SECRET_ACCESS_KEY")
AWS_REGION = "ap-southeast-1"
AWS_S3_BUCKET_NAME = "xxx"
AWS_S3_PUBLIC_URL = "https://xxx/"
AWS_S3_FILE_OVERWRITE = True
```


## 多套配置
在Django ＜ 4.2时如果需要设置多套不同的配置，则可以使用`s3_settings_suffix`，并实例化多个`Storage`例如：
```python
from django_s3_storage.storage import S3Storage

# 默认媒体文件存储 (使用空的后缀)
# class DefaultMediaStorage(S3Storage):
#     s3_settings_suffix = ""

# 敏感用户文件存储
class PrivateUserStorage(S3Storage):
    # 后缀为 "_PRIVATE"
    s3_settings_suffix = "_PRIVATE" 

# 缓存文件存储
class CacheStorage(S3Storage):
    # 后缀为 "_CACHE"
    s3_settings_suffix = "_CACHE"
```

然后在`settings.py`中配置对应后缀的配置项：

|**原始配置项**|**PrivateUserStorage 查找的键**|**CacheStorage 查找的键**|
|---|---|---|
|`AWS_S3_BUCKET_NAME`|`AWS_S3_BUCKET_NAME_PRIVATE`|`AWS_S3_BUCKET_NAME_CACHE`|
|`AWS_S3_BUCKET_AUTH`|`AWS_S3_BUCKET_AUTH_PRIVATE`|`AWS_S3_BUCKET_AUTH_CACHE`|
|`AWS_REGION`|`AWS_REGION_PRIVATE`|`AWS_REGION_CACHE`|

```python
# settings.py

# --- 默认/公共媒体文件配置 (无后缀) ---
AWS_S3_BUCKET_NAME = "my-default-media-bucket"
AWS_S3_BUCKET_AUTH = False  # 公开访问
AWS_S3_PUBLIC_URL = "https://xxx/"

# --- 敏感用户文件配置 (_PRIVATE 后缀) ---
AWS_S3_BUCKET_NAME_PRIVATE = "my-private-user-files"
AWS_S3_BUCKET_AUTH_PRIVATE = True  # 私有，需要预签名 URL
AWS_REGION_PRIVATE = "ap-southeast-1" # 可以是不同的区域

# --- 缓存文件配置 (_CACHE 后缀) ---
AWS_S3_BUCKET_NAME_CACHE = "my-cache-storage-bucket"
AWS_S3_MAX_AGE_SECONDS_CACHE = 60 * 5 # 缓存时间短，5分钟
```

然后再模型类的文件字段中配置`storage`
```python
# models.py
from django.db import models
from your_app.storages import PrivateUserStorage, CacheStorage

class UserDocument(models.Model):
    # 使用 PrivateUserStorage 类
    document = models.FileField(storage=PrivateUserStorage())

class AppCache(models.Model):
    # 使用 CacheStorage 类
    data = models.FileField(storage=CacheStorage())
```

但是在Django ≥ 4.2之后，更建议使用`STORAGES`字典进行配置，Django 4.2 最主要的配置变化是引入了 `STORAGES` 字典，取代了 `DEFAULT_FILE_STORAGE` 和 `STATICFILES_STORAGE` 两个字符串设置。
配置文件如下（这里展示的是django-storage库，django-s3-storage不支持4.2版本）

```python
# settings.py

STORAGES = {
    # 默认文件存储（用于所有未明确指定 storage 的 FileField/ImageField）
    "default": {
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",
        "OPTIONS": {
            # 在这里可以设置特定于默认存储的参数
            "bucket_name": "my-app-media-bucket",
            # ... 其他选项 ...
        }
    },
    # 静态文件存储（通常也使用 django-storages）
    "staticfiles": {
        "BACKEND": "storages.backends.s3boto3.S3StaticStorage", 
    },
    
    # 定义一个自定义存储 'backups'
    "backups": {
        # 假设你使用的是 django-storages 的 S3 后端
        "BACKEND": "storages.backends.s3boto3.S3Boto3Storage", 
        "OPTIONS": {
            # 这是一个完全独立的配置，比如不同的 S3 桶
            "bucket_name": "my-offsite-backup-bucket",
            "location": "archive/",
            "querystring_auth": True, # 确保私有访问
            "default_acl": "private",
        },
    },
}

# 仍然需要设置全局认证信息 (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY 等)
```

```python
# models.py

from django.db import models
from django.conf import settings
from django.utils.module_loading import import_string
from django.core.exceptions import ImproperlyConfigured

# --- 存储实例化函数 ---
def get_custom_storage(storage_key):
    """从 settings.STORAGES 字典中加载并实例化指定的存储后端。"""
    
    storage_config = settings.STORAGES.get(storage_key)
    if not storage_config:
        raise ImproperlyConfigured(f"Storage '{storage_key}' not found in settings.STORAGES.")

    # 1. 导入存储类 (将字符串路径解析为类)
    StorageClass = import_string(storage_config['BACKEND'])
    
    # 2. 获取 OPTIONS 参数 (作为 kwargs 传入 __init__)
    options = storage_config.get('OPTIONS', {})
    
    # 3. 实例化存储对象
    return StorageClass(**options)

# 实例化自定义存储对象
BACKUP_STORAGE = get_custom_storage('backups')


# --- 模型定义 ---
class DatabaseBackup(models.Model):
    date = models.DateTimeField(auto_now_add=True)
    
    pic = models.FileField(
        upload_to='avatar/', 
    )  # <-- 不指定storage时会使用STORAGES的"default"配置
    
    # 将实例化后的存储对象赋值给 storage 参数
    backup_file = models.FileField(
        upload_to='daily_dumps/', 
        storage=BACKUP_STORAGE # <-- 这里指定了另一套配置
    )

    class Meta:
        verbose_name = "数据库备份"
```


## 访问文件
`S3Storage` 的 `url` 方法的目的是根据给定的文件路径 (`name`) 生成一个客户端可以用来访问该 S3 文件的 URL。它主要根据两个核心配置项的行为决定最终 URL 的形式：`AWS_S3_PUBLIC_URL` 和 `AWS_S3_BUCKET_AUTH`。

使用公共 URL (`AWS_S3_PUBLIC_URL`)
如果配置了 **`AWS_S3_PUBLIC_URL`**，该方法会优先使用它来生成 URL。
- **用途:** 通常用于文件通过 **CDN**（如 CloudFront）或其他代理服务器提供服务的情况，或当您不希望使用 AWS S3 默认的桶/路径格式时。
- **生成逻辑:**
	1. 它将 `AWS_S3_PUBLIC_URL` 作为基础 URL。
	2. 它将文件路径 `name` 通过 `filepath_to_uri` 进行编码。
	3. 它使用 `urljoin` 将这两部分连接起来。
	格式如：$AWS\_S3\_PUBLIC\_URL/file/path/to/name$
- **限制:** 如果使用了 `AWS_S3_PUBLIC_URL`，则**不允许**使用 `extra_params`。

在S3本身的逻辑中，文件的主域名和签名并没有强关联。但在`S3Storage`原生的方法中并不支持同时使用CloudFront和私有文件签名，因此一般可以选择重写 `url` 方法，如：
```python title:SignedCloudFrontS3Storage
import posixpath
from datetime import timedelta
from django.utils.timezone import now, make_aware, utc
from django.utils.deconstruct import deconstructible
from django.utils.encoding import filepath_to_uri, force_str

# 假设你的 CloudFrontSigner 和 S3Storage 已经正确导入
from your_app.storage import S3Storage
# from your_app.CloudFrontSigner import CloudFrontSigner # 假设签名类在这里

@deconstructible
class SignedCloudFrontS3Storage(S3Storage):
    """
    S3 Storage subclass that generates CloudFront Presigned URLs.
    """

    default_s3_settings = S3Storage.default_s3_settings.copy()
    default_s3_settings.update({
        # 替换原 CloudFront ID 配置项
        "AWS_S3_CLOUDFRONT_KEY_ID": "", 
        # 控制是否启用 CloudFront 签名模式
        "AWS_S3_CLOUDFRONT_SIGNING_URL": False, 
        # AWS_S3_PUBLIC_URL 和 AWS_S3_MAX_AGE_SECONDS 已存在于 S3Storage 的 default_s3_settings
        "AWS_S3_CLOUDFRONT_KEY_BYTES": "", 
        "AWS_S3_CLOUDFRONT_KEY_PASSWORD": "",
    })

    def _setup(self):
        super()._setup()

        # 检查是否满足 CloudFront 签名 URL 的条件
        self._use_cloudfront_signing = (
            self.settings.AWS_S3_PUBLIC_URL and  # 主域名
            self.settings.AWS_S3_CLOUDFRONT_KEY_ID and  # 签名需要用的ID
            self.settings.AWS_S3_CLOUDFRONT_SIGNING_URL  # 是否启用 CloudFront 签名模式
        )

    def _generate_cloudfront_presigned_url(self, name):
        """
        生成 CloudFront 预签名 URL，使用 CloudFrontSigner 的 generate_presigned_url 方法。
        """
        # 1. 构造基础 URL (使用 AWS_S3_PUBLIC_URL + 文件名)
        url_path = posixpath.join(self.settings.AWS_S3_PUBLIC_URL, filepath_to_uri(name))
        
        # 2. 计算过期时间 (使用 AWS_S3_MAX_AGE_SECONDS 作为签名的有效期)
        expires_in = self.settings.AWS_S3_MAX_AGE_SECONDS
        expiry_datetime = now() + timedelta(seconds=expires_in)
        
        # 确保 expiry 是 awareness datetime object
        if expiry_datetime.tzinfo is None or expiry_datetime.tzinfo.utcoffset(expiry_datetime) is None:
            expiry_datetime = make_aware(expiry_datetime, utc)

        # 3. 调用 CloudFrontSigner 的 generate_presigned_url 方法
        try:
	        key_id = self.settings.AWS_S3_CLOUDFRONT_KEY_ID 
	        key_bytes = self.settings.AWS_S3_CLOUDFRONT_KEY_BYTES 
	        key_password = self.settings.AWS_S3_CLOUDFRONT_KEY_PASSWORD
	        
            signed_url = CloudFrontSigner.generate_presigned_url(
                url=url_path, 
                expiry=expiry_datetime, 
                # 使用 S3Storage 的新配置项 AWS_S3_CLOUDFRONT_KEY_ID 
                cloudfront_id=key_id, 
                key_bytes=key_bytes, 
                key_password=key_password 
            )
            return signed_url
            
        except Exception as e:
            # 记录错误 (假设 log 已导入)
            # log.warning("Failed to generate CloudFront signed URL for %s: %s", name, e)
            # 如果签名失败，返回未签名的公共 URL（或抛出异常，取决于安全策略）
            return url_path 

    def url(self, name, extra_params=None):
        name = force_str(name)

        # 检查是否满足 CloudFront 签名 URL 的条件
        if self._use_cloudfront_signing:
            if extra_params:
                # 遵循 S3Storage 的约定，公共 URL 或自定义签名不允许 extra_params
                raise ValueError(
                    "Use of extra_params is not allowed with CloudFront Signed URLs."
                )
            return self._generate_cloudfront_presigned_url(name)
        
        # 回退到 S3Storage 默认的 URL 生成逻辑 (公共 URL 或 S3 预签名 URL)
        return super().url(name, extra_params)
```

其中工具类
```python title:CloudFrontSigner
import base64
import hashlib
import json
# boto
import boto3
from botocore.signers import CloudFrontSigner as BotoSigner

# cryptography
from cryptography.fernet import Fernet
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding


class CloudFrontSigner:
    """A class for genenrating CloudFront signed bookies.

    This class is based on https://gist.github.com/mjohnsullivan/31064b04707923f82484c54981e4749e

    Modified to use newer cryptography API and for convenient argument passing.

    Licensed under Apache License 2.0 (https://www.apache.org/licenses/LICENSE-2.0)

    Copyright (C) 2017 Matt Sullivan and Alvanon HK, Ltd.
    """

    @classmethod
    def rsa_sign(cls, message, key_bytes, key_password):
        private_key = serialization.load_pem_private_key(
            key_bytes,
            password=key_password,
            backend=default_backend(),
        )

        return private_key.sign(message, padding.PKCS1v15(), hashes.SHA1())

    @classmethod
    def sign(cls, message, key_bytes, key_password):
        return cls.rsa_sign(
            message, key_bytes, key_password
        )

    @classmethod
    def _replace_unsupported_chars(cls, policy_b64):
        return policy_b64.replace("+", "-").replace("=", "_").replace("/", "~")

    @classmethod
    def generate_policy(cls, url, expiry):
        """Returns the policy as a json string and as the json string encoded with base64."""
        policy = {
            "Statement": [
                {
                    "Resource": url,
                    "Condition": {
                        "DateLessThan": {"AWS:EpochTime": ceil(expiry.timestamp())},
                    },
                }
            ],
        }

        # Setting these explicit separators removes all whitespaces.
        policy_json = json.dumps(policy, separators=(",", ":"))

        policy_b64 = base64.b64encode(policy_json.encode()).decode()
        policy_b64 = cls._replace_unsupported_chars(policy_b64)

        return policy_json, policy_b64

    @classmethod
    def generate_signature(cls, policy_json, key_bytes, key_password):
        """Creates a signature for the policy from the key, returning a string"""
        sig_bytes = cls.rsa_sign(policy_json.encode(), key_bytes, key_password)
        return cls._replace_unsupported_chars(base64.b64encode(sig_bytes).decode())

    @classmethod
    def generate_cookies(cls, policy_b64, signature, cloudfront_id):
        """Returns a dictionary for cookie values in the form 'COOKIE NAME': 'COOKIE VALUE'."""
        return {
            "CloudFront-Policy": policy_b64,
            "CloudFront-Signature": signature,
            "CloudFront-Key-Pair-Id": cloudfront_id,
        }

    @classmethod
    def generate_signed_cookies(
        cls,
        url,
        expiry,
        key_bytes=settings.AWS_S3_CLOUDFRONT_KEY_BYTES,
        key_password=settings.AWS_S3_CLOUDFRONT_KEY_PASSWORD,
        cloudfront_id=settings.AWS_S3_CLOUDFRONT_KEY_ID,
    ):
        policy_json, policy_b64 = cls.generate_policy(url, expiry)
        signature = cls.generate_signature(policy_json, key_bytes, key_password)
        return cls.generate_cookies(policy_b64, signature, cloudfront_id)

    @classmethod
    def quote(cls, string):
        return quote(string, safe="~!*()-_'./")

    @classmethod
    def generate_presigned_url(
        cls, 
        url, 
        expiry, 
        cloudfront_id,
        key_bytes,
        key_password,
    ):
        signer = BotoSigner(cloudfront_id, cls.sign)

        return signer.generate_presigned_url(url, date_less_than=expiry)
```

其中的配置项有
```python title:settings
# settings.py

# 1. CloudFront 域名 (AWS_S3_PUBLIC_URL 必须设置)
AWS_S3_PUBLIC_URL = "https://cdn.example.com"

# 2. CloudFront Key ID (新添加的配置项)
# 注意：你需要确保你的 AWS_S3_CLOUDFRONT_KEY_ID, AWS_S3_CLOUDFRONT_KEY_BYTES, 
# 和 AWS_S3_CLOUDFRONT_KEY_PASSWORD 在 settings 中有定义，供 CloudFrontSigner 使用
AWS_S3_CLOUDFRONT_KEY_ID = "xxxxxxxxxxxxxx" 
AWS_S3_CLOUDFRONT_KEY_BYTES = xx # read pem file
AWS_S3_CLOUDFRONT_KEY_PASSWORD = None

# 3. 启用 CloudFront 签名模式 (新添加的配置项)
AWS_S3_CLOUDFRONT_SIGNING_URL = True 

# 4. S3 存储桶的默认缓存时间作为签名的有效期
AWS_S3_MAX_AGE_SECONDS = 3600 # 签名有效期为 1 小时

# 5. 配置您的媒体文件使用这个自定义存储类
DEFAULT_FILE_STORAGE = 'your_app.storages.SignedCloudFrontS3Storage'
```

使用：
```python
class CDNStorage(SignedCloudFrontS3Storage):
    s3_settings_suffix = "_CDN"
```

这个类将自动查找：
1. `AWS_S3_BUCKET_NAME_CDN`
2. `AWS_S3_CLOUDFRONT_KEY_ID_CDN`
3. `AWS_S3_CLOUDFRONT_SIGNING_URL_CDN`