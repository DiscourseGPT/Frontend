<template>
  <div
    class="min-h-screen bg-[#2D2D2C] w-[100vw] text-white py-10 px-10 backdrop-blur-3xl"
  >
    <div class="text-left">
      <router-link to="/">
        <button>
          <i class="fa-solid fa-angle-left fa-2xl"></i>
        </button>
      </router-link>
    </div>

    <div class="">
      <div class="text-7xl mt-10">GPT</div>
      <div class="mt-5 text-2xl text-neutral-400">Call Duration - 00:11</div>
    </div>
    <div class="mt-16 flex px-0 justify-between mx-5">
      <div
        class="h-20 w-20 py-7 rounded-full backdrop-brightness-75"
        :class="[microphone ? '' : 'bg-red-600']"
      >
        <button @click="microphone_exchange()">
          <i class="fa-solid fa-microphone fa-2xl"></i>
        </button>
      </div>
      <div class="h-20 w-20 rounded-full py-7 backdrop-brightness-75">
        <i class="fa-solid fa-bars fa-2xl"></i>
      </div>
      <div class="h-20 w-20 py-7 rounded-full backdrop-brightness-75">
        <i class="fa-solid fa-volume-high fa-2xl"></i>
      </div>
    </div>
    <div class="mt-6 flex px-0 justify-between mx-5">
      <div class="h-20 w-20 py-7 rounded-full backdrop-brightness-75">
        <i class="fa-solid fa-plus fa-2xl"></i>
      </div>
      <div class="h-20 w-20 py-7 rounded-full backdrop-brightness-75">
        <i class="fa-solid fa-record-vinyl fa-2xl"></i>
      </div>
      <div class="h-20 w-20 py-7 rounded-full backdrop-brightness-75">
        <i class="fa-regular fa-comment fa-2xl"></i>
      </div>
    </div>
    <div class="h-20 w-20 rounded-full py-7 mx-auto my-28 bg-red-600">
      <i class="fa-solid fa-phone-slash fa-2xl"></i>
    </div>
    <audio controls :src="audioUrl" class="hidden" autoplay></audio>
  </div>
</template>
<script>
import axios from "axios";
/* eslint-disable */
export default {
  name: "AppIncall",
  mounted() {
    // this.call_audio();
  },
  data() {
    return {
      audioUrl: null,
      microphone: false,
      audioChunks: [],
      audioContext: null,
      mediaRecorder: null,
    };
  },
  methods: {
    call_audio() {
      axios
        .get("http://127.0.0.1:8000/api/audio/", { responseType: "arraybuffer" })
        .then((response) => {
          const audioBlob = new Blob([response.data], { type: "audio/mp3" });
          this.audioUrl = URL.createObjectURL(audioBlob);
          console.log(this.audioUrl);
          // Now 'audioUrl' contains the URL of your audio file
          // You can use it in an <audio> tag or wherever you need
        })
        .catch((error) => {
          console.error("Error fetching audio file:", error);
        });
    },
    microphone_exchange() {
      if (this.microphone == true) {
        this.stopRecording();
        this.microphone = false;
      } else {
        this.startRecording();
        this.microphone = true;
      }
    },
    async startRecording() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.audioContext = new AudioContext();
        const input = this.audioContext.createMediaStreamSource(stream);

        this.mediaRecorder = new MediaRecorder(stream);

        this.mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) {
            this.audioChunks.push(event.data);
          }
        };

        this.mediaRecorder.onstop = () => {
          this.audioContext.close();
          const audioBlob = new Blob(this.audioChunks, { type: "audio/mpeg" });
          this.audioUrl = URL.createObjectURL(audioBlob);
        };

        this.mediaRecorder.start();
        this.recording = true;
      } catch (error) {
        console.error("Error accessing microphone:", error);
      }
    },

    async stopRecording() {
      if (this.recording) {
        this.mediaRecorder.stop();
        this.recording = false;

        // Wait for the 'dataavailable' event to ensure all data is processed
        await new Promise((resolve) => {
          this.mediaRecorder.ondataavailable = (event) => {
            console.log("Data available:", event.data);

            // Add the data to the audioChunks array
            if (event.data.size > 0) {
              this.audioChunks.push(event.data);
            }

            resolve(event);
          };
        });

        // Explicitly close the MediaRecorder and the associated stream
        this.mediaRecorder.stream.getTracks().forEach((track) => track.stop());
        this.mediaRecorder.onstop = null;

        // Log the length of audioChunks to verify if data is present
        console.log("Length of audioChunks:", this.audioChunks.length);

        this.sendAudio();
      }
    },

    sendAudio() {
      if (this.audioChunks.length === 0) {
        console.warn("No audio to send.");
        return;
      }

      const audioBlob = new Blob(this.audioChunks, { type: "audio/mpeg" });

      const formData = new FormData();
      formData.append("audio_file", audioBlob, "audio.mp3");

      axios.post("http://127.0.0.1:8000/api/upload-audio/", formData, {
          headers: {
            "Content-Type": "multipart/form-data",
          },
        })
        .then((response) => {
          
          console.log("Audio sent successfully:", response.data);
          this.call_audio()
        })
        .catch((error) => {
          
          console.error("Error sending audio:", error);
        });
    },
  },
};
</script>
