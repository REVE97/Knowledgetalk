<template>
  <section>
    <div id="roomButton">
      <input
        id="roomIdInput"
        v-model.trim="roomId"
        type="text"
        placeholder="room id"
      />

      <button :disabled="!ready || creating" @click="createRoom">CreateRoom</button>
      <button :disabled="!ready || joining || joined" @click="joinRoom">JoinRoom</button>
      <button :disabled="!ready || !joined" @click="startP2P">p2p</button>
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

  if (!Con) {
    throw new Error("Knowledgetalk SDK 로딩 안됨");
  }
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
        case "join": {
          const joinedUserId = msg?.user?.id || msg?.user?.userId;
          ensureVideoBox(joinedUserId);
          break;
        }

        case "leave": {
          const leftUserId =
            typeof msg.user === "string"
              ? msg.user
              : msg?.user?.id || msg?.user?.userId;

          removeVideoBox(leftUserId);
          break;
        }

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
});

const createRoom = async () => {
  creating.value = true;
  try {
    const roomData = await kt.createRoom();
    if (roomData.code !== "200") return alert("createRoom failed!");
    roomId.value = roomData.roomId;
  } finally {
    creating.value = false;
  }
};

const joinRoom = async () => {
  if (!roomId.value.trim()) {
    const input = prompt("입장할 roomId를 입력하세요.");
    if (!input) return;
    roomId.value = input.trim();
  }

  joining.value = true;
  try {
    const roomData = await kt.joinRoom(roomId.value.trim());
    if (roomData.code !== "200") return alert("joinRoom failed!");

    joined.value = true;

    const members = roomData.members || {};
    for (const memberId in members) {
      if (memberId === kt.getUserId()) continue;
      ensureVideoBox(memberId);
    }
    await nextTick();
  } finally {
    joining.value = false;
  }
};

const startP2P = async () => {
  const targetId = (prompt("상대방 id 입력") || "").trim();
  if (!targetId) return;

  const localStream = await navigator.mediaDevices.getUserMedia({
    video: true,
    audio: false
  });

  const me = kt.getUserId();
  await setVideoStream(me, localStream);

  await kt.publishP2P(targetId, "cam", localStream);
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
