---
title: MongoDB 인덱스 - null 필드 처리
author: youngDaLee
date: 2024-11-14 12:00:00 +0900
categories: [DB]
tags: [MongoDB,index]
---

> 이 글은 2024년 11월 [Velog](https://velog.io/@youngda/MongoDB%EC%9D%98-%EC%9D%B8%EB%8D%B1%EC%8A%A4)에 작성했던 글 입니다


MongoDB는 NoSQL이기 때문에 필드 구조가 일정하지 않을 수 있다.
그렇다면 인서트한 도큐먼트에 인덱스를 건 필드가 존재하지 않는다면 어떻게 처리될까?
아래 네 가지 경우 각각 어떻게 처리되는지 궁금했다.
```
// 1. 일반 인덱스
db.collection.createIndex({ "name": 1 })
// 2. 희소 인덱스
db.collection.createIndex({ "name": 1 }, { sparse: true })
// 3. 부분 인덱스
db.collection.createIndex({ "name": 1 }, 
{ 
    partialFilterExpression: {
        name: {$ne: null} 
    }
}) 
// 4. 유니크 인덱스
db.collection.createIndex({ "name": 1 }, {unique:true})

// 데이터 입력 예시
db.collection.insert({ 
    "name": "다영", 
    "age": 11
})
// 이 상황!!! -> 여기서 어떻게 처리되지?
db.collection.insert({ 
    "nickname": "dy", 
    "age": 13
})
```

### 1. 일반 인덱스
```
db.collection.createIndex({ "name": 1 });
```
특징
* 컬렉션의 모든 문서를 대상으로 인덱스를 생성
* name 필드가 없어도 **해당 문서가 인덱스에 포함**됨!! -> null로 간주되어 인덱스에 Null값으로 저장됨

빈 값 처리
* name 필드가 없는 문서가 있으면 null 값으로 인덱스에 저장되기 때문에, name이 존재하지 않는 문서까지도 인덱스 검색 결과에 포함된다
* db.collection.find({ "name": null }) -> name 필드 없는 문서 인덱스로 조회

### 2. 희소 인덱스(sparse index)
```
db.collection.createIndex({ "name": 1 }, { sparse: true });
```
특징
* 필드가 존재하는 문서에 대해서만 인덱스를 생성
* name 필드가 없는 문서는 **인덱스에 포함되지 않음**

빈 값 처리
* name 필드가 없는 문서는 아예 인덱스에 포함X
* {name:null} 로 지정하여 데이터를 저장하면 인덱스에 포함하여 저장함!!

특이사항
```
db.collection.createIndex({ "name": 1 }, { sparse: true });

db.collection.insert({ // name필드 저장 x -> sparse인덱스에 포함되지 않음
    "nickname": "dy", 
    "age": 13
})
db.collection.insert({ // name필드 null로 저장 -> sparse인덱스에 포함됨
    "name": null, 
    "age": 13
})

db.collection.find({ "name": null })
```
위 경우에는 어떻게 인덱스를 타지??

<img width="539" alt="image" src="https://github.com/user-attachments/assets/0c250e21-bf85-491a-a015-a9cb861f7955">

* 일단 조회는 의도한대로 되는 것 확인됨

<img width="495" alt="image" src="https://github.com/user-attachments/assets/25a5ade5-e45a-4c6b-bc9e-cb43d9097350">

* 혹시몰라 데이터를 500건씩 넣고 검색해봤지만 COLLSCAN(풀스캔)을 하고 있는것을 확인

<img width="432" alt="image" src="https://github.com/user-attachments/assets/999d8bda-621c-4e87-ad09-043b9a84a37e">

* 인덱스 문제는 아님(string 검색할때는 인덱스 잘 탐)


그렇다면 아예 Null인 데이터만 넣은 경우에는 인덱스를 타나?(필드 없는 경우 없이 모두 인덱스를 인서트한 경우)
<img width="677" alt="image" src="https://github.com/user-attachments/assets/a1e798d6-da15-4e43-b9d5-d5356470f0f4">

* 안탄다...

#### 결론
* sparse 인덱스는 필드가 비어있으면 해당 필드를 제외하고 인덱스를 저장한다
* 인덱스 필드를 null로 지정하면 해당 인덱스를 포함하여 저장한다
* 다만 null로 검색해도 인덱스를 사용하지는 않는다

### 3. 부분 인덱스(partial index)
```
db.collection.createIndex({ "name": 1 }, 
{ 
    partialFilterExpression: {
        name: {$ne: null} 
    }
});
```
* 특정 조건을 만족하는 문서에 대해서만 인덱스를 생성
* 위의 설정은 name 필드가 null이 아닌 문서만 인덱스에 포함하도록 지정

빈 값 처리
* name 필드가 null이거나 필드 자체가 존재하지 않는 문서는 인덱스에 포함되지 않움
* db.collection.find({ "name": null }) -> 인덱스 사용 안됨
* 희소 인덱스(sparse index)와 차이점
  * name:null 값을 null로 지정해서 저장해도 부분인덱스에는 저장되지 않는다
  * 메모리 확보 면에서 부분 인덱스가 뛰어남


<img width="802" alt="image" src="https://github.com/user-attachments/assets/5f69187f-4db9-4af9-8141-9f025b8361c2">

* 공식 문서에서도 sparse인덱스보다 부분인덱스를 활용하는것을 권장하고 있다

### 4. 유니크 인덱스(unique index)
```
db.collection.createIndex({ "name": 1 }, { unique: true });
```
* 인덱스 필드 값이 중복되지 않도록 강제함
* 기본적으로 MongoDB에서는 null 값도 유니크하게 취급합
* 따라서 name 필드가 없는 문서는 name이 null로 인식되며, 중복된 null 값이 허용되지 않음

빈 값 처리
* name 필드가 없는 문서를 삽입할 때 **첫 번째 문서의 경우 문제없이 저장**되지만, 두 번째부터는 E11000 duplicate key error 오류가 발생
<img width="915" alt="image" src="https://github.com/user-attachments/assets/137ce2a6-7714-4e4c-9608-1e0f933b92d0">

* 빈 값을 검색할 때는 인덱스를 사용함
<img width="562" alt="image" src="https://github.com/user-attachments/assets/7c1e7e1f-ba74-4740-8c88-e8c45c338100">

### 정리
| 인덱스 종류   | name 필드가 없는 문서 인덱스 포함 여부 | name:null 문서 인덱스 포함 여부 | 중복 null 허용 여부 | 빈 값 처리 결과 |
| :---- |:---- |:---- |:---- |:---- |
|일반 인덱스 | 포함|포함| 허용 | null로 인덱싱됨 |
|희소 인덱스 | 미포함|포함| 허용 | 인덱스 미포함 |
| 부분 인덱스 | 미포함(조건에 따라 다름) |미포함(조건에 따라 다름) | 허용| 인덱스 미포함 |
| 유니크 인덱스 | 포함(null로 취급) | 포함 | 미허용 | null 검색 시 인덱스 활용 |

