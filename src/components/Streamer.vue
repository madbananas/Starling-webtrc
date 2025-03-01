<template>
  <div class="main">
    <div v-for="camera in cameras" :key="camera.id">
      <video :id="'video-' + camera.id" class="video-box" autoplay playsinline muted />
      <audio :id="'audio-' + camera.id" muted />
      <div class="unmute-button" @click="unmuteAudio(camera)" v-show="camera.muted">Unmute</div>
      <div class="talk-button" @click="talkbackEnable(camera, !camera.talkback)">{{ camera.talkback ? 'End Talk' : 'Talk' }}</div>
      <div class="camera-name">{{ camera.name }}</div>
      <div class="status-text">{{ camera.statusText }}</div>
    </div>
  </div>
</template>

<script>
  const HUB_IP = '10.0.0.169'; // Replace with your hub's IP
  const HUB_API_KEY = 'IoCwutVjRkly'; // Replace with your integration's SDC API key, created with the Starling app

  const axios = require('axios');

  export default {
    data() {
      return {
        cameras: [],
        updateTimer: null
      }
    },
    async mounted() {
      await this.getCameras();
      this.startVideoStreams();
    },
    methods: {
      async getCameras() {
        let devices = await this.callSdcApi('devices');
        if (devices.status != 'OK') {
          console.log('Couldn\'t retrieve Google Home devices.');
          return;
        }

        console.log('Got SDC Nest devices:', devices.devices);
        let cameras = devices.devices.filter(device => device.category == 'cam' && device.supportsWebRtcStreaming);
        if (!cameras.length) {
          console.log('No cameras found.');
          return;
        }

        this.cameras = cameras.map(camera => ({
            id: camera.id,
            name: camera.name,
            statusText: 'Connecting ...',
            playing: false,
            muted: true,
            talkback: false
        }));
      },
      startVideoStreams() {
        this.cameras.forEach(async camera => {
          let remoteVideo = document.getElementById('video-' + camera.id);
          let remoteAudio = document.getElementById('audio-' + camera.id);

          camera.peerConnection = new RTCPeerConnection();
          camera.peerConnection.ontrack = event => {
            console.log('Got WebRTC stream:', event);
            if (event.track.kind == 'video' && remoteVideo.srcObject != event.streams[0]) {
              remoteVideo.srcObject = event.streams[0];
            } else if (event.track.kind == 'audio' && remoteAudio.srcObject != event.streams[0]) {
              remoteAudio.srcObject = event.streams[0];
            }
            if (!camera.playing) {
              camera.playing = true;
              camera.statusText = 'LIVE';
              setInterval(() => this.extendStream(camera.id, camera.streamId), 2 * 60 * 1000);
            }
          };

          try {
            camera.peerConnection.addTransceiver('audio', { direction: 'sendrecv' });
            camera.peerConnection.addTransceiver('video', { direction: 'sendrecv' });
            camera.peerConnection.createDataChannel('sendDataChannel', null);

            let microphoneStream = await navigator.mediaDevices.getUserMedia({ audio: true });
            camera.peerConnection.addTrack(microphoneStream.getTracks()[0], microphoneStream);

            const offer = await camera.peerConnection.createOffer();
            await camera.peerConnection.setLocalDescription(offer);

            const result = await this.callSdcApi('devices/' + camera.id + '/stream', { offer: offer.sdp, requestLocal: false });
            await camera.peerConnection.setRemoteDescription({ type: 'answer', sdp: result.answer });

            camera.streamId = result.streamId;
          } catch(error) {
            console.log(error);
            camera.statusText = 'Error';
          }
        });
      },
      unmuteAudio(camera) {
        camera.muted = false;
        document.getElementById('audio-' + camera.id).muted = false;
        document.getElementById('audio-' + camera.id).play();
      },
      async extendStream(cameraId, streamId) {
        await this.callSdcApi('devices/' + cameraId + '/stream/' + streamId + '/extend', { });
      },
      async talkbackEnable(camera, active) {
        camera.talkback = active;
        await this.callSdcApi('devices/' + camera.id + '/stream/' + camera.streamId + '/talkback', { active });
      },
      async callSdcApi(endpoint, body, query) {
        let url = 'http://' + HUB_IP + ':3080/api/connect/v2/' + endpoint + '?key=' + HUB_API_KEY + (query ? '&' + query : '');
        if (body) {
          return (await axios.post(url, body)).data;
        } else {
          return (await axios.get(url)).data;
        }
      }
    }
  }
</script>

<style scoped>
  .main {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(400px, 1fr));
    grid-gap: 1rem;
  }

  .main > div {
    position: relative;
    background: #f2f2f7;
    padding: 1rem;
    place-items: center;
    border-radius: 16px;
    max-width: calc(100vw - 3rem);
  }

  .main > div img {
    width: 100%;
    grid-area: 1/1/2/2;
  }

  .camera-name, .status-text {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    text-align: center;
  }

  .camera-name {
    margin-top: 8px;
    font-weight: 500;
    max-width: 100%;
  }

  .unmute-button {
    position: absolute;
    top: 24px;
    left: 24px;
    width: 80px;
    height: 32px;
    background-color: lightblue;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid darkgray;
    border-radius: 12px;
    cursor: pointer;
  }

  .talk-button {
    position: absolute;
    top: 24px;
    right: 24px;
    width: 80px;
    height: 32px;
    background-color: lightblue;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid darkgray;
    border-radius: 12px;
    cursor: pointer;
  }

  .video-box {
    width: 100%;
    display: flex;
  }
</style>
