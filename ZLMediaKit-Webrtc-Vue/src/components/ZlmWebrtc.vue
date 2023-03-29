<!--  -->
<template>
  <div style="text-align: center">
    <div>
      <video ref="video" controls autoplay style="text-align: left">
        Your browser is too old which doesn't support HTML5 video.
      </video>

      <video ref="selfVideo" controls autoplay style="text-align: right">
        Your browser is too old which doesn't support HTML5 video.
      </video>
    </div>

    <div>
      <p>
        <label for="streamUrl">url:</label>
        <input type="text" style="co" ref="url" v-model="streamUrl" />
      </p>

      <p>
        <label for="simulcast">simulcast:</label>
        <input type="checkbox" ref="simulcast" />
      </p>
      <p>
        <label for="useCamera">useCamera:</label>
        <input type="checkbox" checked="checked" ref="useCamera" />
      </p>

      <p>
        <label for="audioEnable">audioEnable:</label>
        <input type="checkbox" ref="audioEnable" checked="checked" />
      </p>

      <p>
        <label for="videoEnable">videoEnable:</label>
        <input type="checkbox" ref="videoEnable" checked="checked" />
      </p>

      <p>
        <label for="method">method(play or push or echo):</label>
        <input
          type="radio"
          name="method"
          @click="radioChange('echo')"
          value="echo"
        />echo
        <input
          type="radio"
          name="method"
          @click="radioChange('push')"
          value="push"
        />push
        <input
          type="radio"
          name="method"
          @click="radioChange('play')"
          value="play"
          checked="true"
        />play
      </p>
      <p>
        <label for="resolution">resolution:</label>
        <!-- <select id="resolution" :value="resolution"></select> -->
        <select @change="changeResolution">
          <option :value="resolution.value" v-for="resolution in resolutions">
            {{ resolution.text }}
          </option>
        </select>
      </p>
      <p>
        <label for="datachannel">datachannel:</label>
        <input ref="datachannel" name="datachannel" type="checkbox" value="0" />
      </p>
      <button @click="startVideo">开始(start)</button>
      <button @click="stopVideo">停止(stop)</button>

      <p>
        <label for="msgsend">msgsend:</label>
        <input type="text" ref="msgsend" value="hello word !" />
      </p>
      <p>
        <label for="msgrecv">msgrecv:</label>
        <input type="text" ref="msgrecv" disabled />
      </p>
      <button @click="send">发送(send by datachannel)</button>
      <button @click="close">关闭(close datachannel)</button>
    </div>
  </div>
</template>

<script setup name="video">
import { ref, getCurrentInstance } from "vue";
//控件
const video = ref(null);
const selfVideo = ref(null);
const simulcast = ref(null);
const useCamera = ref(null);
const audioEnable = ref(null);
const videoEnable = ref(null);
const datachannel = ref(null);
const msgrecv = ref(null);
const msgsend = ref(null);
const url = ref(null);

const player = ref();
const recvOnly = ref(true);
let ishttps = "https:" == document.location.protocol ? true : false;
let isLocal = "file:" == document.location.protocol ? true : false;
const streamUrl = ref(
  document.location.protocol +
    "//" +
    window.location.host +
    "/index/api/webrtc?app=live&stream=test&type=play"
);
console.log(streamUrl.value);
if (!ishttps && !isLocal) {
    alert(
      "本demo需要在https的网站访问 ,如果你要推流的话(this demo must access in site of https if you want push stream)"
    );
}

if (isLocal) {
  streamUrl.value =
    "http://127.0.0.1" + "/index/api/webrtc?app=live&stream=test&type=play";
}

//点击radio事件
const radioChange = (value) => {
  let url = new URL(streamUrl.value);
  console.log(value);
  url.searchParams.set("type", value);
  streamUrl.value = url;
  if (value == "play") {
    recvOnly.value = true;
  } else if (value == "echo") {
    recvOnly.value = false;
  } else {
    recvOnly.value = false;
  }
};

//resolution选项
const resolutions = ref([]);
ZLMRTCClient.GetAllScanResolution().forEach((r, i) => {
  let text = r.label + "(" + r.width + "x" + r.height + ")";
  resolutions.value.push({ text: text, value: r });
});

//改变Resolution
const resolution = ref(resolutions.value[0].text);
const changeResolution = (e) => {
  resolution.value = e.target.options[e.target.selectedIndex].text;
};

//开始播放
const start_play = () => {
  let res = resolution.value.match(/\d+/g);
  let h = parseInt(res.pop());
  let w = parseInt(res.pop());

  player.value = new ZLMRTCClient.Endpoint({
    element: video.value, // video 标签
    debug: true, // 是否打印日志
    zlmsdpUrl: streamUrl.value, //流地址
    simulcast: simulcast.value.checked,
    useCamera: useCamera.value.checked,
    audioEnable: audioEnable.value.checked,
    videoEnable: videoEnable.value.checked,
    recvOnly: recvOnly.value,
    resolution: { w: w, h: h },
    usedatachannel: datachannel.value.checked,
  });

  player.value.on(ZLMRTCClient.Events.WEBRTC_ICE_CANDIDATE_ERROR, function (e) {
    // ICE 协商出错
    console.log("ICE 协商出错");
  });

  player.value.on(ZLMRTCClient.Events.WEBRTC_ON_REMOTE_STREAMS, function (e) {
    //获取到了远端流，可以播放
    console.log("播放成功", e.streams);
  });

  player.value.on(
    ZLMRTCClient.Events.WEBRTC_OFFER_ANWSER_EXCHANGE_FAILED,
    function (e) {
      // offer anwser 交换失败
      console.log("offer anwser 交换失败", e);
      stop();
    }
  );

  player.value.on(ZLMRTCClient.Events.WEBRTC_ON_LOCAL_STREAM, function (s) {
    // 获取到了本地流

    selfVideo.value.srcObject = s;
    selfVideo.value.muted = true;
    //console.log('offer anwser 交换失败',e)
  });

  player.value.on(ZLMRTCClient.Events.CAPTURE_STREAM_FAILED, function (s) {
    // 获取本地流失败
    console.log("获取本地流失败");
  });

  player.value.on(
    ZLMRTCClient.Events.WEBRTC_ON_CONNECTION_STATE_CHANGE,
    function (state) {
      // RTC 状态变化 ,详情参考 https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/connectionState
      console.log("当前状态==>", state);
    }
  );

  player.value.on(
    ZLMRTCClient.Events.WEBRTC_ON_DATA_CHANNEL_OPEN,
    function (event) {
      console.log("rtc datachannel 打开 :", event);
    }
  );

  player.value.on(
    ZLMRTCClient.Events.WEBRTC_ON_DATA_CHANNEL_MSG,
    function (event) {
      console.log("rtc datachannel 消息 :", event.data);
      msgrecv.value.value = event.data;
    }
  );
  player.value.on(
    ZLMRTCClient.Events.WEBRTC_ON_DATA_CHANNEL_ERR,
    function (event) {
      console.log("rtc datachannel 错误 :", event);
    }
  );
  player.value.on(
    ZLMRTCClient.Events.WEBRTC_ON_DATA_CHANNEL_CLOSE,
    function (event) {
      console.log("rtc datachannel 关闭 :", event);
    }
  );
};

const startVideo = () => {
  console.log("开始");
  stopVideo();
  let res = resolution.value.match(/\d+/g);
  let h = parseInt(res.pop());
  let w = parseInt(res.pop());
  if (useCamera.value.checked && !recvOnly.value) {
    ZLMRTCClient.isSupportResolution(w, h)
      .then((e) => {
        start_play();
      })
      .catch((e) => {
        alert("not support resolution");
      });
  } else {
    start_play();
  }
};

const stopVideo = () => {
  console.log("停止");
  if (player.value) {
    player.value.close();
    player.value = null;
    var remote = video.value;
    if (remote) {
      remote.srcObject = null;
      remote.load();
    }
    var local = selfVideo.value;
    local.srcObject = null;
    local.load();
  }
};

const send = () => {
  if (player.value) {
    //send msg refernece https://developer.mozilla.org/en-US/docs/Web/API/RTCDataChannel/send
    player.value.sendMsg(msgsend.value.value);
  }
};

const close = () => {
  if (player) {
    player.value.closeDataChannel();
  }
};
</script>

<style lang="scss" scoped></style>
