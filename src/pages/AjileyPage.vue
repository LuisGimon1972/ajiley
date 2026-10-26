<template>
  <q-page class="flex flex-center text-white q-pa-md ajiley-page">
    <div class="game-container column items-center q-gutter-md q-pa-xl" style="max-width: 850px; width: 100%">
      
      <!-- Cabeçalho / Título e Placar -->
      <div class="column items-center q-gutter-sm w-full q-px-md" style="width: 100%">
        <div class="text-h4 text-weight-bold text-white tracking-wide text-shadow">
          El Ajiley (Cuarenta y Una) Pro
        </div>
        
        <!-- Placar e Fichas -->
        <div class="row justify-center items-center q-gutter-md q-my-xs">
          <q-chip class="game-chip" icon="person">Suas Fichas: {{ fichasHumano }}</q-chip>
          <q-chip class="game-chip" icon="smart_toy">Fichas IA: {{ fichasIA }}</q-chip>
          <q-chip class="game-chip-gold" icon="local_fire_department">Pote: {{ pote }} fichas</q-chip>
        </div>

        <!-- Botões de Controle -->
        <div class="row justify-center items-center q-gutter-sm flex-wrap q-mt-sm">
          <q-btn color="amber-9" text-color="dark" icon="play_arrow" label="Nova Partida" @click="iniciarPartida" class="action-btn text-weight-bold" unelevated />
          <q-btn color="cyan-8" icon="info" label="Regras do Ajiley" @click="mostrarRegras = true" class="action-btn text-weight-bold" unelevated />
          <q-btn color="negative" icon="exit_to_app" label="Sair" @click="exitGame" class="action-btn text-weight-bold" unelevated />
        </div>
      </div>

      <!-- Banner de Estado e Mensagens -->
      <div class="status-banner text-amber-3 text-center q-px-md q-py-sm shadow-1">
        {{ mensagemStatus }}
      </div>

      <!-- Tapete de Jogo (Mesa) -->
      <div class="game-table-felt column items-center q-gutter-lg q-pa-lg shadow-10" style="width: 100%;">
        
        <!-- 1. Área do Oponente (IA) -->
        <div class="column items-center">
          <div class="text-subtitle2 text-blue-2 q-mb-xs tracking-wider">
            Oponente (IA) — Cartas: {{ maoAI.length }}
          </div>
          <div class="row q-gutter-md justify-center">
            <div v-for="(carta, idx) in maoAI" :key="'ai-'+idx" class="table-card-back flex flex-center">
              <span class="text-h5 text-amber-4">🂠</span>
            </div>
          </div>
        </div>

        <!-- 2. Centro da Mesa / Ações de Fase -->
        <div class="row justify-center items-center w-full q-py-md q-gutter-md">
          <div class="column items-center">
            <div class="text-caption text-amber-3 text-weight-bold">Fase Atual</div>
            <q-badge color="orange-8" class="text-subtitle1 q-pa-sm q-mt-xs">{{ faseAtualTexto }}</q-badge>
          </div>

          <!-- Botões para Apostas -->
          <div v-if="faseJogo === 'APUESTA_1' || faseJogo === 'APUESTA_2'" class="row q-gutter-sm">
            <q-btn color="positive" label="Apostar / Igualar (10)" @click="apostar(10)" unelevated />
            <q-btn color="negative" label="Passar / Retirar" @click="passarRodada" unelevated />
          </div>

          <!-- Botões para Descarte -->
          <div v-if="faseJogo === 'DESCARTE'" class="column items-center">
            <div class="text-caption text-yellow-2">Selecione até 4 cartas para trocar:</div>
            <q-btn color="amber-7" text-color="dark" label="Confirmar Descarte" @click="executarDescarte" class="q-mt-xs text-weight-bold" unelevated />
          </div>
        </div>

        <!-- 3. Mão do Jogador -->
        <div class="column items-center">
          <div class="text-subtitle2 text-blue-2 q-mb-xs tracking-wider">
            Sua Mão <span v-if="faseJogo === 'DESCARTE'">(Clique para selecionar e descartar)</span>
          </div>
          <div class="row q-gutter-md justify-center">
            <div
              v-for="(carta, index) in maoHumano"
              :key="'humano-'+index"
              class="player-card flex flex-center"
              :class="{ 
                'selected-card': cartasSelecionadasDescarte.includes(index),
                'disabled-card': faseJogo !== 'DESCARTE'
              }"
              @click="toggleSelecionarCarta(index)"
            >
              <img src="images/cartasTruco.png" class="atlas-img" :style="getSpriteStyle(carta)" />
            </div>
          </div>
        </div>

      </div>

      <!-- Diálogo de Regras -->
      <q-dialog v-model="mostrarRegras">
        <q-card class="bg-dark text-white q-pa-lg dialog-effect" style="max-width: 520px;">
          <q-card-section>
            <div class="text-h6 text-amber-4 text-weight-bold q-mb-md">Regras do Ajiley</div>
            <p class="text-grey-3 text-body2">
              • <b>Objetivo:</b> Somar a maior pontuação no mesmo naipe.<br>
              • <b>Mãos máximas:</b> <b>Ajiley (51 pts)</b> com o curinga ou <b>Cuarenta y una (41 pts)</b>.<br>
              • <b>Passo a passo:</b><br>
              1. Recebe 2 cartas $\rightarrow$ 1ª Aposta.<br>
              2. Recebe mais 3 cartas (total 5 em mãos).<br>
              3. Descarte: troca até 4 cartas para ficar com uma mão final de 4 cartas.<br>
              4. 2ª Aposta $\rightarrow$ Showdown.<br>
              • <b>O Curinga (7 de Oros):</b> Vale 10 pontos do naipe que mais lhe conveniar!
            </p>
          </q-card-section>
          <q-card-actions align="right">
            <q-btn color="amber-5" text-color="dark" label="Entendido" v-close-popup unelevated />
          </q-card-actions>
        </q-card>
      </q-dialog>

      <!-- Diálogo de Fim de Rodada -->
      <q-dialog v-model="mostrarFimRodada" persistent>
        <q-card class="bg-dark text-white q-pa-xl text-center dialog-effect">
          <q-card-section>
            <q-icon name="military_tech" color="amber-4" size="64px" class="q-mb-sm" />
            <div class="text-h5 text-amber-4 text-weight-bold">Resultado do Ajiley</div>
            <div class="text-subtitle1 q-mt-md text-grey-2" v-html="mensagemResultado"></div>
          </q-card-section>
          <q-card-actions align="center" class="q-mt-md">
            <q-btn color="amber-5" text-color="dark" icon="replay" label="Próxima Rodada" @click="iniciarPartida" v-close-popup unelevated />
          </q-card-actions>
        </q-card>
      </q-dialog>

    </div>
  </q-page>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';

const router = useRouter();

const fichasHumano = ref(100);
const fichasIA = ref(100);
const pote = ref(0);

const faseJogo = ref('INICIAL'); // 'APUESTA_1', 'REPARTO_3', 'DESCARTE', 'APUESTA_2', 'SHOWDOWN'
const mensagemStatus = ref('Clique em "Nova Partida" para começar o Ajiley.');

const maoHumano = ref<string[]>([]);
const maoAI = ref<string[]>([]);
const cartasSelecionadasDescarte = ref<number[]>([]);

const mostrarRegras = ref(false);
const mostrarFimRodada = ref(false);
const mensagemResultado = ref('');

const naipes = ['copa', 'oro', 'espada', 'basto'];
const valoresValidos = [1, 2, 3, 7, 10, 11, 12];

const faseJogoTextoMap: Record<string, string> = {
  'APUESTA_1': '1ª Rodada de Apostas (2 cartas)',
  'DESCARTE': 'Fase de Descarte (mão de 5 para 4 cartas)',
  'APUESTA_2': '2ª Rodada de Apostas (Mão final)',
  'SHOWDOWN': 'Contagem Final'
};

const faseAtualTexto = computed(() => faseJogoTextoMap[faseJogo.value] || 'Aguardando');

const getSpriteStyle = (carta: string) => {
  if (!carta) return {};
  const partes = carta.split('_');
  const num = parseInt(partes[0]);
  const naipe = partes[1].toLowerCase();

  let linha = 0;
  if (naipe === 'copa') linha = 0;
  else if (naipe === 'oro' || naipe === 'ouro') linha = 1;
  else if (naipe === 'espada') linha = 2;
  else if (naipe === 'basto') linha = 3;

  let coluna = num - 1;
  if (num >= 10) coluna = num - 3;

  return {
    transform: `translate(${-coluna * 10}%, ${-linha * 25}%)`,
  };
};

const criarBaralhoAjiley = () => {
  let baralho: string[] = [];
  for (const naipe of naipes) {
    for (const val of valoresValidos) {
      baralho.push(`${val}_${naipe}`);
    }
  }
  return baralho.sort(() => 0.5 - Math.random());
};

const iniciarPartida = () => {
  mostrarFimRodada.value = false;
  cartasSelecionadasDescarte.value = [];
  pote.value = 20;
  fichasHumano.value -= 10;
  fichasIA.value -= 10;

  const baralho = criarBaralho();

  // Passo 1: Recebe 2 cartas
  maoHumano.value = [baralho.pop()!, baralho.pop()!];
  maoAI.value = [baralho.pop()!, baralho.pop()!];

  faseJogo.value = 'APUESTA_1';
  mensagemStatus.value = 'Passo 1 & 2: Analise suas 2 cartas iniciais e faça sua aposta!';
};

const apostar = (valor: number) => {
  pote.value += valor;
  fichasHumano.value -= valor;

  if (faseJogo.value === 'APUESTA_1') {
    mensagemStatus.value = 'Passo 3: Recebendo mais 3 cartas (total de 5 em mãos)...';
    setTimeout(() => {
      const baralho = criarBaralho();
      // Passo 3: Recebe 3 cartas adicionais
      maoHumano.value.push(baralho.pop()!, baralho.pop()!, baralho.pop()!);
      maoAI.value.push(baralho.pop()!, baralho.pop()!, baralho.pop()!);

      faseJogo.value = 'DESCARTE';
      mensagemStatus.value = 'Passo 4: Selecione as cartas que deseja descartar e clique em Confirmar.';
    }, 1200);
  } else if (faseJogo.value === 'APUESTA_2') {
    faseJogo.value = 'SHOWDOWN';
    calcularResultadoFinal();
  }
};

const passarRodada = () => {
  mensagemStatus.value = 'Você passou. A IA recolhe o pote.';
  fichasIA.value += pote.value;
  pote.value = 0;
  mensagemResultado.value = 'A IA venceu por desistência.';
  mostrarFimRodada.value = true;
};

const toggleSelecionarCarta = (index: number) => {
  if (faseJogo.value !== 'DESCARTE') return;
  const pos = cartasSelecionadasDescarte.value.indexOf(index);
  if (pos > -1) {
    cartasSelecionadasDescarte.value.splice(pos, 1);
  } else {
    if (cartasSelecionadasDescarte.value.length < 4) {
      cartasSelecionadasDescarte.value.push(index);
    }
  }
};

const executarDescarte = () => {
  const baralho = criarBaralho();
  // Substituir cartas descartadas para manter 4 cartas finais
  for (const idx of cartasSelecionadasDescarte.value) {
    maoHumano.value[idx] = baralho.pop()!;
  }
  // Se descartou menos de 4, mantemos o tamanho final em 4 cortando o excedente ou ajustando
  if (maoHumano.value.length > 4) {
    maoHumano.value = maoHumano.value.slice(0, 4);
  }
  cartasSelecionadasDescarte.value = [];

  // IA descarta 2 cartas aleatórias
  maoAI.value.pop();
  maoAI.value.pop();

  faseJogo.value = 'APUESTA_2';
  mensagemStatus.value = 'Passo 5: Segunda rodada de apostas com a mão final formada!';
};

const calcularPontosCarta = (carta: string) => {
  const val = parseInt(carta.split('_')[0]);
  if (val === 1) return 11; // As
  if (val === 7) return 7;  // Siete
  return 10; // Rey, Caballero, Sota, 3, 2
};

const avaliarMaoAjiley = (mao: string[]) => {
  const somasPorNaipe: Record<string, number> = { copa: 0, oro: 0, espada: 0, basto: 0 };
  let temComodim = false;

  for (const carta of mao) {
    if (carta === '7_oro' || carta === '7_oros') {
      temComodim = true;
      continue;
    }
    const [valStr, naipe] = carta.split('_');
    const pts = calcularPontosCarta(carta);
    if (somasPorNaipe[naipe] !== undefined) {
      somasPorNaipe[naipe] += pts;
    }
  }

  // Encontrar o melhor naipe base sem o curinga
  let melhorNaipe = 'copa';
  let maiorSoma = 0;
  for (const n in somasPorNaipe) {
    if (somasPorNaipe[n] > maiorSoma) {
      maiorSoma = somasPorNaipe[n];
      melhorNaipe = n;
    }
  }

  // Atribuir o 7 de oros (comodim) ao melhor naipe dando +10 pontos
  if (temComodim) {
    maiorSoma += 10;
  }

  return { pontos: maiorSoma, naipe: melhorNaipe, temAjiley: maiorSoma >= 51 };
};

const calcularResultadoFinal = () => {
  const resHumano = avaliarMaoAjiley(maoHumano.value);
  const resAI = avaliarMaoAjiley(maoAI.value);

  let resultado = `Sua pontuação: <b>${resHumano.pontos} pts</b> (${resHumano.naipe})<br>Pontuação da IA: <b>${resAI.pontos} pts</b> (${resAI.naipe})<br><br>`;

  if (resHumano.pontos >= resAI.pontos) {
    fichasHumano.value += pote.value;
    resultado += `🎉 <b>Parabéns! Você venceu o pote com ${resHumano.pontos} pontos!</b>`;
  } else {
    fichasIA.value += pote.value;
    resultado += `🤖 <b>A IA levou o pote com ${resAI.pontos} pontos.</b>`;
  }

  pote.value = 0;
  mensagemResultado.value = resultado;
  mostrarFimRodada.value = true;
};

const exitGame = () => {
  router.push('/');
};

onMounted(() => {
  iniciarPartida();
});
</script>

<style scoped>
.ajiley-page {
  background: radial-gradient(circle at center, #0e2a6b 0%, #061336 100%);
  min-height: 100vh;
  width: 100%;
}

.game-container {
  background: rgba(15, 35, 95, 0.85);
  backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 24px;
  box-shadow: 0 20px 45px rgba(0, 0, 0, 0.6);
}

.status-banner {
  width: 100%;
  background: rgba(0, 0, 0, 0.35);
  border-left: 4px solid #ffb300;
  border-radius: 0 10px 10px 0;
  font-weight: 500;
  font-size: 14px;
}

.game-table-felt {
  width: 100%;
  background: radial-gradient(circle, #105c47 0%, #063024 100%);
  border: 4px solid rgba(212, 175, 55, 0.35);
  border-radius: 20px;
  box-shadow: inset 0 6px 20px rgba(0, 0, 0, 0.8), 0 8px 24px rgba(0, 0, 0, 0.4);
}

.table-card-back {
  width: 83px;
  height: 105px;
  background: linear-gradient(135deg, #1565c0 0%, #0d47a1 100%);
  border-radius: 10px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.4);
}

.player-card {
  position: relative;
  width: 95px;
  height: 120px;
  background-color: #ffffff;
  border-radius: 12px;
  cursor: pointer;
  overflow: hidden;
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.5);
  border: 3px solid transparent;
  transition: transform 0.25s ease, border-color 0.25s ease;
}

.player-card:hover:not(.disabled-card) {
  transform: translateY(-10px) scale(1.03);
}

.selected-card {
  border-color: #ffc107 !important;
  transform: translateY(-12px);
  box-shadow: 0 0 15px rgba(255, 193, 7, 0.8);
}

.disabled-card {
  cursor: default;
}

.atlas-img {
  position: absolute;
  top: 0;
  left: 0;
  width: 1000%;
  height: 400%;
  max-width: none;
  pointer-events: none;
}

.game-chip {
  background: rgba(0, 0, 0, 0.4) !important;
  color: #fff !important;
  border: 1px solid rgba(255, 255, 255, 0.15);
  font-weight: 500;
}

.game-chip-gold {
  background: linear-gradient(135deg, #ffa000 0%, #ff6f00 100%) !important;
  color: #fff !important;
  font-weight: bold;
}

.action-btn {
  width: 140px;
  font-size: 11px;
  border-radius: 8px;
}

.dialog-effect {
  background: rgba(13, 27, 75, 0.95) !important;
  backdrop-filter: blur(16px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.15);
}
</style>