<template>
  <section>
    <div id="roomButton">
      <input
        id="roomIdInput"
        v-model.trim="roomId"
        type="text"
        placeholder="room id"
      />

      <button :disabled="!ready || creating || joined" @click="createVideoRoom">
        CreateVideoRoom
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

const subscribed = new Set();

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
  if (!el) return;
  el.srcObject = stream;
};

const setScreenStream = async (id, stream) => {
  ensureUser(id);
  await nextTick();
  const el = document.getElementById(`screenVideo-${id}`);
  if (!el) return;
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