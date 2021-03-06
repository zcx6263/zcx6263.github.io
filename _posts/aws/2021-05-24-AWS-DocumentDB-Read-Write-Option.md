---
layout: post
title: "[DocumentDB] Read Preference, Write Concern 설정"
subtitle: "AWS DocumentDB(MongoDB) Replication"
comments: true
categories : AWS
date: 2021-05-24
background: '/img/posts/mac.png'
---

## DocumentDB의 Replication    

`DocumentDB의 Replication은 기존 MongoDB의 Replica Set과 동일한 구조를 
갖고 있다.`         
따라서 기존 MongoDB와 동일한 방식으로 이용 가능하고, SDK 또한 동일하게 사용 가능하다.   

이번 포스팅에서 AWS DocumentDB(MongoDB) 사용시 Performance와 데이터 일관성을 
유지할 수 있는 여러가지 
Read preference, Write Concern 설정 방법을 알아 보자.    

- - - 

## Read Preference 란 무엇인가    

Read Preference 란 MongoDB의 Replica Set 설정시 Primary 와 Secondary 노드에 
대한 작업 처리 분산에 대한 설정이다.   
먼저 Replica Set 구조를 살펴보면, MySQL의 Replication 과 동일한 방식이다.   

모든 명령은 기본적으로 Primary 에서 처리하며 Secondary는 Primary에 
기록된 데이터를 Sync 하며 유지하게 된다.   

<img width="475" alt="스크린샷 2021-05-24 오후 10 49 02" src="https://user-images.githubusercontent.com/26623547/119357656-c7769400-bce2-11eb-8b83-8698f7c4cafe.png">   

`하지만, 이러한 구조는 Client가 Primary 하고만 데이터를 주고 받기 때문에 
MongoDB 서버로 총 3대를 사용함에도 작업 분산 효과를 누리지 못하며, 
        오로지 Secondary 노드는 데이터 백업 용도로만 사용할 수 밖에 없다.`    

이러한 비효율적인 구조를 조금이나마 개선할 수 있는 방법으로, Read 작업에 
대해서는 Secondary에서 처리하도록 설정이 가능하다.   

이것을 Read preference라고 한다.   

Read preference는 아래와 같이 5가지 방식을 지원한다.   

- primary    
- primary preferred   
- secondary   
- secondary prefferred   
- nearest   

#### 1) primary    

모든 read와 write 명령이 Primary 노드에서만 처리된다.   
따로 read preference 설정을 하지 않았다면, primary 설정으로 처리된다.   
`모든 명령이 Primary 에서만 처리하기 때문에 부하 분산 효과는 없으며, 
    secondary는 단지 백업 용도로 사용된다.`        

#### 2) primaryPreferred    

read와 write 명령이 모두 Primary 에서 처리되는 점에서 1번의 primary 모드와 
동일하다. 
`하지만, primary가 unavailable 상태인 경우, read 명령을 secondary 쪽으로 전달하여 
처리한다.`    
즉, primary가 장애가 날경우, write 명령은 동작하지 않지만 read 명령은 secondary에서 
동작 할 것이므로 primary 모드 보다는 service availability가 조금 더 
개선된 형태라고 보면 된다.   

#### 3) secondary    

모든 read명령은 secondary로 보내 처리하는 형태이다.    
`primary의 경우 write 작업만 처리하고 secondary에서 모든 read 명령을 처리하므로 
부하 분산 효과가 있다.`    
`하지만, primary와 secondary의 데이터는 asynchronous하기 때문에 사용상 주의해야 한다.`   

<img width="425" alt="스크린샷 2021-05-24 오후 11 01 46" src="https://user-images.githubusercontent.com/26623547/119359576-be86c200-bce4-11eb-9fe2-666a81e01f85.png">    

#### 4) secondaryPreferred    

모든 read 명령은 secondary로 보내 처리되는 점은 3번의 secondary 모드와 동일하다.   
하지만, secondary 멤버들이 모두 unvailable 한 경우 read 명령을 primary로 넘겨 처리하게 된다.   
secondary 모드 보다는 service availability가 조금 더 개선된 형태라고 보면 된다.   

#### 5) nearest   

read 명령의 경우 primary / secondary 상관없이 network latency가 가장 적은 인스턴스로 보내 
처리하는 방식이다.   
write 명령은 다른 모드와 동일하게 primary 에서처리한다.     
network latency는 maxStalenessSeconds의 설정 값을 확인하여 예상 지연값보다 해당 값이 
큰 노드를 제외하고 나머지 노드 중 random하게 지정하게 read 명령을 처리하게 된다.    


- - - 

## Write Concern 이란 무엇인가?      

`Write concern은 MongoDB가 Client의 요청으로 데이터를 기록할 때, 
      해당 요청에 대한 Response를 어느 시점에 주느냐에 대한 
      동작 방식을 지칭한다.`    

<img width="705" alt="스크린샷 2021-05-24 오후 11 14 33" src="https://user-images.githubusercontent.com/26623547/119360691-d579e400-bce5-11eb-9ce2-9889afc995ea.png">    

위 그림을 보면 MongoDB는 Client가 보낸 데이터를 Primary에 기록하고, 이에 대한 
Response를 Client에게 보내게 된다.   
이때, MongoDB를 Replication으로 구성하였을 경우 Primary의 복제본을 유지하는 
Secondary가 함께 구성되게 된다.    

만약 위 그림과 같이 Primary 1대와 Secondary 2대로 구성하였을 경우, 
    Client가 보낸 데이터의 Write 처리는 Primary에서만 먼저 처리하게 되며, 
    이후 Secondary로 변경된 데이터를 동기화 시키는 단계를 거친다.    

`이 때 눈여겨 봐야 하는 점은 Primary와 Secondary 간 동기화 되는 시간차가 있다는 
점이다.`    

만약 Client가 보낸 데이터를 Primary가 처리한 직후 Client 쪽으로 Response를 보내고 
이후, Primary와 Secondary 간 동기화가 진행된다고 가정하면    
Client 가 Response를 받은 시점과 Primary에서 Secondary로 Sync 되는 타이밍 사이에는 
데이터 일관성이 보장되지 않는 위험 구간이 존재하게 되는 것이다.    

<img width="674" alt="스크린샷 2021-05-24 오후 11 19 23" src="https://user-images.githubusercontent.com/26623547/119361322-7ec0da00-bce6-11eb-8fc1-3257462a3961.png">   

만약 이 사이에 Primary 에 장애가 발생 했다고 가정해 보면, 아직 최신 
데이터를 Sync 하지 못한 Secondary 멤버가 Primary로 승격되게 되고 Client는 
이를 알아차리지 못한 채 이미 작업이 완료된 Response를 받았기 때문에 
Client가 알고 있는 데이터와 DB 의 데이터가 unmatch 되는 상황이 발생하게 
된다.    

`이러한 문제를 해결하기 위해 Client 쪽에 보내는 response 시점을 
Primary와 Secondary 가 동기화 된 이후로 설정이 가능한데 
이것이 바로 Write concern 설정의 핵심이다.`    

<img width="709" alt="스크린샷 2021-05-24 오후 11 23 33" src="https://user-images.githubusercontent.com/26623547/119361877-17575a00-bce7-11eb-8d5e-5af906ccfae2.png">    

Write Concern을 설정하게 되면 
`Primary가 데이터 쓰기를 처리한 이후 바로 Client에게 response를 보내는 것이 
아니라 Secondary 쪽으로 데이터를 동기화 작업을 완료한 이후에 
Client에게 response를 보내게 된다.`        

이렇게 되면 Client와 Primary, Secondary 간에 데이터 일관성을 유지할 수 있게 된다.    

Write Concern을 지정하는데는 크게 w, j, wtimeout을 설정 할 수 있는데 자세한 내용은 아래와 같다.   

#### 1) w option    

w 를 설정하게 되면, ReplicaSet 에 속한 멤버 중 지정된 수 만큼의 멤버에게 
데이터 쓰기가 완료되었는지 확인한다.   
만약 Primary/Secondary 가 총 3대로 구성된 ReplicaSet일 경우, 
    w = 3으로 설정시 3대의 멤버에 데이터 쓰기가 완료된 것을 확인하고 response를 
    반환하게 된다. 보통은 w = 1 이 default이며, 이런 경우 Primary에만 기록 완료되면 
    response하게 된다.   

만약, w = majority로 설정할 경우, 멤버의 과반수 이상을 자동으로 설정하게 된다.   
즉, 3대의 멤버가 Replicaset에 속해 있을 경우 w = 2와 동일한 설정으로 보면 된다.   

#### 2) j option   

해당 값을 설정하면, 데이터 쓰기 작업이 디스크상의 journal에 기록된 후 완료로 
판단하는 옵션이다.   
만약, Replicaset의 멤버가 3대인 경우 w = majority, j = true 로 설정시   
Primary 1 대 Secondary 1대 총 2대의 멤버에서 디스크의 journal 까지 기록이 
완료 된 후 response하게 된다.    

#### 3) wtimeout    

해당 값을 설정하면, Primary에서 Secondary로 데이터 동기화시 timeout 값을 설정하는 옵션이다. 
만약 wtimeout의 limit을 넘어가게 되면 실제로 데이터가 primary에 기록되었다고 해도 error를 
리턴하게 된다.    
설정 단위는 millisecond 이다.   

- - - 

## DocumentDB의 Write Concern 확인    

DocumentDB에 설정된 Write Concern 값을 확인해 보자.   
설정 확인은 MongoDB에 접속 후 rs.conf() 명령을 통해 확인 가능하다.   

```
rs0:PRIMARY> rs.conf()
```



 

- - -   

**Reference**

<https://docs.mongodb.com/manual/core/read-preference/>    
<https://bluese05.tistory.com/73>    

{% highlight ruby linenos %}


{% endhighlight %}


{%- if site.disqus.shortname -%}
    {%- include disqus.html -%}
{%- endif -%}

