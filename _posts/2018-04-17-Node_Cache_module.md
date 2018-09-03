---
title: "Node js 에서 axios 호출에 대한 Cache 적용"
categories: 
- Node.js
excerpt: |
  프로덕션 환경에서 최적의 성능을 내기위해 꼭 해줘야 하는 작업 중 하나가 요청 결과를 캐싱하는 일이다.
  같은 응답결과를 원하는 요청의 경우에는 가지고 있는 즉, 캐시에 저장된 이전 호출의 결과를 리턴해서 성능향상을 하는 것이다.
feature_text: |
  axios 로 호출되는 결과를 캐싱하는 모듈 정리
feature_image: "https://picsum.photos/2560/600?random"
---

* table of contents
{:toc .toc}



프로덕션 환경에서 최적의 성능을 내기위해 꼭 해줘야 하는 작업 중 하나가 요청 결과를 캐싱하는 일이다.
같은 응답결과를 원하는 요청의 경우에는 가지고 있는 즉, 캐시에 저장된 이전 호출의 결과를 리턴해서 성능향상을 하는 것이다.

## Node js 용 Cache 모듈 비교
### axios-extensions
axios-extensions 안에는 작게 두개의 모듈이 들어있다. cacheAdapterEnhancer 와 throttleAdapterEnhancer 인데 

cacheAdapterEnhancer: axios 로 호출된 결과를 캐싱하여 다음 호출 부턴 저장된 결과가 리턴됨. 
throttleAdapterEnhancer: 원하는 시간(임계값) 이 지나면 다시 진짜 호출을 해서 캐싱한다. 즉 Max Age 와 같은 개념.

사용법
``` javascript
import axios from 'axios';
import { cacheAdapterEnhancer } from 'axios-extensions';

const http = axios.create({
	baseURL: '/',
	headers: { 'Cache-Control': 'no-cache' },
	// cache will be enabled by default
	adapter: cacheAdapterEnhancer(axios.defaults.adapter, true)
});
```

``` javascript
import axios from 'axios';
import { throttleAdapterEnhancer } from 'axios-extensions';

const http = axios.create({
	baseURL: '/',
	headers: { 'Cache-Control': 'no-cache' },
	adapter: throttleAdapterEnhancer(axios.defaults.adapter, 2 * 1000)
});
```

cacheAdapterEnhancer 와 throttleAdapterEnhancer 가 차이가 없어보이지만 약간 목적이 어떤것에 포커싱을 둘 것인가에 따라 둘 다 선택할 수도, 둘 중 하나만 선택해서 사용할수도 있다.
cacheAdapterEnhancer의 경우 cacheAdapterEnhancer(`adapter`, `cacheEnabledByDefault = false`, `enableCacheFlag = 'cache'`, `defaultCache = new LRUCache({ maxAge: FIVE_MINUTES })`) 의 기본값을 가지고 생성되므로 LRUcache 또는 캐시 플래그를 다르게 설정하고 싶을때 이용가능하다. 
throttleAdapterEnhancer 의 경우 어떤것도 신경쓰고싶지 않고 "원하는 시간동안 한번 캐싱 결과를 갱신한다" 에 초점을 맞춘 매소드이다. throttleAdapterEnhancer(`adapter`, `threshold = 1000`, `cacheCapacity = 10`) 로 쓰면 된다.


### axios-cache-adapter

**setupCache**
캐시 어댑터 인스턴스를 만들어 사용할 수 있다. 아래는 그 어댑터의 가능한 설정. setupCache()의 리턴값은 이 캐시 어댑터 객체다.
* maxAge {Number}: 각 요청에 대해 캐싱할 기간(시간)  defaults: 15 minutes (15 * 60 * 1000)
* limit {Number}: 캐싱 할 요청의 갯수 (LIFO), default: no limit
* store {Object}: localForage 인스턴스, defaults: in memory store
* key {Mixed}: 캐싱의 키로 쓰일 것. Defaults:  req.url
* exclude {Object}: 캐싱에 포함시키지 않을 것
* filter {Function}: 요청받을 매소드와 캐시에 포함시키지 않을 것, defaults: null
* query {Boolean}: query를 담고있는 모든 요청이 캐싱되지 않는다. defaults: true
* paths {Array}: URL이 해당 해당 정규식에 대응한다면 캐싱되지 않는다. defaults: []
* clearOnStale {Boolean}: Stale 상태일때. 즉, maxAge 가 지나면 값을 지운다. default: true
* clearOnError {Boolean}: 에러가 나면 값을 지운다
* debug {Boolean}: 콘솔에 로그를 찍는다. defaults: false


**setup**
캐시를 가지는 axios 인스턴스를 얻을 수 있다.


### axios-cache-plugin
기본으로는 filter 를 추가하지 않으면 모듈을 추가하고 설정했다 해도 캐싱되지 않는다. 아래와 같은 속성을 설정 가능하고 사용시 필터로 api를 추가하면 된다.
``` javascript
let http = wrapper(axios, {
  maxCacheSize: 15,  // cached items amounts. if the number of cached items exceeds, the earliest cached item will be deleted. default number is 15.
  ttl: 60000, // time to live. if you set this option the cached item will be auto deleted after ttl(ms).
  excludeHeaders: true // should headers be ignored in cache key, helpful for ignoring tracking headers
})


http.__addFilter(/getItemInfoByIdsWithSecKill/)
 
http({
  url: 'http://example.com/item/getItemInfoByIdsWithSecKill',
  method: 'get',
  params: {
    param: JSON.stringify({
      debug_port: 'sandbox1'
    })
  }
})
 
```

maxCacheSize 가 10 이면 10개를 넘었을 경우 가장 초기에 추가했던 api 가 캐시되지 않는다.

