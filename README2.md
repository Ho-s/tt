### 2가지 문제


#### 1. 잘못된 타입으로 인한 url undefined
app/loading/page.tsx 의 49line 에서 `matchedUser` 에는 socketId가 존재하지 않고, id가 존재함

as-is

```js
      setTimeout(() => router.push(`/chat?matchId=${matchedUser.socketId}`), 1500);
//    http://localhost:3000/chat?matchId=undefined로 redirect
```


to-be
```js
      setTimeout(() => router.push(`/chat?matchId=${matchedUser.id}`), 1500);
//    http://localhost:3000/chat?matchId=mzulm_7my42tB-EVAAAR로 정상 redirect
```




#### 2. 서버 단에서, 메시지 전송시의 event를 on 하고 있지 않음. (필요 코드 누락)

메시지 전송시 client는 send-message를 emit 하고 receive-message 를 on 하는데, (/app/chat/page.tsx)
서버단에서는 그 이벤트에 대한 handling을 하고 있지 않음 (lib/backendSocket.js)
관련해서 아래에 코드 추가 필요 (상세 코드는 직접 작성 필요)

```js
  io.on('connection', (socket) => {
//...

    socket.on('send-message', (data) => {
       io.to().emit('recieve-message')
    });
//...
})

```
