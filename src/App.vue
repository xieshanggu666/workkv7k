<script setup>
import { ref } from 'vue'
import { useSkyStore } from '@/store/sky'
import MainScene from '@/components/MainScene.vue'
import RaceAnim from '@/components/RaceAnim.vue'
import HangarDrawer from '@/components/HangarDrawer.vue'
import TrophyDrawer from '@/components/TrophyDrawer.vue'
import LineupDrawer from '@/components/LineupDrawer.vue'

const store = useSkyStore()
store.init()

const drawer = ref('')            // '' | 'hangar' | 'trophy' | 'lineup'
// 排班抽屉场景：plan=仅调整排班（FAB）；start=赛前排班确认（点开赛）
const lineupMode = ref('plan')
// 当前观看的比赛：{ race, mode: 'live' | 'replay' }，三者（动画/实时排名/结算卡）共用同一份记录
const viewing = ref(null)
// 排班确认开赛中（等待服务端生成比赛记录）
const starting = ref(false)

// live：开赛或中断续看；replay：历史回放（绝不再次结算）
function openRace(race, mode = 'live') { viewing.value = { race, mode } }
async function goBack() {
  viewing.value = null
  await store.refresh()
}
function openHangar() { drawer.value = drawer.value === 'hangar' ? '' : 'hangar' }
function openTrophy() { drawer.value = drawer.value === 'trophy' ? '' : 'trophy' }
// FAB：仅排班；主场景开赛按钮：赛前排班确认
function openLineupPlan() { lineupMode.value = 'plan'; drawer.value = drawer.value === 'lineup' ? '' : 'lineup' }
function openLineupStart() { lineupMode.value = 'start'; drawer.value = 'lineup' }
// 排班抽屉确认开赛：服务端按已保存排班生成完整比赛记录
async function lineupStart(c) {
  if (starting.value || !c) return
  starting.value = true
  const r = await store.startRace(c.id)
  starting.value = false
  if (r.ok) {
    drawer.value = ''
    if (r.resumed) store.tip('继续观看未结束的比赛')
    openRace(r.race, 'live')
  } else store.tip(r.msg || '当前还不能参加该站')
}
</script>

<template>
  <div class="game">
    <div v-if="store.toast" class="toast">✨ {{ store.toast }}</div>

    <!-- 顶栏状态条 -->
    <header class="topbar">
      <div class="brand">
        <div class="logo">🛸</div>
        <div class="t">天空之城<small>AIRWAVE RACING · S{{ store.team.season }}</small></div>
      </div>
      <div class="stat-chips">
        <span class="chipx"><b class="ic">🪙</b> ¥{{ store.team.money?.toLocaleString() }}</span>
        <span class="chipx"><b class="ic">✨</b> 声望 {{ store.team.rep }}</span>
        <span class="chipx gold"><b class="ic">🏅</b> 积分 {{ store.team.season_pts }}</span>
        <span class="chipx"><b class="ic">🗼</b> {{ store.state ? store.state.seasonDone + ' / ' + store.state.seasonTotal + ' 站' : '' }}</span>
      </div>
    </header>

    <!-- 主游戏场景：云海浮岛航线图 -->
    <MainScene class="scene" @view="openRace" @lineup="openLineupStart" />

    <!-- 竞速镜头：live（开赛/续看）或 replay（历史回放） -->
    <RaceAnim v-if="viewing" :race="viewing.race" :mode="viewing.mode" @back="goBack" />

    <!-- 右下操作钮 -->
    <div class="fab-col">
      <button class="fab" :class="{ on: drawer === 'trophy' }" @click="openTrophy">🏆<span>赛季之巅</span></button>
      <button class="fab" :class="{ on: drawer === 'lineup' }" @click="openLineupPlan">📋<span>赛事排班</span></button>
      <button class="fab" :class="{ on: drawer === 'hangar' }" @click="openHangar">✈️<span>机库</span></button>
    </div>

    <!-- 抽屉 -->
    <transition name="slide">
      <div v-if="drawer === 'hangar'"><HangarDrawer @close="drawer = ''" /></div>
    </transition>
    <transition name="slide">
      <div v-if="drawer === 'trophy'"><TrophyDrawer @close="drawer = ''" @replay="r => openRace(r, 'replay')" /></div>
    </transition>
    <transition name="slide">
      <div v-if="drawer === 'lineup'">
        <LineupDrawer :mode="lineupMode" @close="drawer = ''" @start="lineupStart" />
      </div>
    </transition>
  </div>
</template>
