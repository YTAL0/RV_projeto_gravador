<!-- <template>
  <div class="praca-container">
    
    <div v-if="!isPlaying" class="start-screen">
      <h2>Bem-vindo à Praça do Leão 🦁</h2>
      <p>Prepare-se para ouvir as mensagens da cidade.</p>
      <button @click="iniciarExperiencia" class="btn-play">Iniciar Experiência Sonora</button>
    </div>

    <div v-else class="playing-screen">
      <h2>Tocando os sons da praça... 🔊</h2>
      <p>Quantidade de mensagens enviadas: {{ playlist.length }}</p>

      <audio
        ref="meuAudio"
        :src="audioAtualUrl"
        @ended="tocarProximo"
        controls
        class="audio-player"
      ></audio>
    </div>
    
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { createClient } from '@supabase/supabase-js'
//n coloquei essa bomba em variavel de ambiente pq é só leitura mesmo, n tem segredo nenhum
const supabaseUrl = 'https://ppsdcoifaifrfgzovwwu.supabase.co'
const supabaseKey = 'sb_publishable_I1kgINGoMJ6h5UYt-q2Kyw_j7-ZP-Wv'
const supabase = createClient(supabaseUrl, supabaseKey)

const playlist = ref([])
const audioAtualUrl = ref('')
const isPlaying = ref(false)

const meuAudio = ref(null) 

onMounted(async () => {
  const { data, error } = await supabase
    .from('mensagens')
    .select('audio_url')
  
  if (data && data.length > 0) {
    playlist.value = data.map(item => item.audio_url).sort(() => Math.random() - 0.5)
  }
})

const iniciarExperiencia = () => {
  if (playlist.value.length === 0) {
    alert('A praça está silenciosa... Volte ao museu e grave a primeira mensagem!')
    return
  }
  
  isPlaying.value = true
  audioAtualUrl.value = playlist.value[0]
  
  setTimeout(() => {
    if (meuAudio.value) meuAudio.value.play()
  }, 100)
}

const tocarProximo = () => {
  if (playlist.value.length <= 1) {
    meuAudio.value.currentTime = 0
    meuAudio.value.play()
    return
  }

  const tocado = playlist.value.shift()
  playlist.value.push(tocado)
  
  audioAtualUrl.value = playlist.value[0]

  setTimeout(() => {
    if (meuAudio.value) {
      meuAudio.value.play()
    }
  }, 150)
}
</script>

<style scoped>
.praca-container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background: #050505; 
  color: white;
  text-align: center;
}

.start-screen, .playing-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.start-screen h2, .playing-screen h2 {
  font-family: sans-serif;
  font-size: 2rem;
  margin-bottom: 10px;
}

.start-screen p, .playing-screen p {
  color: #aaa;
  font-size: 1.1rem;
}

.btn-play {
  background: #ff2d55;
  color: white;
  border: none;
  padding: 16px 32px;
  font-size: 1.2rem;
  font-weight: bold;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.2s ease, background 0.2s ease;
}

.btn-play:hover {
  background: #ff0033;
  transform: scale(1.05);
}

.audio-player {
  margin-top: 20px;
  width: 300px;
}
</style> -->
<template>
  <a-scene
    vr-mode-ui="enabled: false"
    arjs="sourceType: webcam; videoTexture: true; debugUIEnabled: false;"
  >
    <a-camera gps-camera rotation-reader></a-camera>

    <a-box color="red" gps-entity-place="latitude: -4.970194; longitude: -39.015861" scale="2 2 2"></a-box>
  </a-scene>


  <div class="praca-container">
    
    <div v-if="!isPlaying" class="start-screen">
      <h2>Bem-vindo à Praça do Leão 🦁</h2>
      
      <p v-if="buscandoGps">Procurando satélites... 🛰️</p>
      
      <div v-else>
        <p v-if="distancia > limiteMetros">
          Você está a {{ distancia.toFixed(0) }} metros da praça.<br>
          Caminhe até o local para liberar os áudios! 🚶‍♂️
        </p>
        <p v-else>
          Você chegou à praça! 🎉
        </p>
      </div>

      <button 
        @click="iniciarExperiencia" 
        class="btn-play"
        :disabled="distancia > limiteMetros || buscandoGps"
        :class="{ 'btn-bloqueado': distancia > limiteMetros || buscandoGps }"
      >
        {{ distancia > limiteMetros ? '🔒 Bloqueado' : 'Iniciar Experiência Sonora' }}
      </button>
    </div>

    <div v-else class="playing-screen">
      <h2>Tocando os sons da praça... 🔊</h2>
      <p>Quantidade de mensagens enviadas: {{ playlist.length }}</p>

      <audio
        ref="meuAudio"
        :src="audioAtualUrl"
        @ended="tocarProximo"
        controls
        class="audio-player"
      ></audio>
    </div>
    
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = 'https://ppsdcoifaifrfgzovwwu.supabase.co'
const supabaseKey = 'sb_publishable_I1kgINGoMJ6h5UYt-q2Kyw_j7-ZP-Wv'
const supabase = createClient(supabaseUrl, supabaseKey)

const playlist = ref([])
const audioAtualUrl = ref('')
const isPlaying = ref(false)
const meuAudio = ref(null) 

// --- NOVAS VARIÁVEIS DO GPS ---
const buscandoGps = ref(true)
const distancia = ref(9999) // Começa com uma distância impossível
const limiteMetros = 30 // A margem de erro (cerca virtual)
const PRACA_LAT = -4.970194
const PRACA_LNG = -39.015861

// Matemática para calcular a distância em metros (Haversine)
const calcularDistancia = (lat1, lon1, lat2, lon2) => {
  const R = 6371e3; // Raio da Terra em metros
  const rad = Math.PI / 180;
  const dLat = (lat2 - lat1) * rad;
  const dLon = (lon2 - lon1) * rad;
  const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
            Math.cos(lat1 * rad) * Math.cos(lat2 * rad) *
            Math.sin(dLon/2) * Math.sin(dLon/2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  return R * c; // Retorna a distância em metros
}

onMounted(async () => {
  // 1. Busca os áudios
  const { data, error } = await supabase.from('mensagens').select('audio_url')
  if (data && data.length > 0) {
    playlist.value = data.map(item => item.audio_url).sort(() => Math.random() - 0.5)
  }

  // 2. Liga o Radar do GPS
  if ("geolocation" in navigator) {
    navigator.geolocation.watchPosition(
      (posicao) => {
        // Pega a localização do usuário e calcula a distância até a praça
        const userLat = posicao.coords.latitude
        const userLng = posicao.coords.longitude
        distancia.value = calcularDistancia(userLat, userLng, PRACA_LAT, PRACA_LNG)
        buscandoGps.value = false
      },
      (erro) => {
        alert('Por favor, ative a sua localização (GPS) para usar o site!')
        buscandoGps.value = false
      },
      { enableHighAccuracy: true } // Pede a maior precisão possível do celular
    )
  }
})

// O restante do seu código (iniciarExperiencia, tocarProximo) continua igualzinho...
const iniciarExperiencia = () => {
  if (playlist.value.length === 0) {
    alert('A praça está silenciosa... Volte ao museu e grave a primeira mensagem!')
    return
  }
  isPlaying.value = true
  audioAtualUrl.value = playlist.value[0]
  setTimeout(() => { if (meuAudio.value) meuAudio.value.play() }, 100)
}

const tocarProximo = () => {
  if (playlist.value.length <= 1) {
    meuAudio.value.currentTime = 0
    meuAudio.value.play()
    return
  }
  const tocado = playlist.value.shift()
  playlist.value.push(tocado)
  audioAtualUrl.value = playlist.value[0]
  setTimeout(() => { if (meuAudio.value) meuAudio.value.play() }, 150)
}
</script>

<style scoped>
/* A magia do CSS para a câmera aparecer por baixo de tudo */
.praca-container {
  position: absolute; /* Descola o Vue do fundo da tela */
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 10; /* Coloca o Vue NA FRENTE da câmera */
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  /* Mudamos o fundo para semi-transparente (rgba) para vermos a câmera! */
  background: rgba(5, 5, 5, 0.85); 
  color: white;
  text-align: center;
}

/* O restante do seu CSS continua... */
.start-screen, .playing-screen { display: flex; flex-direction: column; align-items: center; gap: 20px; }
.start-screen h2, .playing-screen h2 { font-family: sans-serif; font-size: 2rem; margin-bottom: 10px; }
.start-screen p, .playing-screen p { color: #aaa; font-size: 1.1rem; }

.btn-play {
  background: #ff2d55;
  color: white;
  border: none;
  padding: 16px 32px;
  font-size: 1.2rem;
  font-weight: bold;
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.2s ease, background 0.2s ease;
}

.btn-play:hover:not(:disabled) { background: #ff0033; transform: scale(1.05); }

/* Classe para deixar o botão cinza quando estiver bloqueado */
.btn-bloqueado {
  background: #444;
  cursor: not-allowed;
  opacity: 0.7;
}

.audio-player { margin-top: 20px; width: 300px; }
</style>