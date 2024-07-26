# 👩‍👧‍👧Woori_MySQL_ELK_mini_project

## 개요
Ubuntu 환경에서 Logstash를 활용해 MySQL과 ElasticSearch를 연동하고, Kibana를 활용해 타이타닉 데이터셋과 에스토니아 데이터셋의 시각화 및 분석을 진행하였습니다. 

## 🦕Team
|<img src="https://avatars.githubusercontent.com/u/107031994?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/65991884?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/114724461?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|
|Jeongju Park<br/>[@gorapang](https://github.com/gorapang)|[@RyuChaeHyun](https://github.com/RyuChaeHyun)|[@hyunkkkk](https://github.com/hyunkkkk)|

---
### 🦕 **활용 데이터**

- Titanic Dataset
(https://www.kaggle.com/competitions/titanic/data?select=train.csv)
- The Estonia Disaster Passenger List
(https://www.kaggle.com/datasets/christianlillelund/passenger-list-for-the-estonia-ferry-disaster)

<br>

### 🦕 **개발환경**
```
Ubuntu 22.04
MySQL  Ver 8.0.37-0ubuntu0.22.04.3 for Linux on x86_64 ((Ubuntu))
LogStash 7.17.22
ElasticSearch 7.17.22
```
---

## 1️⃣ MySQL에 csv 파일 업로드하기

### 📍Trouble Shooting #1

**🤔 문제**

아래와 같은 에러가 발생하며 **307번째 데이터까지만 저장되는 현상**이 발생하였다.

![Untitled (1)](https://github.com/user-attachments/assets/9c1da1bf-340d-45e8-8ef5-882c23ea82e2)


**💡 원인**

308번 데이터의 이름: 

`"Penasco y Castellana, Mrs. Victor de Satode (Maria Josefa Perez de Soto y Vallejo)”`

Name의 필드 타입은 `VARCHAR(64)`인데, 308번 **데이터의 이름이 너무 길어서** 이후로는 저장되지 않는 문제였다.

✅ **해결**: train 테이블의 name 크기를 변경한다.

```sql
desc train;

ALTER TABLE train MODIFY COLUMN name VARCHAR(200);
TRUNCATE TABLE train;
```

이후 다시 데이터 가져오기를 통해 csv 파일을 가져오면 된다.

## **2️⃣** MySQL JDBC 드라이버 설치

Ubuntu에 MySQL JDBC 드라이버를 설치한다. `mysql-connector-java-8.0.26`을 설치하였다.

```bash
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.26.tar.gz
tar -xvf mysql-connector-java-8.0.26.tar.gz
```

```bash
sudo mv mysql-connector-java-8.0.26/mysql-connector-java-8.0.26.jar /usr/share/logstash/logstash-core/lib/ja
```

## 3️⃣ Logstash 설정 파일 수정

`mysql_to_es.conf`

```sql
input{
  jdbc{
    jdbc_driver_library => "/usr/share/logstash/logstash-core/lib/jars/mysql-connector-java-8.0.26.jar"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/fisa?useUnicode=true&characterEncoding=utf-8&serverTimezone=UTC"
    jdbc_user => "root2"
    jdbc_password => "root2"
    statement => "SELECT * FROM train"
  }
}

output {
  stdout {
    codec => rubydebug
  }

  elasticsearch {
    hosts => ["http://localhost:9201"]
    index => "titanic_train"
  }
}
```

- logstash  설정 파일을 수정하고 logstash를 재실행한다.

## 4️⃣ 주제 선정 및 데이터 수집⛵

**타이타닉과 다른 해양 사고(Estonia Disaster)의 데이터 비교**라는 주제를 선정하였다.

캐글의 *The Estonia Disaster Passenger List* 데이터셋을 사용하였다.

[Titanic - Machine Learning from Disaster](https://www.kaggle.com/competitions/titanic/data?select=train.csv)

[The Estonia Disaster Passenger List](https://www.kaggle.com/datasets/christianlillelund/passenger-list-for-the-estonia-ferry-disaster)

### 🚢 **Titanic : Data Dictionary**

| Variable | Definition | Key |
| --- | --- | --- |
| survival | 생존율 | 0 = No, 1 = Yes |
| pclass | 티켓 클래스 | 1 = 1st, 2 = 2nd, 3 = 3rd |
| sex | 성별 |  |
| Age | 나이 |  |
| sibsp | 함께 탑승한 형제자매, 배우자 수 총합 |  |
| parch | 함께 탑승한 부모, 자녀 수 총합 |  |
| ticket | 티켓 넘버 |  |
| fare | 탑승 요금 |  |
| cabin | 객실 넘버 |  |
| embarked | 탑승 항구 | C = Cherbourg, Q = Queenstown, S = Southampton |

### 🌊 Estonia: Data Dictionary

| Variable | Definition | Key |
| --- | --- | --- |
| Country | 국가 |  |
| Firstname | 이름 |  |
| Lastname | 성 |  |
| Sex | 성별 | M = Male, F = Female |
| Age | 나이 |  |
| Category | 승객분류 | C = Crew, P = Passenger |
| Survived | 생존율 | 0 = No, 1 = Yes |
- **공통된 칼럼**: 이름, 성별, 나이, 생존여부
- **타이타닉에만 존재하는 칼럼**: 형제의 수, 부모의 수, 티켓 등급, 요금, 항구
- **에스토니아에만 존재하는 칼럼**: 국가, 카테고리(승객 or 크루원)

## 5️⃣ 데이터 전처리

### 📊 타이타닉 결측치 분석

```sql
mysql> select count(*) from train where Age is NULL;
+----------+
| count(*) |
+----------+
|      177 |
+----------+
```

Age가 null인 행이 177개 존재함을 확인하였다.


### 📊 타이타닉 결측지 전처리

**방법**1️⃣ **타이타닉 데이터셋의 age 컬럼의 null값을 평균값으로 채운다.**

→ 다른 행들의 데이터 유실을 막기 위함

```sql
CREATE TABLE train_copy LIKE train; //테이블 복제
INSERT INTO train_copy SELECT * FROM train; //데이터 복제

SET @avg_age = (SELECT AVG(age) FROM train WHERE age IS NOT NULL);

UPDATE train
SET age = @avg_age
WHERE age IS NULL;
```

**처리 전**

![image](https://github.com/user-attachments/assets/c9cd4e5d-3c16-4a5c-9fd8-f7ea7e9822f1)


**처리 후**

![Untitled (2)](https://github.com/user-attachments/assets/110c4da8-7531-4267-aeb8-c5e033644471)


**방법2️⃣ 타이타닉 데이터셋의 age 필드 값이 NULL인 행을 제외한다.**

→ age 컬럼을 평균으로 맞출 경우 평균에 해당하는 나이 컬럼에 데이터가 몰리게 되고, 정확한 분석이 되지 않는다. 그래서 age 컬럼이 null 인 필드를 제외하고 나이 필드와 관련된 분석은 위 데이터로 진행하였다. 

**logstash - *.conf 수정**

```jsx
input{
  jdbc{
    jdbc_driver_library => "/lib/mysql-connector-java-8.0.18.jar"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/fisa?useUnicode=true&characterEncoding=utf-8&serverTimezone=UTC"
    jdbc_user => "root"
    jdbc_password => "root"
    statement => "SELECT * FROM train_copy where age is NOT NULL"
    # schedule => "* * * * *"
  }
}
```

**✅ 결과**

891→714로 줄어들었다.

![Untitled (3)](https://github.com/user-attachments/assets/8f712d28-7081-47da-9486-ff7d4587d9c2)


### 📊 Age_Group 필드 추가

✅ **나이대별 분석을 위해 나이대별 분류 전처리 작업을 진행한다.**

- `age_group` 컬럼 생성
- age 값에 따른 나이대 값을 age_group에 세팅

**Titanic**

```sql
ALTER TABLE train
ADD COLUMN age_group VARCHAR(50);

UPDATE train
SET age_group = CASE
    WHEN age BETWEEN 0 AND 9 THEN '0대'
    WHEN age BETWEEN 10 AND 19 THEN '10대'
    WHEN age BETWEEN 20 AND 29 THEN '20대'
    WHEN age BETWEEN 30 AND 39 THEN '30대'
    WHEN age BETWEEN 40 AND 49 THEN '40대'
    WHEN age BETWEEN 50 AND 59 THEN '50대'
    WHEN age BETWEEN 60 AND 69 THEN '60대'
    WHEN age BETWEEN 70 AND 79 THEN '70대'
    WHEN age >= 80 THEN '80대 이상'
    ELSE 'Other'
END;
```

**Estonia**

```sql
ALTER TABLE estonia
ADD COLUMN age_group VARCHAR(50);

UPDATE estonia
SET age_group = CASE
    WHEN age BETWEEN 0 AND 9 THEN '0대'
    WHEN age BETWEEN 10 AND 19 THEN '10대'
    WHEN age BETWEEN 20 AND 29 THEN '20대'
    WHEN age BETWEEN 30 AND 39 THEN '30대'
    WHEN age BETWEEN 40 AND 49 THEN '40대'
    WHEN age BETWEEN 50 AND 59 THEN '50대'
    WHEN age BETWEEN 60 AND 69 THEN '60대'
    WHEN age BETWEEN 70 AND 79 THEN '70대'
    WHEN age >= 80 THEN '80대 이상'
    ELSE 'Other'
END;
```

![Untitled (4)](https://github.com/user-attachments/assets/b7b84ffa-e1d1-44ad-a282-5e7ee718d0a5)


---

# 6️⃣ SQL 데이터 분석

### 🧑 **나이대별 생존율**

- 80대 1명 중 1명이 생존한 것을 제외하면, 10세 이하 아이들의 생존율이 가장 높다.

```sql
SELECT
    age_group,
    COUNT(*) AS total,
    SUM(Survived) AS survived,
    ROUND(SUM(Survived) / COUNT(*) * 100, 2) AS survival_rate
FROM
    train
GROUP BY
    age_group
ORDER BY 
    age_group;
```

```sql
+--------------+-------+----------+---------------+
| age_group    | total | survived | survival_rate |
+--------------+-------+----------+---------------+
| 0대          |    62 |       38 |         61.29 |
| 10대         |   102 |       41 |         40.20 |
| 20대         |   220 |       77 |         35.00 |
| 30대         |   167 |       73 |         43.71 |
| 40대         |    89 |       34 |         38.20 |
| 50대         |    48 |       20 |         41.67 |
| 60대         |    19 |        6 |         31.58 |
| 70대         |     6 |        0 |          0.00 |
| 80대 이상    |     1 |        1 |        100.00 |
| Other        |   177 |       52 |         29.38 |
+--------------+-------+----------+---------------+
```

### 🧑‍🦰 **어떤 나이대, 성별의 생존률이 가장 높을까?**

- 80대 남성 1명 중 1명이 생존한 것을 제외하면, **60대 여성 - 50대 여성 - 30대 여성** 순으로 생존율이 높았다.

```sql
SELECT
    age_group,
    Sex,
    COUNT(*) AS total,
    SUM(Survived) AS survived,
    ROUND(SUM(Survived) / COUNT(*) * 100, 2) AS survival_rate
FROM
    train
GROUP BY
    age_group, Sex
ORDER BY
    survival_rate DESC;
```

```sql
+--------------+--------+-------+----------+---------------+
| age_group    | Sex    | total | survived | survival_rate |
+--------------+--------+-------+----------+---------------+
| 80대 이상    | male   |     1 |        1 |        100.00 |
| 60대         | female |     4 |        4 |        100.00 |
| 50대         | female |    18 |       16 |         88.89 |
| 30대         | female |    60 |       50 |         83.33 |
| 10대         | female |    45 |       34 |         75.56 |
| 20대         | female |    72 |       52 |         72.22 |
| 40대         | female |    32 |       22 |         68.75 |
| Other        | female |    53 |       36 |         67.92 |
| 0대          | female |    30 |       19 |         63.33 |
| 0대          | male   |    32 |       19 |         59.38 |
| 30대         | male   |   107 |       23 |         21.50 |
| 40대         | male   |    57 |       12 |         21.05 |
| 20대         | male   |   148 |       25 |         16.89 |
| 50대         | male   |    30 |        4 |         13.33 |
| 60대         | male   |    15 |        2 |         13.33 |
| Other        | male   |   124 |       16 |         12.90 |
| 10대         | male   |    57 |        7 |         12.28 |
| 70대         | male   |     6 |        0 |          0.00 |
+--------------+--------+-------+----------+---------------+
```

---

# 7️⃣ 데이터 시각화

## 👀 Titanic - Estonia 비교

- **생존율**
    
    전체 생존율을 비교했을때, **타이타닉** 생존율이 에스토니아 생존율에 비해 20% 이상 **더 높다.**
    
![image](https://github.com/user-attachments/assets/944ce2fd-9701-402a-bbde-cd2e81787975)


---

- **탑승객 성비**
    
    탑승객 성비를 비교했을때, 타이타닉은 **남자가 약 2배** 더 많이 탑승했으며 에스토니아는 **거의 동일**하게 탑승하였다. 
    
![image](https://github.com/user-attachments/assets/c2373930-8287-4512-93c4-3852c3a64f1c)


---

위 결과를 바탕으로 **타이타닉과 에스토니아의 성별별 생존자와 사망자 수를 비교**하였다.

---

- **타이타닉 성별별 생존자/사망자 수**
    
    타이타닉의 경우, **여자**는 **생존자수**가 사망자수의 **두배** 이상이었지만 **남자**는 **사망자수**가 생존자수의 **네배** 이상이었다. 
    
    ⇒ 타이타닉 사고에서 *여성의 생존율이 상대적으로 높고* *남성의 생존율이 극도로 낮은 것*으로 나타났다.
    
![성별에 따른](https://github.com/user-attachments/assets/61225fed-fd31-4a14-8743-90cabcab1dd3)

```
여 : 생존자수가 사망자수의 2배 이상
남 : 사망자수가 생존자수의 4배 이상
```

---

- **에스토니아 성별별 생존자/사망자 수**
    
    **에스토니아의 경우**,  **남자**는 **사망자수**가 생존자수의 **3배** 이상이었지만, **여자**는 **사망자수**가 생존자수의 **10배** 이상이었다.
    
    ⇒ 에스토니아 여객선 사고에서 *여성의 생존율이 극도로 낮고 남성의 생존율도 낮은 것*으로 나타났다.
    
![성별에 따른2](https://github.com/user-attachments/assets/358e5bd5-91a4-4b4b-897f-b33c578cdb61)

```jsx
남 : 사망자수가 생존자수의 3배 이상
여 : 사망자수가 생존자수의 10배 이상
```

---

**❓타이타닉이 에스토니아 보다 여성 생존율이 훨씬  더 높은 이유는❓**

타이타닉 호에서는 승무원과 승객이 선장 에드워드 스미스의 지시에 따라 **규범적으로 행동**했기 때문이다. 

타이타닉호 선장이었던 에드워드 존 스미스는 구명보트가 한정돼 있기 때문에 ***사회적 약자인 여성과 어린이를 먼저 구하라고 지시했다.***  승객 중에서 어린이, 여자, 남자 순으로 탈출토록 했고, 총으로 공포를 쏘면서 이성을 잃은 사람들이 질서를 유지하도록 하게 했다. 따라서 타이타닉은 선장의 책임감과 직업의식으로 여성 생존율이 훨씬 더 높게 나왔고, **에스토니아는 상대적으로 힘이 센 남성의 생존율이 더 높게 나온 것으로 보인다.**

---

- **타이타닉 나이대별 생존자/사망자 수**
    
    **0~10세**에서는 ****생존자 수가 사망자 수보다 높았다. 
    
    나머지 나이대에서는 사망자 수가 생존자 수보다 높으며, 특히 **20대**의 생존자 수 대비 사망자 수가 현저히 높다.
    
![10대 비교](https://github.com/user-attachments/assets/ad13bfb6-f988-4241-8afa-3a1f83022269)

    

- **에스토니아 나이대별 생존자/사망자 수**

![10대 비교2](https://github.com/user-attachments/assets/8e50e83a-baa1-4481-b600-f735e0b39a99)


---

그래서 에스토니아와 타이타닉의 나이대별 생존자/사망자 수를 비교한 결과, 타이타닉에서 10대 이하 승객들의 **생존자 수**가 사망자 수보다 더 큰 것으로 나타났다.

---

## 👀 타이타닉 데이터 시각화

- **탑승 항구별 생존자/사망자 수**
    
    **S 항구**(Southampton)에서 탑승한 승객은 사망자가 생존자의 2배 이상이었으나, **C 항구**(Cherbourg)에서 탑승한 승객은 생존자가 더 많았다. 단순히 항구 위치의 차이로 생존율의 차이가 발생할 것 같지는 않았기 때문에, 탑승 항구별 티켓 등급을 분석하였다.
    
![확인](https://github.com/user-attachments/assets/53541383-4a5b-48f1-95b1-ebfb422e5130)

    

---

- **탑승 항구별 티켓 등급**
    
    **S 항구**에서 탑승한 승객의 티겟 등급은 **3등석**이 가장 많았다. **C 항구**에서 탑승한 승객의 티겟 등급은 **1등석이 50% 이상**이었다.
    
![Untitled (5)](https://github.com/user-attachments/assets/76e5fcd9-9cc4-43e8-9f40-7648bf8e6ba1)


- **생존여부별 등급 비율**
    
    **사망자** 중에서는 **3등급**의 비율이 가장 높았다. **생존자** 중에서는 **1등급이 50% 이상**으로 가장 큰 비중을 차지하였다.
    
![등급2](https://github.com/user-attachments/assets/aab01d42-581a-4d33-971f-c42fecc69a0c)

    

결론적으로, **티켓 등급이 높을수록 생존 확률이 높았음**을 확인할 수 있다.

또한, S 항구에서는 3등급 티켓을 구매한 사람이 많았고, C 항구에서는 1등급 티켓을 구매한 사람이 많은 것을 보아, **C 항구에는 부유한 사람들**이 많이 거주하고 있었으며, **S 항구에는 상대적으로 경제적 여유가 적은 사람들**이 많이 거주하고 있었음을 유추할 수 있다.

---

## 👀 에스토니아 데이터 시각화

- **승객 카테고리**
    
    P는 Passenger (승객), C는 Crew (승무원)를 의미한다. 에스토니아호의 사람들 중 **20%는 승무원**이었다.
    

![image](https://github.com/user-attachments/assets/71dac0a7-f015-45d2-9bae-c2d9bac46eee)


- **승객 카테고리 별 생존자/사망자 수**
    
    승객과 승무원 모두 생존자 수가 사망자 수보다 현저히 낮다. 
    
    그러나 비율로 보았을 때 **승무원의 생존율**이 승객의 생존율보다 높다.
    
![Untitled (6)](https://github.com/user-attachments/assets/8fa7603d-1bd7-4a8b-b0a5-6141abe02a1a)


- **국가**
    
    에스토니아호는 에스토니아에서 스웨덴으로 가는 선박이었다. 
    
    때문에 에스토니아인보다 **스웨덴인**이 더 많이 탑승한 것으로 보인다.
    

![image](https://github.com/user-attachments/assets/94d16402-72f9-4ab2-b1bf-52cf3c227196)


- **국가별 생존자/사망자 수**
    
    스웨덴인이 에스토니아인에 비해 더 많이 탑승했음에도 불구하고, **에스토니아인**이 스웨덴인보다 더 많이 생존했다. 
    
    기록에 따르면 당시 선장의 침몰 안내 방송은 에스토니아어로 이루어졌다고 한다. 따라서 에스토니아인의 생존 확률이 더 높았던 것으로 보인다.
    
![Untitled (7)](https://github.com/user-attachments/assets/2c945bc1-72f0-4648-a3f9-d742508915c8)

