---
title: "EADDRNOTAVAIL 문제, CPU sy. 점유율 문제"
categories: 
- Node.js
excerpt: |
  Node js 에서 발생하는 EADDRNOTAVAIL 문제 점유율 문제에 대한 이유와 해결법입니다.
feature_text: |
  EADDRNOTAVAIL 문제, CPU sy. 점유율 문제에 대해서
feature_image: "https://picsum.photos/2560/600?random"
---

* table of contents
{:toc .toc}




### EADDRNOTAVAIL 문제
EADDRNOTAVAIL 문제는 해당 서버에서 다른 서버를 많이 호출하는 서버에서 발생한다. TCP는 즉 안정적으로 소캣을 닫고 파괴하기위해 graceful closure를 하는데 이런 우아한 종료로 인해 TIME_WAIT 을 사용하게된다. 그래서 소캣 port 가 사용이 끝난 뒤 파괴될 때마다 TIME_WAIT 에 걸리게되는데 EADDRNOTAVAIL는 이런 상태의 포트가 많이 쌓이게 되어서 발생하는 문제이다. 

### CPU sy. 점유율 문제
위와 같은 맥락에서 계속 port를 맺고 끊기 바쁘기 때문에 cpu 의 system 영역(커널영역)의 점유율이 비정상적으로 치솟는 것이다.

## 해결방법


### node 의 Http.Agent 란?
http agent 는 클라이언트에 대한 포트 연결 지속성 및 포트 재사용을 관리한다. 주어진 포트에 대한 요청 대기열을 유지하고 관리하는데 이 대기열이 빌때 까지 각각의 포트에 대해 소캣 연결을 재사용하게 한다. 이렇게 연결을 주고 받음으로 인해 사용되었던 소캣은 파괴되거나 요청이 다시 생기면 재사용되도록 pool 에서 관리가 된다. 여기서 KeepAlive 속성을 true 로 해 놓으면 소캣 사용이 끝나도 파괴되지 않고 살아있게 되고 재사용 된다.
KeepAlive 가 false로 되어있다면 모든 요청에 대해 연결을 항상 파괴 후 다시 만들어야 하고 그로인한 cpu sy.(system 영역)비용이 발생한다. 

agent 는 아래와 같이 만들 수 있다.
``` javascript
const http = require('http');
const keepAliveAgent = new http.Agent({ keepAlive: true });
options.agent = keepAliveAgent;
http.request(options, onResponseCallback);
```
만약 axios 를 사용하는 경우에는 아래와 같이 설정이 가능하다.
``` javascript
const http = require('http');
let axiosInstant = axios.create({
    baseURL: "urlPath",
    httpAgent: new http.Agent({keepAlive: true, keepAliveMsecs: 1000, maxSockets: 100})
});
```
> NOTE: ***http.Agent*** 를 생성할 때 설정할수 있는 ***컨피그 속성***은 아래와 같다. 위의 예제 코드의 경우 그 중 3가지 속성에 대해서만 정의한 예제이다.
>
> keepAlive: boolean (default: false)
>
> keepAliveMsecs: number (default: 1000ms)
>
> maxSockets: number (default: Infinity)
>
> maxFreeSockets: number (default: 256) 비어있는 상태로 열어 둘 수 있는 최대 소캣 수


### 번외
#### Core 수가 많은 서버(실험: 8core, 8gb)에서 Keep Alive 설정을 하지 않았을 때
일정 시간이 지나면 이유없이 급격히 TPS가 떨어진다. 

![Alt text](https://monosnap.com/image/8oooY0SRpUNRWwjrwwyinsEH6IFbaF)
