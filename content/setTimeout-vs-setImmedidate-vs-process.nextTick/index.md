---
emoji: ⏲️
title: setTimeout(), setImmedidate(), process.nextTick()
date: '2022-02-06 17:00:00'
author: reviday
tags: blog javscript node.js
categories: javascript node.js
---

# 🎯 목표
동작을 지연시키는 것으로만 알고 있었던 이 세 가지 함수 ```setTimeout()```, ```setImmedidate()```, ```process.nextTick()```에 대해 알아보고 실행 시점에 대한 차이를 간단하게나마 알아보고자 한다.

<br />

# 📄 개념
[Node.js 공식 문서](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)를 확인하면 정의는 아래와 같고, 나름대로 개념을 정리(<span style="color:grey">~~짜집기~~</span>)해보았다.

<br />

## 1. setTimeout()
> ```setTimeout()``` schedules a script to be run after a minimum threshold in ms has elapsed.
- <strong style="color:rgba(217, 115, 13, 1);">지정된 시간(ms) 이후</strong>에 함수 또는 지정된 코드를 실행하는 타이머를 설정한다.

<br />

## 2. setImmedidate()
> ```setImmediate()``` is designed to execute a script once the current poll phase completes.
- <strong style="color:rgba(217, 115, 13, 1);">현재 폴링</strong> 단계가 완료되면 실행된다.
- 작업을 비동기적으로 실행한다는 면에서는 process.nextTick()과 유사하지만, setImmedidate()는 이미 큐에 있는 <strong style="color:rgba(217, 115, 13, 1);">I/O 이벤트들의 뒤에 대기</strong>하게 된다는 차이가 있다.

<br />

## 3. process.nextTick()
> ```process.nextTick()``` fires immediately on the same phase.
- 현재 진행 중인 작업의 완료 시점 뒤로 함수의 실행을 지연시킨다(즉, 같은 phase 내에서 즉시 실행된다).
- 해당 함수로 지연된 콜백은 <strong style="color:rgba(144, 101, 176, 1);">마이크로 태스크</strong>라고 불리며, 이미 큐에 있는 <strong style="color:rgba(217, 115, 13, 1);">I/O 이벤트들 앞에 위치</strong>하게 되어 현재의 작업이 완료된 후에 바로 실행된다.

<br />

# ⛳ 결론부터 간단하게
1. 일반적으로 ```setTimeout(cb, 0)``` < ```setImmediate()``` < ```process.nextTick()``` 순으로 먼저 실행된다.
2. ```setTimeout(cb, 0)```, ```setImmediate()```의 실행순서는 <strong style="color:rgba(217, 115, 13, 1);">I/O 사이클 및 프로세스 성능</strong>에 따라 변할 수 있다.  
3. ```process.nextTick()```를 사용하면 현재 작업 바로 뒤로 지연되며, 해당 함수로 지연된 콜백은 <strong style="color:rgba(144, 101, 176, 1);">마이크로 태스크</strong>라고 불린다. 

<br />

# 🧑🏻‍💻 코드로 확인해보자

<br />

## 1️⃣ setTimeout(func, delay)에서 delay가 0ms라고 0ms 후에 실행되지 않는다.
Node.js 소스의 [node/lib/timers.js](https://github.com/nodejs/node/blob/master/lib/timers.js#L140)를 살펴보면 ```setTimeout()``` 함수를 확인할 수 있고,
```javascript
function setTimeout(callback, after, arg1, arg2, arg3) {

	...

  const timeout = new Timeout(callback, after, args, false, true);
  insert(timeout, timeout._idleTimeout);

  return timeout;
}
```
이곳에서 사용된  [Timeout()](https://github.com/nodejs/node/blob/b2edcfee46097fe8e0510a455b97d5c6d0cac5ec/lib/internal/timers.js#L167)의 정의를 확인해보면 다음과 같이 정의되어 있다.
```javascript
function Timeout(callback, after, args, isRepeat, isRefed) {
  after *= 1; // Coalesce to number or NaN
  if (!(after >= 1 && after <= TIMEOUT_MAX)) {
    if (after > TIMEOUT_MAX) {
      process.emitWarning(`${after} does not fit into` +
                          ' a 32-bit signed integer.' +
                          '\nTimeout duration was set to 1.',
                          'TimeoutOverflowWarning');
    }
    after = 1; // Schedule on next tick, follows browser behavior
  }

...

}
```
즉, <strong style="color:rgba(217, 115, 13, 1);">delay를 0ms으로 설정하여도 1ms로 설정되어 동작</strong>한다.

<br />

## 2️⃣ setTimeout() vs setImmedidate() 
<strong style="color:rgba(217, 115, 13, 1);">I/O 사이클 내에 있지 않은</strong> 아래 스크립트를 실행하게 되면 두 타이머가 실행되는 순서는 <strong style="color:rgba(217, 115, 13, 1);">프로세스의 성능</strong>에 의해 제한되기 때문에 <strong style="color:rgba(217, 115, 13, 1);">결과가 일정하지 않다.</strong> 그 이유는 위에서 설명한 내용(<a href="#-개념" style="color:grey">**개념**</a>)에 있다. 

```javascript
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});

// Output 
timeout
immediate

// Output 
immediate
timeout
```
하지만 <strong style="color:rgba(217, 115, 13, 1);">I/O 사이클 내</strong>에 위치하게 되면 <strong style="color:rgba(217, 115, 13, 1);">항상 setImmediate()가 먼저 실행</strong>된다.
```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});

// Output 
immediate
timeout
```

<br />

## 3️⃣ setTimeout() vs setImmedidate() 
이미 개념 설명 부분에서 차이를 설명하였기에 누가 더 빠른 동작을 보일지 자명하지만, 코드로 확인해보기 위하여 [스택오버플로우](https://stackoverflow.com/questions/17502948/nexttick-vs-setimmediate-visual-explanation)에서 가져온 예시를 들어 살펴보도록 하자.

### setImmediate() 
```javascript
setImmediate(function A() {
  setImmediate(function B() {
    console.log(1);
    setImmediate(function D() { console.log(2); });
    setImmediate(function E() { console.log(3); });
  });
  setImmediate(function C() {
    console.log(4);
    setImmediate(function F() { console.log(5); });
    setImmediate(function G() { console.log(6); });
  });
});

setTimeout(function timeout() {
  console.log('TIMEOUT FIRED');
}, 0);

/* 위에서 언급했다 시피, I/O 사이클 내에 있지 않으므로 실행할 때마다 결과가 달라질 것이다. */
// TIMEOUT FIRED 1 4 2 3 5 6
// OR
// 1 4 TIMEOUT FIRED 2 3 5 6
```

### process.nextTick()
```nextTick()``` 콜백은 항상 현재 코드 실행이 완료된 직후와 이벤트 루프로 돌아가기 전에 실행되기 때문에 이벤트 루프 이후에 실행되는 ```setTimeout()```의 콜백은 항상 마지막에 출력된다. 가령, ```setTimeout()``` 코드의 위치를 위로 올린다고 하여도 결과는 항상 같다.
```javascript
process.nextTick(function A() {
  process.nextTick(function B() {
    console.log(1);
    process.nextTick(function D() { console.log(2); });
    process.nextTick(function E() { console.log(3); });
  });
  process.nextTick(function C() {
    console.log(4);
    process.nextTick(function F() { console.log(5); });
    process.nextTick(function G() { console.log(6); });
  });
});

setTimeout(function timeout() {
  console.log('TIMEOUT FIRED');
}, 0);

/* 항상 nextTick()의 결과가 먼저 출력된다. */
// 1 4 2 3 5 6 TIMEOUT FIRED
```

위 비교로 알 수 있다 시피, ```setTimeout()``` 기준으로 항상 그 앞에서 실행되는 ```process.nextTick()```은 ```setImmediate()``` 보다 먼저 실행된다는 것을 알 수 있다.


<br />

# 참고한 사이트
1. [Node.js 공식 문서](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
2. [StackOverFlow](https://stackoverflow.com/questions/17502948/nexttick-vs-setimmediate-visual-explanation)
3. [[NodeJS] Event-Loop Part 2 : setTimeout() vs setImmediate() vs process.nextTick()](https://medium.com/@rpf5573/nodejs-event-loop-part-2-settimeout-vs-setimmediate-vs-process-nexttick-70ba2a9f0895) - **Byeongin Yoon**

<p style="color:grey">그 외 참고한 블로그들이 더 있는데, 노션에 정리했던 부분을 블로그 글로 작성해본 것이여서 출처가 남아있지 않네요. 😭😭😭 </p>