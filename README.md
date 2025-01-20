# 문제 및 해결

## 1. 모듈 스코프 변수 사용

frontendSocket.js 파일에서 on 으로 이벤트를 받을 경우, nextjs 초기화 과정에서 의도한 순서대로 렌더링이 작동하지 않을 수 있습니다.
따라서 컴포넌트 내부, client-side에서 윈도우가 렌더링 된 이후 초기화 순서를 따르는 게 좋습니다.

### as-is

```ts
//frontendSocket.js
//...기존 코드
```

### to-be (편의를 위해 주석은 삭제)

```ts
//frontendSocket.js
"use client";

import { io } from "socket.io-client";

const socket = io(
  process.env.NEXT_PUBLIC_SOCKET_URL || "http://localhost:3000",
  {
    path: "/socket.io/",
    transports: ["websocket"],
    reconnection: true,
    reconnectionAttempts: 5,
  }
);

export default socket;
```

-> page 내에서 초기화 (layout에서 초기화 하고 ,isConnected를 전역으로 상태관리 하여도 괜찮을듯 싶습니다.)

```ts
//loading/page.tsx

export default function Loading() {
  //...
  const [isConnected, setIsConnected] = useState(false);
  //...

  useEffect(() => {
    const onConnect = () => {
      setIsConnected(true);
      console.log(`Connected to the WebSocket server! Socket ID: ${socket.id}`);
    };

    const onDisconnect = (reason: Socket.DisconnectReason) => {
      setIsConnected(false);
      console.warn(`Disconnected from the WebSocket server: ${reason}`);
    };
    if (socket.connected) {
      onConnect();
    }

    socket.on("connect", onConnect);
    socket.on("disconnect", onDisconnect);

    const onMatchFound = (matchedUser: any) => {
      console.log("Match found:", matchedUser);
      setStatus("Match found! Redirecting...");
      setTimeout(
        () => router.push(`/chat?matchId=${matchedUser.socketId}`),
        1500
      );
    };

    socket.on("match-found", onMatchFound);

    const onNoMatch = () => {
      console.log("No match found");
      setStatus("No match found yet. Retrying...");
    };

    socket.on("no-match", onNoMatch);

    return () => {
      socket.off("connect", onConnect);
      socket.off("disconnect", onDisconnect);

      socket.off("match-found", onMatchFound);
      socket.off("no-match", onNoMatch);
    };
  }, []);

  // ...
  useEffect(() => {
    if (!isConnected) return;
    const storedGender = sessionStorage.getItem("selectedGender");
    const storedSexuality = sessionStorage.getItem("selectedSexuality");

    if (view === "give") {
      setGender(storedGender || "");
      setSexuality(storedSexuality || "");

      // Emit WebSocket event for givers
      console.log("give");
      const res = socket.emit("find-match", {
        userGender: storedGender,
        userSexuality: storedSexuality,
        desiredGender: "",
        desiredSexuality: "",
      });
      console.log(res);
    } else if (view === "get") {
      console.log("get");
      setDesiredGender(storedGender || "");
      setDesiredSexuality(storedSexuality || "");

      // Emit WebSocket event for getters
      socket.emit("find-match", {
        userGender: "",
        userSexuality: "",
        desiredGender: storedGender,
        desiredSexuality: storedSexuality,
      });
    }
  }, [view, isConnected]);

  //...
}
```

## 2. 중복된 서버 event 코드

`Server.js`에서 사용된 socket server는 `lib/backendSocket.js` 이나, 이벤트를 받는 곳은 `pages/api/socket.js` 에서 받아 제대로 이벤트 송/수신이 되고 있지 않았습니다. 하여, `pages/api/socket.js`을 삭제하고, `lib/backendSocket.js`에 코드를 통합하였습니다.

> ps. 아시다시피 socket 이벤트의 경우 `클라이언트 -> 서버`, `서버 -> 클라이언트`의 로직을 따라주셔야합니다.

### as-is

```ts
//api/socket.js
// ...기존 코드
```

```ts
//backendSocket.js
//...기존 코드
```

### to-be

```ts
//api/socket.js (Removed)
```

```ts
///backendSocket.js
const { Server } = require("socket.io");
const userSockets = new Map();

function initializeSocket(server) {
  const io = new Server(server, {
    cors: {
      origin: "http://localhost:3000",
      methods: ["GET", "POST"],
    },
  });

  io.on("connection", (socket) => {
    console.log(`Client connected: ${socket.id}`);

    socket.on("find-match", (data) => {
      console.log("Find match event received:", data);

      userSockets.set(socket.id, {
        id: socket.id,
        ...data,
      });

      const match = findMatch(socket.id, data);
      if (match) {
        io.to(socket.id).emit("match-found", match);
        io.to(match.id).emit("match-found", userSockets.get(socket.id));

        userSockets.delete(socket.id);
        userSockets.delete(match.id);
      } else {
        socket.emit("no-match");
      }
    });
  });

  return io;
}

function findMatch(socketId, userData) {
  console.log(userSockets);
  for (const [id, user] of userSockets) {
    if (
      id !== socketId &&
      user.desiredGender === userData.userGender &&
      user.desiredSexuality === userData.userSexuality &&
      user.userGender === userData.desiredGender &&
      user.userSexuality === userData.desiredSexuality
    ) {
      return user;
    }
  }
  return null;
}

module.exports = initializeSocket;
```

# 결과화면

<img width="1574" alt="image" src="https://github.com/user-attachments/assets/5a569dc5-75f9-46d1-9bd5-05c9cc4acb71" />

#### 수정 파일 목록

<img width="239" alt="image" src="https://github.com/user-attachments/assets/b77aaf39-9235-454d-9ba1-917632902894" />
