---
layout: post
title: "[Scala] Future와 동시성"
subtitle: "Concurrency"    
comments: true
categories : Scala
date: 2021-07-07
background: '/img/posts/mac.png'
---

오늘 멀티 코어 프로세서가 대중화되면서 동시성에 대한 관심도 많이 
늘어났다. 이러한 동시성 프로그래밍을 위해서 기존의 프로그래밍 언어들은 
블로킹을 사용하여 동기화함으로써 동시성을 지원한다.    
자바의 경우도 마찬가지로 공유 메모리와 락을 기반으로 동시성을 지원하고 
있다. 그러나 이러한 블로킹을 사용한 동기화의 경우 Deadlock이나 
Starvation과 같은 문제가 발생할 수 있다. 이렇게 블로킹 기반의 
동시성 처리는 이러한 어려움이 있다. 그래서 비동기적 프로그래밍을 사용하면 
이러한 블로킹을 없앨 수 있다.   
비동기적(asynchronous) 프로그래밍이란 아래의 그림과 같이 메인 프로그램의 
흐름이 있고 이와 독립적으로 실행되는 프로그래밍 스타일을 의미한다.   

<img width="738" alt="스크린샷 2021-07-07 오후 4 26 38" src="https://user-images.githubusercontent.com/26623547/124717513-2a915280-df40-11eb-87e3-5beceecb6552.png">   

`스칼라에서는 이러한 비동기적 프로그래밍을 지원하기 때문에 Future라는 것을 
지원한다. Future는 스칼라의 표준라이브러리로써 Future를 사용해 
변경 불가능한 상태를 비동기적으로 변환하여 블로킹 방식의 동시성 처리의 
어려움을 피할 수 있게 해준다.`       

자바에도 Future가 있지만 스칼라의 Future와 다르다. 두 Future 모두 비동기적인 
연산의 결과를 표현하지만 자바의 경우 블로킹 방식의 get()을 사용해 
결과를 얻어와야 한다. 오히려 자바 8에 추가된 CompletableFuture가 
스칼라의 Future와 비슷하다. CompletableFuture를 정의하고 그 값을 
얻었을 때 행동을 정의할 수 있다.   

반면 스칼라의 Future는 이미 연산 결과의 완료 여부와 관계없이 결과 값의 변환을 
지정할 수 있다. Future의 연산을 수행하는 스레드는 암시적으로 제공되는 
Execution context를 사용해 결정된다. 이러한 방식을 사용해서 
불변 값에 대한 일련의 변환으로 비동기적 연산을 표현할 수 있고, 
    공유 메모리나 락에 대해 신경 쓸 필요가 없는 장점이 있다.   

- - - 

## 1. 낙원의 곤경   

공유 데이터와 락 모델을 사용해 멀티 스레드 어플리케이션을 신뢰성 있게 
구축하기가 어렵다. 프로그램의 각 위치에서 접근하거나 변경하는 데이터 중 
어떤 것을 다른 스레드가 변경하거나 접근할 수 있는지를 추론하고, 어떤 
락을 현재 가지고 있는지 알아야 한다. 거기다가 더 어려운점은 프로그램이 
실행되는 도중에 새로운 락을 얼마든지 만들 수 있다.   
즉, 락이 컴파일 시점에 고정되지 않는다는 점이다.   
자바의 java.util.concurrent 라이브러리는 고수준의 동시성 프로그래밍 
추상화를 제공해서 저수준의 공유 데이터와 락 모델을 사용하는 것보다는 
오류의 여지가 적지만 마찬가지로 내부적으로 공유 데이터와 
락 모델을 사용하기 때문에 해당 문제의 근본적인 어려움을 해소하긴 어렵다.   
스칼라의 Future는 공유 데이터와 락의 필요성을 줄여주는 동시성 처리 
방식을 제공한다.   
물론 Future가 모든 것을 해결할 수는 없지만 위에서 발생한 문제를 
더욱 간단하게 해결해주는 건 확실하다.   

- - -   

## 2. 비동기 실행과 Try    

스칼라에서 메소드를 호출하고 그 결과가 Future라면 그것을 비동기적으로 
진행할 다른 연산을 표현하는 것이다. 이러한 Future를 실행하기 위해서는 
암시적인 execution context가 필요하다.   
Future에는 isCompleted와 value 메서드를 통해서 polling 기능을 제공한다. 
value 메서드의 경우 Option[Try[T]]의 값을 리턴한다. Try는 성공을 
나타내는 Suceess와 예외가 들어있어 실패를 나타내는 Failure 중에 
하나를 표현한다. Try의 목적은 동기적 계산이 있는 try 식이 하는 역할을 
비동기 연산을 할 때 동일한 기능을 제공해주는 역할을 한다.   
Try의 계층 구조는 아래의 그림과 같다.   

<img width="601" alt="스크린샷 2021-07-07 오후 4 52 24" src="https://user-images.githubusercontent.com/26623547/124721457-1a7b7200-df44-11eb-9678-1b06ce840cd1.png">   

동기적 연산에는 try/catch 구문으로 메서드가 던지는 예외를 그 메서드를 실행하는 
스레드에서 잡을 수 있다.   
그러나 비동기적 연산에는 연산을 시작한 스레드가 다른 작업을 계속 진행하는 경우가 있다. 
그래서 비동기적 연산에서는 Try 객체를 사용해서 예외가 발생하여 갑자기 종료되는 
상황에 대비할 수 있다.     





- - - 

**Reference**    

<https://seamless.tistory.com/46>    

{% highlight ruby linenos %}

{% endhighlight %}


{%- if site.disqus.shortname -%}
    {%- include disqus.html -%}
{%- endif -%}

