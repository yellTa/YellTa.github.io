---
layout: post
author: YellTa
title: "[영속성 컨텍스트 DB 충돌에러] [Conflict between DB and PersistContext]"
tags: [issue]
categories: ["Memo","Spring"]
---


# Given
![givenImg](..\assets\images\postImg\231025DBConfict\given1.png)



JPARespository를 이용해 삭제 작업을 진행했다.

.deleteAll() → DB의 모든 내용을 지운다.(Custom Query)

.flush() → 내용을 영속성 컨텍스트에 전달한다.

.initIncrement() → DB table의 자동 인덱스 값을 1로 변경한다.(Custom Query)

<span class="eng">
I executed delete Entity from Db throgh JPARepository<br>
.deleteAll() → Delete All Entity from DB(Custom Query)<br>
.flush() → transfer the result to Persist <br>
.initIncrement() → modifying Auto Increment Value in DB(Custom Query)
</span>

# Error
![Error](..\assets\images\postImg\231025DBConfict\error.png)
.save()를 이용해서 insert를 실행하려 했지만 Entity Exist Exception이 발생했다.
DB에서는 Entity가 지워졌던 상황

<span class="eng">
I tried to save through save Method in JPA but It occurred Entity ExistException.<br>
But Entity remove from DB
</span>


# Conclusion

Spring jpa는 Transaction을 이용한다.

1. deleteAll() → 쿼리문을 이용해 table을 trancate한다.<br>
2. initIncrement() → increment의 값을 초기화 한다.

    >truncate 를 실행하면 자동으로 increment가 초기화 된다.

    ---
        @Modifying
        @Query("alter table table_name auto_increment=1", nativeQuery = true)
        public void initIncrement(){
        }
    ---


3. 커스텀 쿼리의 내용은 영속성 컨텍스트처럼 Transaction이 끝난 후 한꺼번에 적용되는 것이 아니라 즉시 적용 된다.<br>

4. .save()<br>
    save의 결과는 영속성 컨텍스트에 들어가게 된다.  DB는 비어있지만 영속성 컨텍스트는 지워지지 않은 상태기 떄문에 에러가 발생한다.


-----

<br><br><br><br>
Spring jpa uses Transaction

1. deleteAll() → truncate Table by Custom Query<br>
2. initIncrement() → Initializes increment

    >Executing truncate initializes incrment

    ---
        @Modifying
        @Query("alter table table_name auto_increment=1", nativeQuery = true)
        public void initIncrement(){
        }
    ---


3. Custom Query executed at executed time, but Persist context execute Query after finishing Trasaction<br>

4. .save()<br>
    save query stored Persist Context. DB doesn't have any entity but the PersistContext has Entity so 
    It occurred Error(Entity Exist Exception).

-----


# After execute save Method() - Real DB
![afterDB](..\assets\images\postImg\231025DBConfict\afterDB.JPG)<br>

# After execute save Method() - PersistContext
![afterPersist](..\assets\images\postImg\231025DBConfict\afterPersist.JPG)

영속성 컨텍스트에서는 삭제가 먼저 실행되지 않는다. Entity가 있는 상황에서 같은 값을 넣으니 Entity Exist Error가 발생한 것<br>

<span class="eng">
In the persist Context, not exeute delete method 1st. So save same Entity in the persist Context makes  Entity Exist Error.
</span>

JPA는 영속성 컨텍스트를 이용한다.

DB의 스냅샷을 찍어놓고 영속성 컨텍스트의 1차 캐시에서 데이터를 추가하고 삭제한 뒤에 결과를 DB 쿼리로 수행한다.
<br><br>

<span class="eng">
JPA usees Persist Context<br><br>
Take a snapshot of DB and Execute DB Query after save and delete Data to the Persist Context's 1st cache Data.
</span>

![perform](..\assets\images\postImg\231025DBConfict\perform.png)



Transaction이 끝나고 쿼리가 실행되는 순서이다. 

1. Insert
2. update
3. delete

순서로 실행이 된다. 

즉 영속성 컨텍스트에서는 delete가 먼저 수행될 수 없고 DB에 쿼리를 날릴때는 Insert를 먼저 수행하기 때문에 delete가 실행되지 않아 Entity가 존재하는 상태에서 Insert를 하니 에러가 생기게 된다.


-----
<span class="eng">
Query executing Sequence after finishing Transaction</span><br>  
1. Insert
2. update
3. delete

<span class="eng">
Persist Context can't execute delete 1st and execute Insert query when using Query to DB so 
it makes an Error because persist Context is not empty.
</span>


# Conclusion
## slove way
DB와 영속성 컨텍스트의 싱크를 맞추도록 하자
<span class="eng">
Synchronize DB and Persist Context
</span>


![solve1](..\assets\images\postImg\231025DBConfict\solve1.png)
<br>
오른쪽의 그림에서 보면 persist Context를 Clear해서 DB와 동기화를 한다..
<br>

<span class="eng">
Right side of pic, clears Persist Context to synchronize DB.
</span>

![solve3](..\assets\images\postImg\231025DBConfict\solve3.png)<br>
<br>

![solve4](..\assets\images\postImg\231025DBConfict\solve4.png)<br>


![solve2](..\assets\images\postImg\231025DBConfict\solve2.png)



@Modifying이 붙은 쿼리에 clearAutomatically = true 옵션을 부여해 영속성 컨텍스트를 비우도록 한다.<br>
<span class ="eng">
@Query with @Modifying Annotation, add clearAutomatically = true option so clear the Persist Context.
</span>