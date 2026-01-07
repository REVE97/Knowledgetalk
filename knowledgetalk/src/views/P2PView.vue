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

      <button :disabled="!ready || !joined || sharing" @click="startScreenShare">
        화면 공유
      </button>

      <button :disabled="!ready || !joined || !sharing" @click="stopScreenShare">
        공유 종료
      </button>
    </div>

    <div id="videoBox">
      <div v-for="id in videoIds" :key="id" class="multiVideo">
        <p class="userLabel">{{ id }}</p>

        <div class="videoRow">
          <div class="videoCol">
            <p class="subLabel">CAM</p>
            <video :id="`multiVideo-${id}`" autoplay playsinline></video>
          </div>

          <div class="videoCol">
            <p class="subLabel">SCREEN</p>
            <video :id="`screenVideo-${id}`" autoplay playsinline></video>
            <canvas :id="`screenCanvas-${id}`" class="screenCanvas"></canvas>
          </div>
        </div>
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

const videoIds = ref([]);
const logs = ref([]);

let kt = null;
let presenceHandler = null;

let localCamStream = null;
let screenStream = null;

const publishedTargets = new Set();
const screenTargets = new Set();

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

const setCamStream = async (userId, stream) => {
  ensureVideoBox(userId);
  await nextTick();
  const el = document.getElementById(`multiVideo-${userId}`);
  if (el) el.srcObject = stream;
};

const setScreenStream = async (userId, stream) => {
  ensureVideoBox(userId);
  await nextTick();
  const el = document.getElementById(`screenVideo-${userId}`);
  if (!el) return;
  el.srcObject = null;
  await nextTick();
  el.srcObject = stream;
};

const clearScreenVideo = async (userId) => {
  await nextTick();
  const el = document.getElementById(`screenVideo-${userId}`);
  if (el) el.srcObject = null;
};

const getScreenCanvas = (userId) =>
  document.getElementById(`screenCanvas-${userId}`) || undefined;

const startLocalCamIfNeeded = async () => {
  if (localCamStream) return localCamStream;

  localCamStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: false
  });

  const me = kt.getUserId();
  await setCamStream(me, localCamStream);
  return localCamStream;
};

const publishCamTo = async (targetId) => {
  const me = kt.getUserId();
  if (!joined.value || !targetId || targetId === me) return;
  if (publishedTargets.has(targetId)) return;

  await startLocalCamIfNeeded();
  await kt.publishP2P(targetId, "cam", localCamStream);
  publishedTargets.add(targetId);
};

const publishCamToAllInRoom = async (roomData) => {
  const me = kt.getUserId();
  const members = roomData?.members || {};
  for (const id in members) {
    if (id !== me) await publishCamTo(id);
  }
};

const startScreenStreamIfNeeded = async () => {
  if (screenStream) return screenStream;

  screenStream = await navigator.mediaDevices.getDisplayMedia({ video: true });
  const [track] = screenStream.getVideoTracks();
  if (track) track.onended = () => stopScreenShare();

  const me = kt.getUserId();
  await setScreenStream(me, screenStream);
  return screenStream;
};

const screenStartTo = async (targetId) => {
  const me = kt.getUserId();
  if (!joined.value || !targetId || targetId === me) return;
  if (screenTargets.has(targetId)) return;

  const stream = await startScreenStreamIfNeeded();
  const canvas = getScreenCanvas(me);

  const res = await kt.screenStart(stream, targetId, canvas);
  if (res?.code === "200") {
    screenTargets.add(targetId);
  } else {
    alert("screenStart failed!");
  }
};

const screenStartToAllInRoom = async () => {
  const me = kt.getUserId();
  for (const id of videoIds.value) {
    if (id !== me) await screenStartTo(id);
  }
};

onMounted(async () => {
  try {
    kt = createKT();
    const result = await kt.init(cpCode, authKey);
    if (result.code !== "200") return alert("init failed!");
    ready.value = true;

    presenceHandler = async (e) => {
      const msg = e.detail;
      log("receive", msg);

      switch (msg.type) {
        case "join": {
          const uid = msg?.user?.id || msg?.user?.userId || msg?.user;
          if (!uid) break;

          ensureVideoBox(uid);
          await publishCamTo(uid);

          if (sharing.value) await screenStartTo(uid);
          break;
        }

        case "leave": {
          const uid =
            typeof msg.user === "string"
              ? msg.user
              : msg?.user?.id || msg?.user?.userId;

          if (!uid) break;

          removeVideoBox(uid);
          publishedTargets.delete(uid);
          screenTargets.delete(uid);
          await clearScreenVideo(uid);
          break;
        }

        case "subscribed": {
          if (msg.cam) {
            const stream = kt.getStream(msg.user);
            if (stream) await setCamStream(msg.user, stream);
          } else {
            const stream = kt.getStream("screen");
            if (stream) await setScreenStream(msg.user, stream);
            else await clearScreenVideo(msg.user);
          }
          break;
        }

        case "shareStop": {
          const sender = msg.sender || msg.user;
          if (sender) await clearScreenVideo(sender);
          break;
        }

        default:
          break;
      }
    };

    kt.addEventListener("presence", presenceHandler);
  } catch (err) {
    console.error(err);
    alert(String(err?.message || err));
  }
});

onBeforeUnmount(() => {
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
    if (screenStream) {
      screenStream.getTracks().forEach((t) => t.stop());
      screenStream = null;
    }
  } catch (_) {}
});

const createRoom = async () => {
  creating.value = true;
  try {
    const res = await kt.createRoom();
    if (res.code !== "200") return alert("createRoom failed!");
    roomId.value = res.roomId;

    // 생성 후 자동 Join
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
    const res = await kt.joinRoom(rid);
    if (res.code !== "200") return alert("joinRoom failed!");

    joined.value = true;

    await startLocalCamIfNeeded();
    await publishCamToAllInRoom(res);

    await nextTick();
  } finally {
    joining.value = false;
  }
};

const startScreenShare = async () => {
  try {
    sharing.value = true;
    
    await kt.shareStop().catch(() => {});
    screenTargets.clear();

    await startScreenStreamIfNeeded();
    await screenStartToAllInRoom();
  } catch (e) {
    console.error(e);
    sharing.value = false;
    alert(String(e?.message || e));
  }
};

const stopScreenShare = async () => {
  sharing.value = false;
  screenTargets.clear();

  await kt.shareStop().catch(() => {});

  if (screenStream) {
    screenStream.getTracks().forEach((t) => t.stop());
    screenStream = null;
  }

  const me = kt?.getUserId?.();
  if (me) await clearScreenVideo(me);
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

#roomButton {
  display: flex;
  align-items: center;
  gap: 6px;
}

#videoBox {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.multiVideo {
  border: 1px solid #ddd;
  border-radius: 12px;
  padding: 10px;
}

.userLabel {
  margin: 0 0 8px;
  font-weight: 700;
}

.videoRow {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.videoCol {
  position: relative;
}

.subLabel {
  margin: 0 0 6px;
  font-size: 12px;
  opacity: 0.8;
}

#videoBox video {
  width: 100%;
  border-radius: 10px;
  background: #000;
}

.screenCanvas {
  position: absolute;
  left: 0;
  top: 22px;
  width: 100%;
  height: calc(100% - 22px);
  pointer-events: none;
}
</style>