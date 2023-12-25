---
title: 장고와 비즈니스 로직 - 개념
date: 2023-08-20
categories: ["PYTHON", "DJANGO"]
tags: ['python', 'django', 'software']     # TAG names should always be lowercase
toc: true
---

## 들어가며

장고 ORM을 통하여 데이터베이스의 자원에 동시에 접근 할 때, 프로그래머가 예상하는 시나리오와 다른 결과를 얻게 되는 문제가 발생 할 수 있습니다. 은행 송금/출금 예시와 함께 이러한 문제가 발생하는 이유와 그 해결책에 대해서 알아보도록 하겠습니다.

## 문제설명

다음과 같은 은행계좌 모델이 있습니다:
```python
class AccountBalance(models.Model):
    owner = models.ForeignKey('account.User', on_delete=models.CASCADE)
    balance = models.IntegerField(default=0)

    def deposit(self, amount):
        self.balance += amount
        self.save()

    def withdraw(self, amount):
        if self.balance >= amount:
            self.balance -= amount
            self.save()
        else:
            #raise error
            pass
```

서버는 출금, 송금 그리고 계좌잔액확인 각각의 api가 있다라는 가정하에 한 가지 상황을 생각해보도록 하겠습니다.

1. 유저 B의 계좌에는 10,000원이 있다.
2. 유저 A가 유저 B한테 송금 API를 호출하여 5,000원을 송금한다.
3. 유저 B는 7,000원을 출금한다.

이런 정상적인 상황에서 유저 B의 남은 잔액은 `10,000+5,000-7,000=8,000`
8,000원이 됩니다. 하지만 문제가 발생하여, 상황은 다음과 같이 변합니다:

1. 유저 B의 계좌에는 10,000원이 있다.
2. 유저 A가 유저 B한테 송금 API를 호출하여 5,000원을 송금한다.

    **-> 이 과정에서 5초의 딜레이가 발생한다.**

3. 딜레이가 해소되지 않은 상태에서 유저 B는 7,000원을 출금한다.

딜레이가 발생했지만, A가 송금한 사실과, B가 출금한 사실에는 변함이 없기 때문에 B의 남은 잔액은 똑같인 8,000원이 되어야합니다.

하지만 실제로 테스트를 해보면 장고 ORM은 생각과 다르게 동작합니다.
실제로 간단히 API를 작성하며 어떻게 다르게 동작하는지 살펴보도록 하겠습니다.

## 문제구현
```python
@api_view(['GET'])
def check_balance(request): # 계좌잔액확인 API
    account = AccountBalance.objects.get(owner=request.user)
    return Response({'balance': account.balance})

@api_view(['POST'])
def remittance(request): # 송금 API
    account = AccountBalance.objects.get(owner=request.user)
    to_user = AccountBalance.objects.get(owner_id=request.data['id'])
    time.sleep(5) # 5초 지연 발생
    account.withdraw(request.data['amount'])
    to_user.deposit(request.data['amount'])
    return Response({'status':'ok'}, status=status.HTTP_201_created)

@api_view(['POST'])
def withdraw(request):
    account = AccountBalance.objects.get(owner=request.user)
    account.withdraw(request.data['amount'])
    return Response({'status':'ok'}, status=status.HTTP_201_created)
```

위와 같이 뷰 함수를 구성하고 실제로 요청을 해보도록 하겠습니다.
```shell
# A -> B 5000원 송금
> http POST http://localhost:8000/bank/remittance "Authorization:Bearer A token" id:=2 amount:=5000
```
```shell
# 딜레이가 생긴 5초 동안 B가 7000원 출금
> http POST http://localhost:8000/bank/withdraw "Authorization:Bearer B token" amount:=7000
```
```shell
# 처리 결과 (B 잔액)
> http GET http://localhost:8000/bank/check "Authorization:Bearer B token" --body
{
    "balance": 15000
}
```

보이는 바와 같이, 기대했던 8,000원이 잔액으로 남아있는것이 아니라, 15,000원이 잔액으로 남아있습니다.

## 원인파악

이러한 문제가 발생한 원인은 다음과 같습니다.
1. A가 B에게 송금하기 위하여 B의 계좌 db hit(`to_user = AccountBalance.objects.get(owner_id=request.data['id'])`), 이때 B의 잔액은 10,000원
2. 송금이 이루어 지기 전에, B가 출금하기 위하여 B의 계좌 db hit(`account = AccountBalance.objects.get(owner=request.user)`), 이때 B의 잔액은 10,000원
3. B가 잔액 출금 후 DB에 반영(`account.withdraw(request.data['amount'])`), 이때 B의 잔액은 3,000원
4. A가 B에게 5,000원 송금(`to_user.deposit(request.data['amount'])`)후 DB에 반영, 이때 B의 잔액은 15,000원([1]단계에서 가져온 10,000원 + 송금받은 금액 5,000원)

B의 잔액에 대한 수정이 이루어지기 전에, B의 잔액을 (거의)동시에 수정 혹은 접근하다보니 생기는 문제입니다. 그렇다면 이러한 문제를 장고에서 어떻게 해결할수 있을까요?

## [`select_for_update`](https://docs.djangoproject.com/en/4.2/ref/models/querysets/#select-for-update)

장고에서 제공하는 `select_for_update` 메서드를 이용한다면, 다른 API 요청이 데이터베이스 자원을 사용하고 있을때, 다른 API 요청은 해당 자원에 대해 접근 할 수 없게 할 수 있습니다. 이러한 기능을 `베타적 잠금`이라고 합니다. 이를 이용해서 위의 코드를 수정해보도록 하겠습니다.

```python
@api_view(['GET'])
@transaction.atomic()
def check_balance(request):  # 계좌잔액확인 API
    account = AccountBalance.objects.select_for_update().get(owner=request.user)
    return Response({'balance': account.balance})


@api_view(['POST'])
@transaction.atomic()
def remittance(request):  # 송금 API
    account = AccountBalance.objects.select_for_update().get(owner=request.user)
    to_user = AccountBalance.objects.select_for_update(nowait=True).get(owner_id=request.data['id'])
    account.withdraw(request.data['amount'])
    to_user.deposit(request.data['amount'])
    time.sleep(5)  # 5초 지연 발생
    return Response({'status': 'ok'}, status=status.HTTP_201_CREATED)


@api_view(['POST'])
@transaction.atomic()
def withdraw(request):
    account = AccountBalance.objects.select_for_update().get(owner=request.user)
    account.withdraw(request.data['amount'])
    return Response({'status': 'ok'}, status=status.HTTP_201_CREATED)
```
(위 코드는 만약 sqlite 상에서 실습을 하고 있다면 동작하지 않습니다. [참고](https://stackoverflow.com/questions/3172929/operationalerror-database-is-locked))

```shell
# A -> B 5000원 송금
> http POST http://localhost:8000/bank/remittance "Authorization:Bearer A token" id:=2 amount:=5000
```
```shell
# 딜레이가 생긴 5초 동안 B가 7000원 출금
> http POST http://localhost:8000/bank/withdraw "Authorization:Bearer B token" amount:=7000
```
```shell
# 처리 결과 (B 잔액)
> http GET http://localhost:8000/bank/check "Authorization:Bearer B token" --body
{
    "balance": 8000
}
```

기대하는대로 작동하는것을 확인 할 수 있습니다.

## References
[1] [Django docs](https://docs.djangoproject.com/en/4.2/ref/models/querysets/#select-for-update)
[2] [How to Manage Concurrency in Django models](https://hakibenita.com/how-to-manage-concurrency-in-django-models#what-happened-here)
[3] 백엔드 개발을 위한 핸드온 장고