# 개발 시 스타일 가이드

[Django 모델(ORM)](https://docs.djangoproject.com/en/4.0/topics/db/models/) 은 Active Record 패턴으로 구현되어 있습니다. 즉 Django 모델은 데이터를 영속하는 기능도 함께 가지고 있습니다. 또한 [DRF Serializer](https://www.django-rest-framework.org/api-guide/serializers/)에서는 종종 데이터에 대한 검증과 생성 로직을 포함하기도 합니다.

두 프레임워크를 사용하는 상황에서 주의하지 않으면, 비즈니스 로직이 여러 군데 흩어져 스파게티 코드가 될 가능성이 높습니다.

이 스타일 가이드는 Django의 코딩 컨벤션을 정의하여 응집된 비즈니스 로직을 가진 소프트웨어를 작성하도록 돕는 데 목적이 있습니다.

따라서 [Django Style Guide](https://github.com/HackSoftware/Django-Styleguide#approach-2---hacksofts-proposed-way) 를 참고해서 저만의 style guide를 만들었습니다.

---

## 목차

- [개요](#개요)
- [모델](#모델)
- [서비스](#서비스)
- [APIs & Serializers](#apis--serializers)
- [Errors & Exception Handling](#errors--exception-handling)
- [Testing](#testing)

---

## 개요

**비즈니스 로직이 있어야 하는 곳**

- Services - 데이터베이스에 대한 쓰기 작업을 관리하는 곳. (Command)
- Selectors - 데이터베이스에 대한 읽기 작업을 관리하는 곳. (Query)
- Model properties (with some exceptions).
- Model의 추가적인 validations를 위한 clean 함수.

**비즈니스 로직이 있으면 안되는 곳**

- APIs 와 Views.
- Serializers and Forms.
- Form tags.
- Model의 save 함수 (절대 override해서 기능을 구현하면 안됩니다.)
- Custom 매니저 혹은 쿼리셋
- Signals.

**모델의 properties vs selectors**

- 만약 property가 여러 relation에 걸쳐있다면, selector를 사용합니다.
- 만약 property가 no trivial하거나 n+1 문제를 야기한다면, selector를 사용합니다.

---

## 모델(Models)

Django 모델은 데이터 모델에만 신경써야 합니다.

**Base Model**

여러 엔티티에 걸쳐 공통된 필드나 기능이 필요할 때는 BaseModel을 정의하여 상속해서 사용할 수 있습니다.

```python
from django.db import models
from django.utils import timezone


class BaseModel(models.Model):
    created_at = models.DateTimeField(db_index=True, default=timezone.now)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

위 모델을 상속하여 개발할 수 있습니다.

```python
class SomeModel(BaseModel):
    pass
```

**Validation - _clean_ and _full_clean_**

다음과 같은 모델이 있다고 가정해봅시다.

```python
class Course(BaseModel):
    name = models.CharField(unique=True, max_length=255)

    start_date = models.DateField()
    end_date = models.DateField()

    def clean(self):
        if self.start_date >= self.end_date:
            raise ValidationError("End date cannot be before start date")
```

모델에 _clean_ method를 정의하여, 모델이 데이터베이스에 삽입되기 전에 검증할 수 있습니다.
_clean_ method가 호출되기 위해서, 누군가는 모델 인스턴스의 _full_clean_ method를 save() 전에 호출해야합니다.
Django 모델에서 정의하는 _clean_ , _full_clean_ method를 더 알고 싶다면 다음 [문서](https://docs.djangoproject.com/en/4.0/ref/models/instances/#validating-objects)를 참고해주세요.

이 clean method를 호출하는 곳은 service 가 되어야 합니다. 다음의 예시를 참고해주세요.

```python
def course_create(*, name: str, start_date: date, end_date: date) -> Course:
    obj = Course(name=name, start_date=start_date, end_date=end_date)

    obj.full_clean()
    obj.save()
    return obj
```

`course_create` 함수는 service에 해당합니다. service단에서 모델 객체 저장하기 전 호출함으로 데이터에 대한 검증을 할 수 있습니다.

물론, DRF에서 제공하는 serializer를 커스텀하면, 데이터에 대한 검증이 가능합니다. 다만 입력 데이터에 대한 검증은 도메인 로직과 관련되어 있을 가능성이 높습니다. 따라서 일관되게 위와 같은 방법으로 데이터에 대한 유효성을 검증합니다.

**Properties**

모델의 property들은 모델의 instance의 값을 얻을 수 있는 가장 빠르고 좋은 방법입니다.

다음의 예시를 보시죠.

```python
from django.utils import timezone
from django.core.exceptions import ValidationError


class Course(BaseModel):
    name = models.CharField(unique=True, max_length=255)

    start_date = models.DateField()
    end_date = models.DateField()

    def clean(self):
        if self.start_date >= self.end_date:
            raise ValidationError("End date cannot be before start date")

    @property
    def has_started(self) -> bool:
        now = timezone.now()

        return self.start_date <= now.date()

    @property
    def has_finished(self) -> bool:
        now = timezone.now()

        return self.end_date <= now.date()
```

_has_started_, _has_finished_ 와 같은 property들은 자주 사용되는 특성이며 어디서나 사용될 수 있습니다. 또한 코드의 가독성 측면에서도 유리합니다.

service 에서 모델에 대한 조건 분기 처리를 한다고 생각해보죠.

```python
    # property 사용 X
    now = timezone.now()
    if course.end_date <= now.date():

    # property 사용 O
    if course.has_finished():
```

훨씬 깔끔하고, 다른 동료 개발자가 코드를 이해하기도 더 쉬워졌습니다.

다만 property와 관련해서 다음과 같은 경우는 service, selector, utility 를 사용하는 것이 더 낫습니다.

- 여러 관계를 확장하거나 추가 데이터를 가져와야 하는 경우
- 계산이 복잡한 경우

**Methods**

모델에 method를 정의하는 것은 매우 강력합니다. 여러 property를 활용해서 작성할 수 있습니다.

다음의 코드의 _is_within(self, x)_ method를 보시죠.

```python
from django.core.exceptions import ValidationError
from django.utils import timezone


class Course(BaseModel):
    name = models.CharField(unique=True, max_length=255)

    start_date = models.DateField()
    end_date = models.DateField()

    def clean(self):
        if self.start_date >= self.end_date:
            raise ValidationError("End date cannot be before start date")

    @property
    def has_started(self) -> bool:
        now = timezone.now()

        return self.start_date <= now.date()

    @property
    def has_finished(self) -> bool:
        now = timezone.now()

        return self.end_date <= now.date()

    def is_within(self, x: date) -> bool:
        return self.start_date <= x <= self.end_date
```

_is_within_ 은 인자가 필요하기 때문에 property가 될 수 없습니다. 대신 method를 사용합니다.

사실 모델의 method를 정의하는데 있어서 가장 유용한 use case는 모델의 상태를 변경할 때 입니다. 즉 모델이 수행하는 실질적인 비즈니스 로직을 표현하는 데 유용합니다.

다음 코드 예시를 보시죠.

```python
from django.utils.crypto import get_random_string
from django.conf import settings
from django.utils import timezone


class Token(BaseModel):
    secret = models.CharField(max_length=255, unique=True)
    expiry = models.DateTimeField(blank=True, null=True)

    def set_new_secret(self):
        now = timezone.now()

        self.secret = get_random_string(255)
        self.expiry = now + settings.TOKEN_EXPIRY_TIMEDELTA

        return self
```

Token 모델에서 Token을 생성하기 위해 *set_new_secret()*함수를 호출함으로써 올바른 secret과 expiry를 얻을 수 있습니다.

위와 같은 방법을 사용하는 경우는 다음과 같습니다.

1. 간단하게 값을 얻을 수 있고, 인자가 간단하며, non-relational model field일 경우
2. 얻어진 값을 계산하는 로직이 간단할 경우
3. 하나의 attribute를 설정하는 것이 항상 다른 attribute 값을 결정하는 경우.

모델 말고 다른 service, selector, utility로 로직을 옮겨야 하는 경우는 다음과 같습니다.

1. 여러 relation들이 필요하거나, 추가적인 데이터를 가져올 때
2. 계산 로직이 복잡할 경우

**Testing**
어떤 모델이 추가되면, 테스트 코드가 작성되어야 합니다.

```python
from datetime import timedelta

from django.test import TestCase
from django.core.exceptions import ValidationError
from django.utils import timezone

from project.some_app.models import Course


class CourseTests(TestCase):
    def test_course_end_date_cannot_be_before_start_date(self):
        start_date = timezone.now()
        end_date = timezone.now() - timedelta(days=1)

        course = Course(start_date=start_date, end_date=end_date)

        with self.assertRaises(ValidationError):
            course.full_clean()
```

위 경우에서 full_clean()을 호출했을 떄 validation error가 일어나는지 검증합니다. 이 때 이 테스트는 DB를 hit할 필요가 전혀 없습니다. 단지 모델에 대한 순수 로직만 검증해야합니다.

---

## 서비스 (services)

서비스는 클라이언트가 요청한 기능을 실행합니다. 모델 객체를 가져와서, 모델 객체의 비즈니스 로직을 실행하고, 결과를 리턴하도록 합니다.

경우에 따라서 비즈니스 로직을 수행하는데 모델 객체의 로직으로 충분하지 않을 수 있습니다.

서비스는 다음과 같이 작성될 수 있습니다.

- 간단한 function
- 클래스
- 전체 모듈

**예시 - 함수 기반 service**

사용자를 생성하는 예시 service입니다.

```python
def user_create(
    *,
    email: str,
    name: str
) -> User:
    user = User(email=email)
    user.full_clean()
    user.save()

    profile_create(user=user, name=name)
    confirmation_email_send(user=user)

    return user
```

여기서 profile_create와 confirmation_email_send와 같은 다른 service를 호출합니다. 이 예시에서 사용자의 생성과 관련된 모든 로직은 하나의 장소에서 일어나며 추적이 가능하기 떄문에 코드의 응집도가 올라갑니다.

**예시 - 클래스 기반 service**

```python
class FileStandardUploadService:
    """
    This also serves as an example of a service class,
    which encapsulates 2 different behaviors (create & update) under a namespace.

    Meaning, we use the class here for:

    1. The namespace
    2. The ability to reuse `_infer_file_name_and_type` (which can also be an util)
    """
    def __init__(self, user: BaseUser, file_obj):
        self.user = user
        self.file_obj = file_obj

    def _infer_file_name_and_type(self, file_name: str = "", file_type: str = "") -> Tuple[str, str]:
        if not file_name:
            file_name = self.file_obj.name

        if not file_type:
            guessed_file_type, encoding = mimetypes.guess_type(file_name)

            if guessed_file_type is None:
                file_type = ""
            else:
                file_type = guessed_file_type

        return file_name, file_type

    @transaction.atomic
    def create(self, file_name: str = "", file_type: str = "") -> File:
        _validate_file_size(self.file_obj)

        file_name, file_type = self._infer_file_name_and_type(file_name, file_type)

        obj = File(
            file=self.file_obj,
            original_file_name=file_name,
            file_name=file_generate_name(file_name),
            file_type=file_type,
            uploaded_by=self.user,
            upload_finished_at=timezone.now()
        )

        obj.full_clean()
        obj.save()

        return obj

    @transaction.atomic
    def update(self, file: File, file_name: str = "", file_type: str = "") -> File:
        _validate_file_size(self.file_obj)

        file_name, file_type = self._infer_file_name_and_type(file_name, file_type)

        file.file = self.file_obj
        file.original_file_name = file_name
        file.file_name = file_generate_name(file_name)
        file.file_type = file_type
        file.uploaded_by = self.user
        file.upload_finished_at = timezone.now()

        file.full_clean()
        file.save()

        return file
```

service는 다음과 같이 view에서 사용될 수 있습니다.

```python
# https://github.com/HackSoftware/Django-Styleguide-Example/blob/master/styleguide_example/files/apis.py

class FileDirectUploadApi(ApiAuthMixin, APIView):
    def post(self, request):
        service = FileDirectUploadService(
            user=request.user,
            file_obj=request.FILES["file"]
        )
        file = service.create()

        return Response(data={"id": file.id}, status=status.HTTP_201_CREATED)
```

**Modules**

service가 커지면 다음과 같이 모듈화 시킬 수 있습니다.

```
services/
    __init__.py
    jwt.py
    oauth.py
```

### 셀렉터(selectors)

CQRS(Command and Query Responsibility Segregation) 에서 영감을 얻어 모델을 생성하거나 업데이트 하는 등의 영속성 레이어를 거치는 부분과, 모델을 조회하는 부분을 나눕니다.

셀렉터는 service와 마찬가지로 함수, 클래스, 모듈 등으로 구현될 수 있습니다.

### Testing

service 및 selector는 비즈니스 로직을 가지고 있습니다. 따라서 반드시 테스트의 대상이 되어야 합니다.

테스트를 작성할 때 다음과 같은 원칙을 따릅니다.

1. 테스트는 비즈니스 로직을 충분히 검증해야 합니다.
2. 테스트는 데이터베이스를 바탕으로 이루어져야 합니다.
3. 비동기 태스크의 호출 혹은 모든 외부 시스템과의 연동을 mocking하여 테스트 해야합니다.

- 테스트는 주로 given-when-then (준비 - 실행 - 검증) 포맷으로 작성하면 좋습니다.

---

## APIs & Serializers

services & selectors를 사용할 때 모든 APIs는 보기에 간단하고 동일해야 합니다.

API를 개발할 때 다음과 같은 규칙을 따릅니다.

- 하나의 Operation에는 하나의 API만을 개발합니다.
- 가장 간단한 [APIView](https://www.django-rest-framework.org/api-guide/views/) 혹은 [GenericAPIView](https://www.django-rest-framework.org/api-guide/generic-views/)를 상속하여 개발합니다.

  - 더 추상화된 클래스를 사용하지 않는 이유는, 모든 것을 다 serializer를 통해 수행하려고 하기 때문입니다.
  - service, selector를 통해 해당 역할(비즈니스 로직)을 수행하도록 convention을 두었기 때문에 추상화된 클래스(ex. CreateAPIView, ListAPIView etc.)들은 사용하지 않습니다.
  - 다만 아주 간단한 CRUD를 개발하는 경우는 추상화된 클래스를 활용할 수 있습니다.

- 비즈니스 로직이 API 단에 있으면 안됩니다.
- 핵심은, API의 코드를 단순하게 유지하는 것입니다. API단에서는 단순히 서비스를 불러와 호출 후 결과를 돌려주는 것으로 충분합니다.

그리고 API Serialization에 대한 규칙은 다음과 같습니다 :

- input serializer와 output serializer가 명시되어야 합니다.

만약 [DRF's serializer](https://www.django-rest-framework.org/tutorial/1-serialization/)를 사용한다면 다음의 규칙을 따릅니다.

- Serializer 는 API 클래스에 inner class로 정의되어야 하며, InputSerializer 혹은 OutputSerializer라는 이름을 활용하여 네이밍 되어야 합니다.
- 모든 serializer는 가장 간단한 Serializer(from rest_framework.serializers import Serializer)를 활용하도록 합니다. (ModelSerializer X)
- 대부분의 경우 Serializer의 재사용을 피합니다. -> 예상하지 못한 결과를 종종 야기하기 때문입니다. 그러나 경우에 따라서 Nested Serializer 가 필요할 때도 있고 그 Nested된 serializer가 여러 곳에서 재사용 되어야 하는 경우도 존재합니다. 이럴 때는 serializers.py 파일에 정리하여 참조합니다.

### Naming Convention

API를 작성할 때 다음과 같은 Naming Convetion을 따릅니다. `<Entity><Actions ...>`Api

### Class 기반 vs Function 기반

되도록이면 Class 기반으로 작성하도록 합니다. 그 이유는 상속받은 View에서 많은 기능들을 제공하기 때문입니다. Django를 사용하는 이유 중 하나가 생산성이기 때문에 최대한 프레임워크 & 라이브러리에서 제공하는 것을 활용하도록 합니다.

API 작성의 간단한 예시는 다음과 같습니다.

```python

from rest_framework.views import APIView
from rest_framework import serializers
from rest_framework.response import Response

from styleguide_example.users.selectors import user_list
from styleguide_example.users.models import BaseUser


class UserListApi(APIView):
    class OutputSerializer(serializers.Serializer):
        id = serializers.CharField()
        email = serializers.CharField()

    def get(self, request):
        users = user_list()

        data = self.OutputSerializer(users, many=True).data

        return Response(data)
```

API를 개발하는데 있어서 여러 예시를 보고싶다면 다음 [레퍼런스](https://github.com/HackSoftware/Django-Styleguide/blob/master/README.md#list-apis)를 참고하세요.

### URL 설계 규칙

기본적인 [REST API](https://meetup.toast.com/posts/92) 규칙에 기반하여 설계합니다.

다만 특정 비즈니스 로직이 단순 CRUD로 표현하기가 어려울 경우 다음과 같이 action을 url 끝에 붙이는 방식으로 설계합니다.

```
/api/payment/1/complete - PATCH
```

## Errors & Exception Handling

[참조](https://github.com/HackSoftware/Django-Styleguide/blob/master/README.md#approach-2---hacksofts-proposed-way) 에 있는 방법을 따릅니다.

## Testing

Testing에 대해서 다음과 같이 파일 구조를 가지도록 합니다.

```
project_name
├── app_name
│   ├── __init__.py
│   └── tests
│       ├── __init__.py
│       ├── models
│       │   └── __init__.py
│       │   └── test_some_model_name.py
│       ├── selectors
│       │   └── __init__.py
│       │   └── test_some_selector_name.py
│       └── services
│           ├── __init__.py
│           └── test_some_service_name.py
└── __init__.py
```

### Naming conventions

테스트 파일의 이름은 test_the_name_of_the_thing_that_is_tested.py 로 만듦니다.
테스트 케이스는 class TheNameOfTheThingThatIsTestedTests(Testcase): 로 설정합니다.

예를 들어,

```python
def a_very_neat_service(*args, **kwargs):
    pass
```

위 함수를 테스트하는 테스트 코드를 짠다고 하면,

파일 이름은 project_name/app_name/tests/services/test_a_very_neat_service.py 로 만듦니다.

```python
class AVeryNeatServiceTests(TestCase):
    pass
```

파일 안에는 위와 같이 클래스를 만들어 테스트 코드를 구성합니다.

### Celery 활용

사용자 Request에서 시간이 오래 걸리는 작업을 수행하면, API의 성능도 떨어지고 서버 전체의 처리량에 영향을 미칩니다.

다음의 경우가 존재합니다.

- 써드 파티 서비스와의 통신 (메일, 푸시 알림 보내기 등)
- HTTP 사이클 밖에서 실행되어야 하는 무거운 계산 작업
- 주기적으로 실행되어야 하는 Job (Celery beat)

이런 경우 [Celery](https://docs.celeryq.dev/en/stable/) 를 적극적으로 활용하면 좋습니다.
