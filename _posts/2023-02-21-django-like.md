---
title: DRF - 게시물 좋아요 기능 최적화
date: 2023-02-21
categories: ["PYTHO", "DJANGO"]
tags: ['python', 'django', 'restframework']     # TAG names should always be lowercase
toc: true
---
## 들어가며

게시물 모델과 유저 모델이 존재하는데, 게시물에 유저가 좋아요를 하는 기능을 추가하려고 합니다. 좋아요를 누르는 동작은 간단히 구현을 할 수 있지만, 유저가 게시물 리스트를 요청 할 때, 각 게시물에 대하여 해당 유저가 좋아요를 눌렀는지 판단 여부를 판단 할 수 있는 코드를 구현해보고, 이를 최적화하는 과정에 대해서 설명하도록 하겠습니다.

## Model

```python
# 유저 모델
from django.db import models

class User(models.Model):
    name = models.CharField(default='', max_length=10)

    class Meta:
        db_table = 'user'
```
```python
# 게시물 모델
from django.db import models

class UserPostLike(models.Model):
    user = models.ForeignKey('user.User', on_delete=models.CASCADE)
    post = models.ForeignKey('Post', on_delete=models.CASCADE)
    like_at = models.DateTimeField(auto_now_add=True)


class Post(models.Model):
    title = models.CharField(max_length=100, blank=True, default='')
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    like_users = models.ManyToManyField('user.User', through='UserPostLike')

    class Meta:
        db_table = 'post'
```
```shell
# 데이터 생성
# 게시물 두개, 유저 한명 생성 후 첫번째 게시물 좋아요 목록에 유저 추가
>>> Post.objects.create(title='첫번째 글')
>>> Post.objects.create(title='두번째 글')
>>> User.objects.create(name='kvngho')
>>> user = User.objects.first()
>>> post = Post.objects.first()
>>> post.like_users.add(user)
```
Post 모델과 User 모델을 UserPostLike라는 중간테이블을 통하여 다대다관계로 연결하고, 유저가 좋아요를 누르면 해당 Post의 like_users에 해당 유저를 add() 한다면, 좋아요 버튼 기능은 끝입니다. 그렇다면 Post 리스트에서 유저의 좋아요 여부는 어떻게 판단 할 수 있을까요?

## View, Serializer
기본적인 전체 post를 조회 할 수 있는 view와 serializer의 코드는 다음과 같습니다.:
```python
# Post serializer
from rest_framework import serializers
from .models import Post

class PostSerializer(serializers.ModelSerializer):

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'created_at']
```
```python
# Post View
from rest_framework.generics import ListAPIView
from rest_framework.permissions import AllowAny
from .models import Post
from .serializers import PostSerializer

class PostListAPIView(ListAPIView):
    serializer_class = PostSerializer
    queryset = Post.objects.all()
    permission_classes = (AllowAny, )
```

이제 여기서 좋아요 여부를 판단 할 수 있는 필드를 `like_status` 라는 이름으로 추가해보도록 하겠습니다.

## SerializerMethodField
가장 먼저 빠르고 쉽게 생각 할 수 있는 방법은 drf에서 제공하는 serializerMethodField를 사용하는 것입니다. 이를 이용하면 직관적이게 좋아요 여부 필드를 추가 할 수 있습니다:
```python
# Post serializer
class PostSerializer(serializers.ModelSerializer):
    like_status = serializers.SerializerMethodField()

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'like_status', 'created_at']

    def get_like_status(self, obj):
        return obj.like_users.filter(id=self.context['request'].user.id).exists()
```
like_status를 `serializers.SerializerMethodField()`로 선언해줍니다. 이때 메소드의 파라미터에는 메서드의 이름이 들어가며, 따로 파라미터를 지정하지 않으면 `get_필드명`의 메서드로 지정합니다.
serializer는 like_status라는 필드를 직렬화하기 위하여 `get_like_status` 메서드를 실행시킵니다. 해당 메소드는 요청을 한 유저의 id가 해당 post의 좋아요한 유저 id중에 존재하는지를 판별 한 후, 존재한다면 true를, 아니면 false를 반환합니다.

```json
{
    "count": 2,
    "next": null,
    "previous": null,
    "results": [
        {
            "id": 1,
            "title": "첫번째 글",
            "content": "",
            "like_status": true,
            "created_at": "2023-02-21 06:32:17"
        },
        {
            "id": 2,
            "title": "두번째 글",
            "content": "",
            "like_status": false,
            "created_at": "2023-02-21 06:32:37"
        }
    ]
}
```
API 요청을 해보면, 생각대로 맞게 반환하고 있습니다. 하지만 뭔가 찝찝합니다.

### `get_like_status`
해당 함수는 모든 post들에 대해서 실행되고, 해당 함수 내부에서 추가적인 DB 조회를 발생시킵니다.
debug-toolbar를 통해 살펴보면,
```sql
SELECT "post"."id",
       "post"."title",
       "post"."content",
       "post"."created_at"
  FROM "post"
 LIMIT 2

 SELECT '1' AS "a"
  FROM "user"
 INNER JOIN "post_userpostlike"
    ON ("user"."id" = "post_userpostlike"."user_id")
 WHERE ("post_userpostlike"."post_id" = '1' AND "user"."id" = '1')
 LIMIT 1

 SELECT '1' AS "a"
  FROM "user"
 INNER JOIN "post_userpostlike"
    ON ("user"."id" = "post_userpostlike"."user_id")
 WHERE ("post_userpostlike"."post_id" = '2' AND "user"."id" = '1')
 LIMIT 1
```
밑의 2개의 쿼리를 보면 알 수 있듯이, 모든 post에 대해서 좋아요 여부를 판단하는 쿼리를 DB에 hit하고 있는것을 확인 할 수 있습니다. 지금은 게시물이 두개뿐이지만, n개의 게시물이 있다면 n번의 추가 쿼리가 DB에 hit 하게 되고, 이는 성능저하로 이루어 질 수 있습니다.

## Annotate를 이용한 최적화
이러한 N+1 problem을 해결하기 위하여 보통 `prefetch_related`나 `selected_related`를 많이 사용하는데, 그 자체로는 "좋아요 여부"와 같은 Boolean 값의 조건부 로직을 처리하지 못합니다. 좋아요 여부 같은 로직을 처리하기 위하여 Annotate를 사용한 로직은 다음과 같습니다:
```python
# Post view

class PostListAPIView(ListAPIView):
    serializer_class = PostSerializer
    queryset = Post.objects.all()
    permission_classes = (AllowAny, )

    def get_queryset(self):
        queryset = Post.objects.annotate(
            like_status=Case(
                When(like_users__id=self.request.user.id, then=True),
                default=False,
                output_field=BooleanField()
            )
        )
        return queryset
```
```python
# Post Serializer
class PostSerializer(serializers.ModelSerializer):
    like_status = serializers.BooleanField()

    class Meta:
        model = Post
        fields = ['id', 'title', 'content', 'like_status', 'created_at']
```
애초에 serializer로 queryset을 넘겨줄 때 object manager의 annotate 함수를 이용해서, like_status가 포함된 queryset을 넘겨주게 된다면, 추가적인 db hit 없이 좋아요 여부를 판단 할 수 있게 됩니다. 이렇게 코드를 작성하면 위의 serializerMethodField를 이용한 결과와 같은 결과를 반환할 수 있습니다. 실행된 쿼리는 다음과 같습니다.
```sql
SELECT "post"."id",
       "post"."title",
       "post"."content",
       "post"."created_at",
       CASE WHEN "post_userpostlike"."user_id" = '1' THEN 'True'
            ELSE 'False'
             END AS "like_status"
  FROM "post"
  LEFT OUTER JOIN "post_userpostlike"
    ON ("post"."id" = "post_userpostlike"."post_id")
 LIMIT 2
```
이렇게 한번의 DB hit만 발생하게됩니다.

## 마치며
무엇이 옳은 코드인가에 대한 무조건 적인 정답은 없는것 같습니다. 실제로 serializerMethodField를 이용한 방법이 annotate를 이용한 방법보다 본 게시글 예제에서는 더 빠른 응답속도를 보였습니다. 하지만 이는 게시글의 갯수가 2개 밖에 없어서 그런것이고, 게시물의 갯수가 더 증가한다면 아무래도 DB hit의 횟수가 적은 annotate를 이용한 방법이 좀 더 적은 응답속도를 얻을 수 있을것이라고 기대됩니다.

## References
[1] [django restframework](https://github.com/encode/django-rest-framework.git)