<template>
  <section>
    <div id="roomButton">
      <input
        id="roomIdInput"
        v-model.trim="roomId"
        type="text"
        placeholder="room id"
      />

      <button :disabled="!ready || creating || joined" @click="createRoom">
        CreateRoom
      </button>
      <button :disabled="!ready || joining || joined" @click="joinRoom()">
        JoinRoom
      </button>

      <!-- 테스트용 수동 P2P 연결 버튼 -->
      <button :disabled="!ready || !joined" @click="manualP2P">
        p2p(수동)
      </button>
    </div>

    <div id="videoBox">
      <div v-for="id in videoIds" :key="id" class="multiVideo" :id="id">
        <p>{{ id }}</p>
        <video :id="`multiVideo-${id}`" autoplay playsinline></video>
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

const videoIds = ref([]);
const logs = ref([]);

let kt = null;
let presenceHandler = null;

// 내 로컬 스트림은 1번만 켜서 유지
let localStream = null;

// 이미 publish 한 대상 중복 방지
const publishedTargets = new Set();

const log = (type, obj) => {
  const text = JSON.stringify(JSON.parse(JSON.stringify(obj)));
  logs.value.push({ type, text });
};

const ensureVideoBox = (id) => {
  if (!id) return;
  if (!videoIds.value.includes(id)) videoIds.value.push(id);
};

const removeVideoBox = (id) => {
  videoIds.value = videoIds.value.filter((x) => x !== id);
};

const setVideoStream = async (userId, stream) => {
  ensureVideoBox(userId);
  await nextTick();

  const el = document.getElementById(`multiVideo-${userId}`);
  if (!el) return;
  el.srcObject = stream;
};

const startLocalCameraIfNeeded = async () => {
  if (localStream) return localStream;

  localStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: false
  });

  const me = kt.getUserId();
  await setVideoStream(me, localStream); // 내 화면 즉시 표시
  return localStream;
};

const publishTo = async (targetId) => {
  if (!targetId) return;
  const me = kt.getUserId();
  if (!joined.value) return;
  if (targetId === me) return;

  // 중복 publish 방지
  if (publishedTargets.has(targetId)) return;

  await startLocalCameraIfNeeded();
  await kt.publishP2P(targetId, "cam", localStream);
  publishedTargets.add(targetId);
};

const publishToAllInRoom = async (roomData) => {
  const me = kt.getUserId();
  const members = roomData?.members || {};

  // members가 객체 형태라고 가정하고 기존 코드와 동일하게 순회
  for (const memberId in members) {
    if (memberId === me) continue;
    await publishTo(memberId);
  }
};

onMounted(async () => {
  try {
    kt = createKT();

    const result = await kt.init(cpCode, authKey);
    if (result.code !== "200") {
      alert("init failed!");
      return;
    }
    ready.value = true;

    presenceHandler = async (event) => {
      const msg = event.detail;
      log("receive", msg);

      switch (msg.type) {
        // 누군가 Join하면, 버튼 누를 필요 없이 즉시 상대에게 publish
        case "join": {
          const joinedUserId = msg?.user?.id || msg?.user?.userId || msg?.user;
          if (joinedUserId) {
            ensureVideoBox(joinedUserId);     // 상대 박스 생성(아이디 표시)
            await publishTo(joinedUserId);    
          }
          break;
        }

        case "leave": {
          const leftUserId =
            typeof msg.user === "string"
              ? msg.user
              : msg?.user?.id || msg?.user?.userId;

          removeVideoBox(leftUserId);
          publishedTargets.delete(leftUserId);
          break;
        }

        // 상대 스트림이 구독(수신)되면 화면에 붙임
        case "subscribed": {
          const userId = msg.user;
          const stream = kt.getStream(userId);
          if (stream) await setVideoStream(userId, stream);
          break;
        }
      }
    };

    kt.addEventListener("presence", presenceHandler);
  } catch (e) {
    console.error(e);
    alert(String(e?.message || e));
  }
});

onBeforeUnmount(() => {
  try {
    if (kt && presenceHandler) kt.removeEventListener("presence", presenceHandler);
  } catch (_) {}

  // 카메라 자원 정리
  try {
    if (localStream) {
      localStream.getTracks().forEach((t) => t.stop());
      localStream = null;
    }
  } catch (_) {}
});

const createRoom = async () => {
  creating.value = true;
  try {
    const roomData = await kt.createRoom();
    if (roomData.code !== "200") return alert("createRoom failed!");
    roomId.value = roomData.roomId;

    // 방 생성 후 자동으로 Join까지 처리(요구사항: CreateRoom만 눌러도 내 카메라/ID 나오게)
    await joinRoom(roomId.value.trim());
  } finally {
    creating.value = false;
  }
};

const joinRoom = async (overrideRoomId) => {
  if (!overrideRoomId && !roomId.value.trim()) {
    const input = prompt("입장할 roomId를 입력하세요.");
    if (!input) return;
    roomId.value = input.trim();
  }

  const rid = (overrideRoomId || roomId.value).trim();
  if (!rid) return;

  joining.value = true;
  try {
    const roomData = await kt.joinRoom(rid);
    if (roomData.code !== "200") return alert("joinRoom failed!");

    joined.value = true;

    // Join 성공 즉시 내 카메라/ID 표시
    await startLocalCameraIfNeeded();

    // 이미 방에 있는 사람들에게 자동 publish
    await publishToAllInRoom(roomData);

    await nextTick();
  } finally {
    joining.value = false;
  }
};

// 필요하면 수동으로 특정 대상에게 publish 가능
const manualP2P = async () => {
  const targetId = (prompt("상대방 id 입력") || "").trim();
  if (!targetId) return;
  await publishTo(targetId);
};
</script>

<style>
#roomIdInput {
  padding: 10px;
  width: 200px;
  margin: 5px;
  border: 2px darkslategray solid;
  border-radius: 10px;
}

#videoBox {
  display: inline-grid;
  grid-template-columns: repeat(auto-fill, minmax(100%, auto));
}
#videoBox video {
  width: 100%;
}

#roomButton {
  display: flex;
  align-items: center;
  gap: 6px;
}
</style>