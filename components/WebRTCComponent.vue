<template>
  <div>
    <h1>WebRTC with WebTransport</h1>
    <button @click="initializeConnection">Initialize Connection</button>
    <p v-if="status">Status: {{ status }}</p>

    <div v-if="webTransportReady">
      <h2>Send a Datagram</h2>
      <input v-model="datagramMessage" placeholder="Enter message" />
      <button @click="sendDatagram">Send Datagram</button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const status = ref('Disconnected');
const datagramMessage = ref('');
const webTransportReady = ref(false);

let webTransport = null;
let rtcPeerConnection = null;
let stream = null;
let streamWritable = null;
let streamReadable = null;
let datagramWritable = null;
let datagramReadable = null;
async function setRTC() {
    // Create WebRTC connection
    rtcPeerConnection = new RTCPeerConnection();
    const dataChannel = rtcPeerConnection.createDataChannel('data');

    const offer = await rtcPeerConnection.createOffer();
    await rtcPeerConnection.setLocalDescription(offer);
    return offer;
}

async function readData(readable) {
  const reader = readable.getReader();
  while (true) {
    const { value, done } = await reader.read();
    if (done) {
      break;
    }
    // value is a Uint8Array.

    console.log(new TextDecoder().decode(value));
  }
}

async function writeData(writable, data) {
  const writer = writable.getWriter();
  writer.write(data);
  status.value = `Wrote ${data} to the server...`
  writer.releaseLock()
}

async function sendOffer(offer) { 
  await writeData(streamWritable, new TextEncoder().encode(offer));
  status.value = 'Offer sent via WebTransport datagrams. Waiting for answer...';
}
async function initializeConnection() {
  try {
    status.value = 'Connecting to WebTransport...';

    // Initialize WebTransport
    webTransport = new WebTransport('https://localhost:4433/');
    await webTransport.ready;

    status.value = 'WebTransport connected. Creating WebRTC offer...';

    // Enable datagram section
    webTransportReady.value = true;

    stream = await webTransport.createBidirectionalStream()
    streamWritable = stream.writable;
    streamReadable = stream.readable;
  
    datagramWritable = webTransport.datagrams.writable;
    datagramReadable = webTransport.datagrams.readable;
    await readData(streamReadable);
    await readData(datagramReadable);
  } catch (error) {
    console.error(error);
    status.value = 'Error: ' + error.message;
  }
}

async function sendDatagram() {
  if (!webTransport || !webTransportReady.value) {
    alert('WebTransport connection is not ready.');
    return;
  }

  try {
    const sdp = await setRTC();
   
    await writeData(streamWritable, new TextEncoder().encode(datagramMessage.value));
    
    datagramMessage.value = ''; // Clear the input
  } catch (error) {
    console.error(error);
    alert('Failed to send datagram: ' + error.message);
  }
}
</script>

<style scoped>
button {
  padding: 10px 20px;
  margin-top: 20px;
  margin-right: 10px;
}

input {
  padding: 10px;
  margin-top: 20px;
  margin-right: 10px;
  width: 300px;
}

h2 {
  margin-top: 40px;
}
</style>
