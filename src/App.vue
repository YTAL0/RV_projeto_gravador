<template>
  <div class="container">
    <div v-if="isRecording" class="timer">
      00:{{ remainingTime.toString().padStart(2, '0') }}
    </div>

    <input
      v-model="audioName"
      class="name-input"
      maxlength="30"
      placeholder="Digite seu nome"
      :disabled="isUploading"
    />

    <div v-if="isUploading" style="color: white; font-family: monospace;">
      Enviando para seu amor......ou qualquer aleatório que queira ouvir.
    </div>

    <button
      v-if="!isRecording && !isUploading"
      class="heart-btn record-btn"
      @click="startRecording"
    >
      <div class="heart-wrapper">
        <svg class="heart-svg" viewBox="0 0 200 185" xmlns="http://www.w3.org/2000/svg">
          <path
            class="heart-glow"
            d="M100 170 C100 170 10 110 10 55 C10 25 35 5 65 5 C80 5 92 12 100 22 C108 12 120 5 135 5 C165 5 190 25 190 55 C190 110 100 170 100 170 Z"
            transform="scale(1.18) translate(-16, -12)"
          />
          <path
            class="heart-path"
            d="M100 170 C100 170 10 110 10 55 C10 25 35 5 65 5 C80 5 92 12 100 22 C108 12 120 5 135 5 C165 5 190 25 190 55 C190 110 100 170 100 170 Z"
          />
        </svg>
        <Mic class="heart-icon" :size="52" />
      </div>
    </button>

    <button
      v-else-if="isRecording"
      class="heart-btn stop-btn"
      @click="stopRecording"
    >
      <svg class="heart-svg" viewBox="0 0 200 185" xmlns="http://www.w3.org/2000/svg">
        <path
          class="heart-path-stop"
          d="M100 170 C100 170 10 110 10 55 C10 25 35 5 65 5 C80 5 92 12 100 22 C108 12 120 5 135 5 C165 5 190 25 190 55 C190 110 100 170 100 170 Z"
        />
      </svg>
      <Square class="heart-icon heart-icon-stop" :size="40" />
    </button>

    <audio
      v-if="audioUrl && !isUploading"
      :src="audioUrl"
      controls
      class="audio-player"
    />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { Mic, Square } from 'lucide-vue-next'
import { createClient } from '@supabase/supabase-js'
const supabaseUrl = 'https://ppsdcoifaifrfgzovwwu.supabase.co'
const supabaseKey = 'sb_publishable_I1kgINGoMJ6h5UYt-q2Kyw_j7-ZP-Wv'

const supabase = createClient(supabaseUrl, supabaseKey)
const audioName = ref('')
const isRecording = ref(false)
const isUploading = ref(false)
const audioUrl = ref(null)

const maxDuration = 15
const remainingTime = ref(maxDuration)

let mediaRecorder
let chunks = []
let countdownInterval
let currentStream

function startCountdown() {
  remainingTime.value = maxDuration
  countdownInterval = setInterval(() => {
    remainingTime.value--
    if (remainingTime.value <= 0) {
      stopRecording()
    }
  }, 1000)
}

const startRecording = async () => {
  if (!audioName.value.trim()) {
    alert('Por favor, digite seu nome antes de gravar.')
    return
  }
  
  audioUrl.value = null 
  
  currentStream = await navigator.mediaDevices.getUserMedia({ audio: true })
  mediaRecorder = new MediaRecorder(currentStream)
  chunks = []

  mediaRecorder.ondataavailable = (event) => {
    chunks.push(event.data)
  }

  mediaRecorder.onstop = async () => {
    const blob = new Blob(chunks, { type: 'audio/webm' })
    audioUrl.value = URL.createObjectURL(blob) 
    currentStream?.getTracks().forEach(track => track.stop())
    
    await uploadAudio(blob)
  }

  mediaRecorder.start()
  isRecording.value = true
  startCountdown()
}

const stopRecording = () => {
  if (!mediaRecorder || mediaRecorder.state === 'inactive') return
  clearInterval(countdownInterval)
  mediaRecorder.stop()
  isRecording.value = false
}

const uploadAudio = async (audioBlob) => {
  isUploading.value = true
  
  try {
    const fileName = `audio_${Date.now()}.webm`
    const { data: uploadData, error: uploadError } = await supabase.storage
      .from('audio')
      .upload(fileName, audioBlob, { contentType: 'audio/webm' })

    if (uploadError) throw uploadError
    const { data: publicUrlData } = supabase.storage
      .from('audio')
      .getPublicUrl(fileName)

    const finalAudioUrl = publicUrlData.publicUrl

    const { error: dbError } = await supabase
      .from('mensagens')
      .insert([
        { nome: audioName.value, audio_url: finalAudioUrl }
      ])

    if (dbError) throw dbError

    alert('Mensagem enviada com sucesso para a praça!')

  } catch (error) {
    console.error('Erro no processo de upload:', error)
    alert('Erro ao enviar o áudio. Verifique as configurações do Supabase.')
  } finally {
    isUploading.value = false
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  background: #000;
  overflow: hidden;
}

#app {
  width: 100%;
  height: 100vh;
}

.container {
  width: 100%;
  height: 100vh;
  background: #000;

  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 40px;
}

.timer {
  color: white;
  font-size: 3rem;
  font-weight: bold;
  font-family: monospace;
}

.heart-btn {
  position: relative;
  width: 280px;
  height: 260px;
  background: transparent;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.3s;
  padding: 0;
}
.heart-wrapper {
  position: relative;
  width: 160px;
  height: 148px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.heart-btn:hover {
  transform: scale(1.05);
}

.heart-svg {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: visible;
}


.heart-icon {
  position: relative;
  z-index: 1;
  color: white;
  margin-top: 8px; 
}

.heart-path {
  fill: #ff2d55;
}
.heart-path-stop {
  fill: #1f1f1f;
  stroke: #ff2d55;
  stroke-width: 6;
}

.heart-icon-stop {
  color: #ff2d55;
}

.audio-player {
  width: 90%;
  max-width: 400px;
}

.name-input {
  width: 300px;
  max-width: 90%;
  padding: 14px;
  border-radius: 12px;
  border: 1px solid #333;
  background: #111;
  color: white;
  font-size: 16px;
}
.heart-glow {
  fill: #ff2d55;
  opacity: 0.3;
  animation: heartPulse 1.5s ease-in-out infinite;
}
@keyframes heartPulse {
  0%   { opacity: 0.2; transform: scale(1.10) translate(-9px, -8px); }
  50%  { opacity: 0.55; transform: scale(1.22) translate(-18px, -14px); }
  100% { opacity: 0.2; transform: scale(1.10) translate(-9px, -8px); }
}
</style>