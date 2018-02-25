---
title: Node.js의 GC는 어떻게 동작하는가?
date: 2018-02-19 21:13:39
tags:
---


---
이 글은 [RisingStack](https://blog.risingstack.com)의 Node.js at scale 시리즈 중에서 [Node.js Garbage Collection Explained](https://blog.risingstack.com/node-js-at-scale-node-js-garbage-collection/) 글을 번역한 글입니다.
(저자에게 댓글로 허락을 구하긴 했는데 아직 답변이 없어서 나중에 이 글이 삭제될 가능성이 있음을 알려드립니다.)

---

이 글에서는 Node.js garbage collection이 어떻게 작동하는지, 당신이 코드를 작성할 때 백그라운드에서 어떤 일이 일어나는지, 그리고 메모리가 어떻게 회수되는지 배우게 될 것입니다.
{% asset_img 'ancient-garbage-collector-in-action.jpg' 'Garage Collector' %}


# Node.js 어플리케이션의 메모리 관리

모든 어플리케이션은 제대로 동작하려면 메모리가 필요합니다. 메모리 관리는 프로그램이 메모리를 요청할때 동적으로 메모리 영역을 할당하고 해제할 수 있는 방법을 제공합니다.

어플리케이션 레벨에서 메모리는 수동 혹은 자동으로 관리됩니다. 자동으로 메모리를 관리하는 경우는 일반적으로는 가비지 컬렉터를 사용합니다.

다음 코드는 ``C`` 언어에서 메모리를 수동으로 할당하는 방법을 보여줍니다.
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {

   char name[20];
   char *description;

   strcpy(name, "RisingStack");

   // memory allocation
   description = malloc( 30 * sizeof(char) );
	
   if( description == NULL ) {
      fprintf(stderr, "Error - unable to allocate required memory\n");
   } else {
      strcpy( description, "Trace by RisingStack is an APM.");
   }
   
   printf("Company name = %s\n", name );
   printf("Description: %s\n", description );

   // release memory
   free(description);
}
```
수동으로 메모리를 관리하는 경우에는 반드시 개발자가 더 이상 사용하지 않는 메모리를 해제를 해줘야 합니다. 이 방법은 개발자가 실수하는 경우 어플리케이션에 몇 가지 주요 버그를 만들어 내게 됩니다.
- **메모리 누수(Memory leaks)** - 메모리를 사용하고나서 해제하지 않으면 메모리 누수가 발생합니다.
- **Wild/dangling pointers** - 삭제된 객체의 포인터가 재사용되는 경우 발생합니다. 다른 데이터 구조를 덮어 쓰거나 중요한 정보를 읽을 때 심각한 보안 이슈가 발생할 수 있습니다.(역자주: 저도 dangling pointer에 대해서 잘 알지 못합니다. 상세한 정보는 [위키-허상 포인터](https://ko.wikipedia.org/wiki/%ED%97%88%EC%83%81_%ED%8F%AC%EC%9D%B8%ED%84%B0)를 참고해 주세요.)

**다행히도 Node.js는 가비지 컬렉터와 함께 제공되므로 메모리 할당을 수동으로 관리 할 필요가 없습니다.**


# 가비지 컬렉터 컨셉

가비지 컬렉터는 메모리를 자동으로 관리할 수 있는 방법입니다. 가비지 컬렉터(Garbage Collector 일반적으로 ``GC``라고 부릅니다.)가 하는 일은 사용되지 않는 객체(garbage)가 차지하고 있는 메모리를 회수하는 일입니다. 이 방법은 John McCarthy가 발명했으며 1959년 LISP에서 처음 사용되었습니다.

GC가 특정 객체가 더 이상 사용되지 않는다는 것을 알 수있는 방법은 해당 객체에 다른 객체들의 참조가 있는지를 통해 알 수 있습니다.

## GC 수행 전 메모리

아래 다이어그램은 참조가 있는 객체와 참조가 없는 객체를 보여주고 있습니다. 참조가 없는 객체는 GC 실행시 수집될 수 있는 대상입니다.

{% asset_img 'memory-state-before-node-js-garbage-collection.png' 'GC 수행전 메모리' %}

## GC 수행 후 메모리

GC가 수행되면 참조가 없는 객체는 삭제가 되고 해당 객체가 사용중이던 메모리는 회수됩니다.

{% asset_img 'memory-state-after-node-js-garbage-collection.png' 'GC 수행전 메모리' %}


# GC 사용시 얻을 수 있는 이점

- Wild/dangling pointers 버그를 예방할 수 있습니다.
- 이미 회수된 메모리 공간을 다시 회수하려고 시도하지 않습니다.
- 일부 메모리 누수를 막아줍니다.

물론 ``GC``가 당신의 모든 문제를 해결해 주지는 못합니다. 그리고 ``GC``는 메모리 관리를 위한 은총알이 아닙니다. 당신이 명심해야 할 사항들을 살펴보도록 하겠습니다.

## GC 사용시 명심해야 할 것들

- **성능에 미치는 영향(performance impact)** - ``GC`` 대상을 결정하기 위해서 ``GC``는 컴퓨팅 파워를 사용합니다.
- **예측할 수 없는 지연(unpredictable stalls)** - 최신의 ``GC``는 ``stop-the-world`` 수집을 피하려고 노력합니다.(역자주: 기본적으로 ``GC``를 수행하게 되면 해당 어플리케이션은 ``GC``가 끝날때 까지 멈추게 됩니다. 따라서 이런 문제를 해결하기 위해서 최신 ``GC``는 다양한 방법을 사용합니다.)


# Node.js의 GC와 메모리 관리 실습

실습을 통해서 알아봅시다.

## The Stack

스택은 지역변수와 힙에 있는 객체의 포인터 또는 어플리케이션의 흐름을 제어하기 위해 정의된 포인터를 가지고 있습니다.

다음 코드에서 ``a``와 ``b``는 스택에 생성될 것입니다.

```js
function add (a, b) {
  return a + b;
}

add(4, 5);
```

## The Heap

힙은 문자열이나 객체같은 참조형 객체를 저장하는데 사용됩니다.(역자주: 참조형 객체의 포인터는 스택에 생성됩니다.)

다음 코드에서 ``Car`` 객체는 힙에 생성될 것입니다.
```js
function Car (opts) {
  this.name = opts.name;
}

const LightningMcQueen = new Car({name: 'Lightning McQueen'});
```

이 후, 메모리는 다음과 같이 보입니다.
{% asset_img 'node-js-garbage-collection-first-step-object-placed-in-memory-heap.png' 'step 1' %}

``Car``를 좀 더 추가해 보면 메모리는 다음과 같이 보입니다.
```js
function Car (opts) {
  this.name = opts.name;
}

const LightningMcQueen = new Car({name: 'Lightning McQueen'});
const SallyCarrera = new Car({name: 'Sally Carrera'});
const Mater = new Car({name: 'Mater'});
```
{% asset_img 'node-js-garbage-collection-second-step-more-elements-added-to-the-heap.png' 'step 2' %}

만약 ``GC``가 지금 수행된다면 ``root``가 모든 객체를 참조하고 있기 때문에 아무것도 변하지 않을 것입니다.

약간의 파트를 추가해서 좀 더 흥미롭게 만들어 보겠습니다.
```js
function Engine (power) {
  this.power = power;
}

function Car (opts) {
  this.name = opts.name;
  this.engine = new Engine(opts.power);
}

let LightningMcQueen = new Car({name: 'Lightning McQueen', power: 900});
let SallyCarrera = new Car({name: 'Sally Carrera', power: 500});
let Mater = new Car({name: 'Mater', power: 100});
```
{% asset_img 'node-js-garbage-collection-assigning-values-to-the-objects-in-heap.png' %}

``Mater``에 ``Mater = undefined`` 처럼 다른 값을 할당하면 어떻게 보일까요?
{% asset_img 'node-js-garbage-collection-redefining-values.png' %}

원본 ``Mater`` 객체는 ``root``와의 참조가 끊어지게 됩니다. 따라서 다음 ``GC`` 수행시에 ``Mater`` 객체는 힙에서 해제될 것입니다.
{% asset_img 'node-js-garbage-collection-freeing-up-unreachable-object.png.png' %}

이제 우리는 ``GC``의 기본 동작 원리를 이해하고 ``V8``에서 어떻게 구현되었는지 살펴보겠습니다.

# Garbage Collection Methods

다음으로 넘어가기 전에 [how the Node.js garbage collection methods work](https://blog.risingstack.com/finding-a-memory-leak-in-node-js/) 이글을 먼저 읽기를 권장합니다.


## New Space and Old Space

힙은 ``New Space``, ``Old Space`` 두 개의 메인 영역을 가지고 있습니다.

``New Space``는 새로운 할당이 일어나는 곳입니다. 이곳은 ``GC``가 자주 일어나며 1 ~ 8MB의 사이즈를 가지고 있습니다. ``New Space``에 존재하는 객체를 ``Young Generation``이라고 합니다.

``Old Space``는 ``New Space``에서 ``GC``로 부터 살아남은 객체들이 이동하게 됩니다. ``Old Space``에 존재하는 객체를 ``Old Generation``이라고 합니다. ``Old Space``는 할당은 빠르지만 ``GC`` 비용이 비싸기 때문에 ``GC``가 자주 수행되지 않습니다.

## Young Generation

일반적으로 약 ~20%의 ``Young Generation``이 살아남아 ``Old Generation``이 됩니다. ``Old Space``에서는 가용한 메모리가 다 소진되면 ``GC``가 수행됩니다. 그래서 ``V8`` 엔진은 두 가지 다른 수집 알고리즘을 사용합니다.

## Scavenge and Mark-Sweep collection

``Scavenge`` 수집은 빠르고 ``Young Generation``에서 수행됩니다. 반면 ``Mark-Sweep``은 상대적으로 느리며 ``Old Generation``에서 수행됩니다.

# 실전 사례 학습 - The [Meteor](https://www.meteor.com/) Case

2013년 [Meteor](https://www.meteor.com/)의 창시자가 메모리 누수에 대한 사례를 발표했습니다. 문제가 되는 코드는 다음과 같습니다.
```js
var theThing = null;
var replaceThing = function () {
  var originalThing = theThing;
  var unused = function () {
    if (originalThing)
      console.log("hi");
  }
  theThing = {
    longStr: new Array(1000000).join('*'),
    someMethod: function () {
      console.log(someMessage);
    }
  };
};
setInterval(replaceThing, 1000);
```
> 클로저가 구현되는 일반적인 방법은 모든 함수 객체가 lexical scope를 나타내는 사전 스타일 객체에 대한 링크를 가지고 있는 것입니다.(역자주: lexical scope가 하나의 JSON 객체이며 클로저 함수가 그 객체에 대한 참조를 가지고 있다는 이야기 인것 같습니다.)
> 만약 ``replaceThing`` 내부에 정의된 두 함수 ``unused``와 ``someMethod``가 ``originalThing``을 실제로 사용한다면 ``originalThing``이 몇 번이 할당되더라도 두 함수가 같은 객체를 참조하고 있다는게 중요하므로 두 함수가 동일한 lexical environment를 공유합니다.
> 이제 크롬의 V8 자바 스크립트 엔진은 어떤 클로저에서도 사용되지 않으면 어휘 환경에서 변수를 유지할 수있을 만큼 스마트합니다. - from the [Meteor blog](https://blog.meteor.com/an-interesting-kind-of-javascript-memory-leak-8b47d2e7f156)
(역자주: 이 부분은 [원본글](https://blog.meteor.com/an-interesting-kind-of-javascript-memory-leak-8b47d2e7f156)을 직접 정독해야 이해가 가능합니다. [원본글](https://blog.meteor.com/an-interesting-kind-of-javascript-memory-leak-8b47d2e7f156)을 참고해 주세요.)

