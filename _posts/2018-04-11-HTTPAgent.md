---
title: "node js 의 HTTP Agent"
categories:
- Node.js
excerpt: |
  에이전트는 Http clients 를 위해 연결을 유지시키고 재사용하도록 관리하는 일을 한다. 주어진 호스트와 포트에 대해 대기하고 있는 요청들을 유지하고 하나의 소캣 연결을 각각의 대기 요청이 다 끝날 때 까지 재사용한다. 그 이후에 파괴시키거나 풀에 넣어놓고 나중에 요청이 오면 다시 사용하게 된다. 파괴될 것인지 아니면 풀에 넣어두고 다시 사용할 것인지는 KeepAlive설정에 달렸다.
feature_text: |
  node js 의 HTTP Agent에 대해
---
# node js 의 HTTP Agent

* table of contents
{:toc .toc}


에이전트는 Http clients 를 위해 연결을 유지시키고 재사용하도록 관리하는 일을 한다. 주어진 호스트와 포트에 대해 대기하고 있는 요청들을 유지하고 하나의 소캣 연결을 각각의 대기 요청이 다 끝날 때 까지 재사용한다. 그 이후에 파괴시키거나 풀에 넣어놓고 나중에 요청이 오면 다시 사용하게 된다. 파괴될 것인지 아니면 풀에 넣어두고 다시 사용할 것인지는 KeepAlive설정에 달렸다. 

KeepAlive 를 설정해서 풀에 들어가 있는 경우 TCP 연결을 다시 사용할 수 있지만 지금은 Client 입장이라 만약 서버가 이 연결을 닫길 원하면 연결이 다시 사용되지않고 파괴된다. 이 경우 풀에서 제거되고 해당 호스트와 포트에 대해 새로운 HTTP 요청이 생기면 그때 다시 연결을 새로 만든다. 또한 서버는 한 소캣 연결에 대해서 여러개의 요청을 애초에 허락하지 않을 수도 있는데 이 때는 매 요청마다 연결이 새로 만들어지고 풀에 들어가지 않는다. Agent 는 요청을 계속 만들어 내지만 각각의 요청은 새로운 연결에서 전송되어진다. 

연결은 클라이언트 쪽에서나 서버쪽에서 둘다 가능한데 이때 풀에서 제거되어진다. 그리고 만약 요청이 더이상 없을 때 노드의 프로세스가 실행되지 않도록 하기위해서 풀에 있는 모든 사용되지 않았던 소캣들은 참조 해제 된다. (socket.unref())

만약 더이상 Agent 객체가 관리하는 Socket 들을 더이상 사용할 일이 없다면 Agent 객체를 destroy() 하는 것은 좋은 습관이다. 사용하지 않는 소캣은 OS 리소스를 소비하기 떄문이다. 

소캣은 close이벤트 또는 agentRemove이벤트를 발생시킬때 에이전트에서 제거된다. 그래서 만약 에이전트를 사용하지 않고 하나의 소캣을 내가 필요한 작업을 다 처리할 동안 열어놓고 바로 파괴하고 싶다면 아래와 같이 사용할 수 있다.

``` javascript
http.get(options, (res) => {
  // Do stuff
}).on('socket', (socket) => {
  socket.emit('agentRemove');
});
```

에이전트는 개별의 한 요청에서만 사용되게 할수도 있는데 아래와 같이 하면된다. 

``` javascript
http.get({
  hostname: 'localhost',
  port: 80,
  path: '/',
  agent: false  // create a new agent just for this one request
}, (res) => {
  // Do stuff with response
});
```

에이전트를 false 로 놓으면 Default 옵션값으로 생성된 하나의 에이전트가 이 하나의 요청에 대해서만 사용되어진다. 

> ***new Agent()*** 의 속성
> 
> keepAlive `boolean` 기본값: false TCP connection을 다시 맺을 필요 없이 재사용 할 수 있는 기능.
>
> keepAliveMsecs `number` 기본값: 1000ms. 한번 소캣 전송 후 이 소캣이 재사용 되기까지의 시간. keepAlive 일때만 가능
> 
> maxSockets `number` 기본값: Infinity. 이 에이전트가 만들 수 있는 최대 소캣 수.
> 
> maxFreeSockets `number` 기본값: 256. 사용하지 않은 채로 둘수 있는 최대 소캣 수. keepAlive 일때만 가능
> 


노드의 HTTP request에서 사용되어지는 기본 에이전트인 globalAgent 는 이 위의옵션의 디폴트 값으로 세팅 되어있다. 만약에 이 속성들 중에 하나라고 바꾸고 싶다면 http.Agent 인스턴스를 무조건 생성해야 한다. 

``` javascript
const http = require('http');
const keepAliveAgent = new http.Agent({ keepAlive: true });
options.agent = keepAliveAgent;
http.request(options, onResponseCallback);
```

http.globalAgent
이 글로벌 에이전트는 모든 HTTP 클라이언트 요청에 사용되어지는 디폴트 에이전트이다.  
