
<template>
  <section>
    <div id="roomButton">
      <input id="roomIdInput" v-model="roomId" type="text" placeholder="room id" />

      <button :disabled="!ready || creating || joined" @click="createVideoRoom">
        CreateVideoRoom
      </button>

      <button :disabled="!ready || joining || joined" @click="joinRoom">
        JoinRoom
      </button>

      <button :disabled="!ready || !joined || sharing" @click="startScreenShare">
        화면 공유
      </button>

      <button :disabled="!ready || !joined || !sharing" @click="stopScreenShare">
        공유 종료
      </button>
    </div>

    <div id="videoBox">
      <div v-for="id in userIds" :key="id" class="multiVideo">
        <p class="userLabel">{{ id }}</p>

        <div class="videoRow">
          <div class="videoCol">
            <p class="subLabel">CAM</p>
            <video :id="`camVideo-${id}`" autoplay playsinline></video>
          </div>

          <div class="videoCol">
            <p class="subLabel">SCREEN</p>
            <video :id="`screenVideo-${id}`" autoplay playsinline></video>
            <canvas :id="`screenCanvas-${id}`" class="screenCanvas"></canvas>
          </div>
        </div>
      </div>
    </div>

    <!-- ✅ 채팅창: joined 이후에만 표시, 전송버튼 없음(Enter 전송), roomId 출력 없음 -->
    <div v-if="joined" id="chatBox">
      <div class="chatList" ref="chatListEl">
        <div style="font-family: Arial, Helvetica, sans-serif;">CHATBOX</div>
        <div
          v-for="(m, i) in chatMessages"
          :key="i"
          class="chatItem"
          :class="{ mine: m.mine }"
        >
          <div class="chatMeta">{{ m.user }}</div>
          <div class="chatBubble">{{ m.message }}</div>
        </div>
      </div>

      <div class="chatInputRow">
        <input
          class="chatInput"
          v-model="chatMessage"
          type="text"
          placeholder="메시지를 입력하고 Enter로 전송"
          @compositionstart="isComposing = true"
          @compositionend="isComposing = false"
          @keydown.enter.prevent="onEnterSend"
        />
      </div>
    </div>

    <div id="printBox">
      <p v-for="(l, i) in logs" :key="i">[{{ l.type }}] {{ l.text }}</p>
    </div>
  </section>
</template>

<script setup>
import { onBeforeUnmount, onMounted, ref, nextTick } from "vue";

function createKT() {
  const Con = new Knowledgetalk();
  if (!Con) throw new Error("Knowledgetalk SDK 로딩 안됨");
  return Con;
}

const cpCode = "KP-20200101-01";
const authKey =
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoi64Kg66as7KeA7Y-s7J247Yq4IiwibWF4VXNlciI6IjUwMDAwMDAwIiwic3RhcnREYXRlIjoiMjAyMC0wMS0wMVQwNjo0NzowMC4wMDBaIiwiZW5kRGF0ZSI6IjIwMzAtMTItMzFUMDY6NDc6MDAuMDAwWiIsImF1dGhDb2RlIjoiS1AtMjAyMDAxMDEtMDEiLCJjb21wYW55Q29kZSI6IkxJQy0wMSIsImlhdCI6MTU4NzUzODExNH0.73A0UiiMHJeIS8pIgoN4DfEWT4QCsMnXkO4uUdnfbYI";

const roomId = ref("");
const ready = ref(false);
const joined = ref(false);
const creating = ref(false);
const joining = ref(false);
const sharing = ref(false);

const userIds = ref([]);
const logs = ref([]);

let kt = null;
let presenceHandler = null;

let localCamStream = null;
let localScreenStream = null;

let publishedCamOnce = false;

// Group: feeds 기반으로 subscribeVideo 중복 방지 (cam:xxx / screen:xxx)
const subscribed = new Set();

/** ✅ Chat */
const chatMessage = ref("");
const chatMessages = ref([]); // { user, message, mine }
const chatListEl = ref(null);
const isComposing = ref(false);

const scrollChatToBottom = async () => {
  await nextTick();
  const el = chatListEl.value;
  if (el) el.scrollTop = el.scrollHeight;
};

const onEnterSend = () => {
  if (isComposing.value) return; // ✅ IME 조합 중 Enter 전송 방지
  sendChat();
};

const sendChat = async () => {
  const msg = (chatMessage.value || "").trim(); // ✅ 전송시에만 trim
  if (!msg) return;
  if (!joined.value) return;

  const rid = kt?.getRoomId?.() || roomId.value.trim();
  if (!rid) return;

  const res = await kt.chat(msg, rid);
  if (res?.code !== "200") {
    alert("chat failed!");
    return;
  }

  const me = kt.getUserId();
  chatMessages.value.push({ user: me, message: msg, mine: true });
  chatMessage.value = "";
  await scrollChatToBottom();
};

const log = (type, obj) => {
  const text = JSON.stringify(JSON.parse(JSON.stringify(obj)));
  logs.value.push({ type, text });
};

const ensureUser = (id) => {
  if (!id) return;
  if (!userIds.value.includes(id)) userIds.value.push(id);
};

const removeUser = (id) => {
  userIds.value = userIds.value.filter((x) => x !== id);
  subscribed.delete(`cam:${id}`);
  subscribed.delete(`screen:${id}`);
};

const setCamStream = async (id, stream) => {
  ensureUser(id);
  await nextTick();
  const el = document.getElementById(`camVideo-${id}`);
  if (el) el.srcObject = stream;
};

const setScreenStream = async (id, stream) => {
  ensureUser(id);
  await nextTick();
  const el = document.getElementById(`screenVideo-${id}`);
  if (!el) return;
  el.srcObject = null;
  await nextTick();
  el.srcObject = stream;
};

const clearScreenVideo = async (id) => {
  await nextTick();
  const el = document.getElementById(`screenVideo-${id}`);
  if (el) el.srcObject = null;
};

const getMyCanvas = () => {
  const me = kt?.getUserId?.();
  if (!me) return undefined;
  return document.getElementById(`screenCanvas-${me}`) || undefined;
};

const startLocalCamIfNeeded = async () => {
  if (localCamStream) return localCamStream;

  localCamStream = await navigator.mediaDevices.getUserMedia({
    video: { width: 640, height: 380 },
    audio: false
  });

  const me = kt.getUserId();
  await setCamStream(me, localCamStream);
  return localCamStream;
};

const publishMyCamIfNeeded = async () => {
  if (!joined.value) return;
  if (publishedCamOnce) return;

  await startLocalCamIfNeeded();
  const ok = await kt.publishVideo("cam", localCamStream);
  if (!ok) {
    alert("publishVideo(cam) failed!");
    return;
  }
  publishedCamOnce = true;
};

const subscribeFeed = async (feed) => {
  if (!feed?.id || !feed?.type) return;

  const key = `${feed.type}:${feed.id}`;
  if (subscribed.has(key)) return;
  subscribed.add(key);

  try {
    const stream = await kt.subscribeVideo(feed.id, feed.type);

    if (feed.type === "cam") {
      await setCamStream(feed.id, stream);
    } else if (feed.type === "screen") {
      await setScreenStream(feed.id, stream);
    }
  } catch (e) {
    subscribed.delete(key);
    console.error("subscribeVideo failed", feed, e);
  }
};

onMounted(async () => {
  try {
    kt = createKT();

    const result = await kt.init(cpCode, authKey);
    if (result.code !== "200") return alert("init failed!");
    ready.value = true;

    presenceHandler = async (event) => {
      const msg = event.detail;
      log("receive", msg);

      switch (msg.type) {
        case "join": {
          const uid = msg?.user?.id || msg?.user?.userId || msg?.user;
          if (uid) ensureUser(uid);
          break;
        }

        case "leave": {
          const uid =
            typeof msg.user === "string"
              ? msg.user
              : msg?.user?.id || msg?.user?.userId;

          if (uid) {
            removeUser(uid);
            await clearScreenVideo(uid);
          }
          break;
        }

        // ✅ Group에서는 publish 이벤트에 cam/screen feed가 같이 오므로 screen type을 따로 처리할 필요가 거의 없음
        case "publish": {
          const feeds = msg.feeds || [];
          for (const feed of feeds) {
            if (feed?.id) ensureUser(feed.id);
            await subscribeFeed(feed);
          }
          break;
        }

        case "shareStop": {
          const sender = msg.sender || msg.user;
          if (sender) {
            subscribed.delete(`screen:${sender}`);
            await clearScreenVideo(sender);
          }
          break;
        }

        case "chat": {
          const sender = msg.user;
          const text = msg.message;
          const me = kt?.getUserId?.();

          // 서버가 에코를 보내는 경우 중복 방지
          if (sender === me) break;

          chatMessages.value.push({ user: sender, message: text, mine: false });
          await scrollChatToBottom();
          break;
        }

        default:
          break;
      }
    };

    kt.addEventListener("presence", presenceHandler);
  } catch (e) {
    console.error(e);
    alert(String(e?.message || e));
  }
});

onBeforeUnmount(async () => {
  try {
    if (kt && presenceHandler) kt.removeEventListener("presence", presenceHandler);
  } catch (_) {}

  try {
    if (localCamStream) {
      localCamStream.getTracks().forEach((t) => t.stop());
      localCamStream = null;
    }
  } catch (_) {}

  try {
    if (localScreenStream) {
      localScreenStream.getTracks().forEach((t) => t.stop());
      localScreenStream = null;
    }
  } catch (_) {}
});

const createVideoRoom = async () => {
  creating.value = true;
  try {
    const res = await kt.createVideoRoom();
    if (res.code !== "200") return alert("createVideoRoom failed!");
    roomId.value = res.roomId;
    await joinRoom(res.roomId);
  } finally {
    creating.value = false;
  }
};

const joinRoom = async (overrideRoomId) => {
  const ridFromArg =
    typeof overrideRoomId === "string" ? overrideRoomId.trim() : "";

  let rid = ridFromArg || roomId.value.trim();

  if (!rid) {
    const input = prompt("입장할 roomId를 입력하세요.");
    if (!input) return;
    rid = input.trim();
    roomId.value = rid;
  }

  joining.value = true;
  try {
    const roomData = await kt.joinRoom(rid);
    if (roomData.code !== "200") return alert("joinRoom failed!");

    joined.value = true;

    ensureUser(kt.getUserId());
    await publishMyCamIfNeeded();

    const members = roomData.members || {};
    for (const memberId in members) ensureUser(memberId);
  } finally {
    joining.value = false;
  }
};

const startScreenShare = async () => {
  try {
    sharing.value = true;

    await kt.shareStop().catch(() => {});

    if (localScreenStream) {
      localScreenStream.getTracks().forEach((t) => t.stop());
      localScreenStream = null;
    }

    localScreenStream = await navigator.mediaDevices.getDisplayMedia({ video: true });
    const [track] = localScreenStream.getVideoTracks();
    if (track) track.onended = () => stopScreenShare();

    const me = kt.getUserId();
    await setScreenStream(me, localScreenStream);

    const canvas = getMyCanvas();
    const res = await kt.screenStart(localScreenStream, undefined, canvas);

    if (res?.code !== "200") {
      sharing.value = false;
      alert("screenStart failed!");
    }
  } catch (e) {
    console.error(e);
    sharing.value = false;
    alert(String(e?.message || e));
  }
};

const stopScreenShare = async () => {
  sharing.value = false;

  await kt.shareStop().catch(() => {});

  if (localScreenStream) {
    localScreenStream.getTracks().forEach((t) => t.stop());
    localScreenStream = null;
  }

  const me = kt?.getUserId?.();
  if (me) {
    subscribed.delete(`screen:${me}`);
    await clearScreenVideo(me);
  }
};
</script>

<style>
/* =========================
   공통 입력 / 버튼 영역
========================= */
#roomIdInput {
  padding: 10px;
  width: 220px;
  margin: 5px;
  border: 2px darkslategray solid;
  border-radius: 10px;
}

#roomButton {
  display: flex;
  align-items: center;
  gap: 6px;
}

/* =========================
   비디오 카드 레이아웃
========================= */
#videoBox {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

/* 사용자 카드 */
.multiVideo {
  border: 1px solid #ddd;
  border-radius: 12px;
  padding: 10px;
}

/* 사용자 ID */
.userLabel {
  margin: 0 0 8px;
  font-weight: 700;
}

/* CAM / SCREEN 2열 배치 */
.videoRow {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

/* =========================
   핵심: SCREEN 영역 레이아웃 고정
========================= */
.videoCol {
  position: relative;
  overflow: hidden;            /* ✅ SDK가 canvas 여러 개 붙여도 높이 증가 차단 */
}

/* 섹션 라벨 */
.subLabel {
  margin: 0 0 6px;
  font-size: 12px;
  opacity: 0.8;
}

/* 모든 video 기본 설정 */
#videoBox video {
  display: block;              /* ✅ inline baseline 여백 제거 */
  width: 100%;
  border-radius: 10px;
  background: #000;
}

/* =========================
   SCREEN video 높이 고정 (세로 확장 방지)
========================= */
video[id^="screenVideo-"] {
  height: 320px;               /* ✅ 화면 공유 최대 출력 높이 */
  max-height: 320px;
  object-fit: contain;         /* 전체 화면 유지 (cover로 바꾸면 꽉 참) */
}

/* =========================
   SCREEN 오버레이 캔버스
   (SDK가 추가하는 canvas들도 전부 흡수)
========================= */
.videoCol canvas {
  position: absolute;
  left: 0;
  top: 22px;                   /* subLabel 높이 */
  width: 100%;
  height: calc(100% - 22px);
  pointer-events: none;
}

/* =========================
   채팅 UI
========================= */
#chatBox {
  margin-top: 14px;
  border: 1px solid #ddd;
  border-radius: 12px;
  padding: 10px;
  width: 500px;
}

.chatList {
  max-height: 220px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 4px;
}

.chatItem {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.chatItem.mine {
  align-items: flex-end;
}

.chatMeta {
  font-size: 12px;
  opacity: 0.7;
  margin-bottom: 2px;
}

.chatBubble {
  background: #f3f3f3;
  border-radius: 10px;
  padding: 8px 10px;
  max-width: 85%;
  white-space: pre-wrap;
  word-break: break-word;
}

.chatItem.mine .chatBubble {
  background: #e9f3ff;
}

.chatInputRow {
  margin-top: 10px;
  display: flex;
}

.chatInput {
  width: min(900px, 100%);
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 10px;
}

</style>