---
title: "[해키피디아] ORM(Object Relational Mapping)"
author: y00n_nms
tags: [y00n_nms, orm]
categories: [Hackypedia]
date: 2021-12-15 14:00:00
cc: true
index_img: /img/hackypedia.png

---

ORM 이전에는 데이터베이스에 접근하기 위하여 다음과 같은 SQL 쿼리를 작성합니다. 하지만 웹 애플리케이션 프로젝트의 규모와 복잡성이 증가할 수록, 특히 처리해야 하는 모든 데이터 필터링을 고려할 때 매우 때때로 매우 복잡해집니다. 또한 객체 모델과 관계형 데이터베이스 간의 불일치로, 관계형 데이터베이스에 객체를 로드하고 저장하면 다음과 같은 5가지 불일치 문제가 발생합니다.

1. Granularity: 때로는 데이터베이스의 테이블 수보다 더 많은 클래스가 있는 객체 모델이 있습니다.
2. Inheritance: RDBMS는 객체지향 프로그래밍 언어의 특징인 상속 개념이 없습니다.
3. Identity: RDBMS는 동일성을 정의하기 위해 primary key를 이용하지만 JAVA는 (a==b)와 (a.equals(b))를 모두 정의합니다.
4. Associations: RDBMS는 연관성을 나타내기 위해 foriegn key를 이용하지만 객체지향언어는 객체의 참조를 이용합니다.
5. Navigation: Java와 RDBMS에서 객체에 액세스하는 방법은 근본적으로 다릅니다.

ORM(Object-Relational Mapping)은 위의 모든 불일치를 처리하는 솔루션으로, 관계형 데이터베이스와 Java, C# 등과 같은 객체 지향 프로그래밍 언어 간에 데이터를 변환하는 프로그래밍 기술입니다. 즉, SQL 대신 이외의 객체 지향 프로그래밍 언어를 선택하여 데이터베이스와 상호작용 할 수 있습니다.

```sql
SELECT * FROM users WHERE email = 'test@test.com';
```

```java
var orm = require('generic-orm-libarry');
var user = orm("users").where({ email: 'test@test.com' });
```

ORM 프레임워크 종류는 다음과 같습니다.

- JAVA : JPA, Hibernate, EclipseLink, DataNucleus, Ebean 등
- C++ : ODB, QxOrm 등
- Python : Django, SQLAlchemy, Storm 등
- iOS : DatabaseObjects, Core Data 등
- .NET : NHibernate, DatabaseObject, Dapper 등
- PHP : Doctrine, Propel, RedBean 등

