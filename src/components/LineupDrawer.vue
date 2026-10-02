<script setup>
import { ref, computed, watch } from 'vue'
import { useSkyStore } from '@/store/sky'
const store = useSkyStore()
// mode: 'plan' 仅排班（FAB 打开）；'start' 赛前排班确认（开赛按钮打开，主按钮为「排班开赛」）
const props = defineProps({ mode: { type: String, default: 'plan' } })
const emit = defineEmits(['close', 'start'])

// 草稿以服务端当前排班初始化；人员/艇只提交 id，加成数值一律由服务端核定
const pilotId = ref(null)
const mechId = ref(null)
const rentalId = ref(null)
watch(() => store.lineup, l => {
  if (!l) return
  pilotId.value = l.pilotId
  mechId.value = l.mechId
  rentalId.value = l.rentalId
}, { immediate: true })

const locked = computed(() => !!store.activeRace) // 比赛进行中排班冻结
const next = computed(() => store.circuits.find(c => !c.finished) || null)
const saving = ref(false)
const dirty = computed(() => {
  const l = store.lineup
  if (!l) return false
  return pilotId.value !== l.pilotId || mechId.value !== l.mechId || rentalId.value !== l.rentalId
})

// 预览与服务端同口径：带队 / 胆识抗性 / 技工调校（数值仅展示，结算以服务端为准）
const lead = p => p ? ((p.skill + p.courage) / 2 * 0.4 + p.exp * 0.15 + (p.mood - 50) * 0.08) : 20
const mechB = m => m ? (m.skill * 0.12 + (m.mood - 50) * 0.06) : 10
const grit = p => 0.5 + (p ? p.courage : 50) / 200
const selPilot = computed(() => store.lineup?.pilots.find(p => p.id === pilotId.value) || null)
const selMech = computed(() => store.lineup?.mechanics.find(m => m.id === mechId.value) || null)
const wIco = { '晴': '🌤️', '风': '🌬️', '雨': '🌧️', '雾': '🌫️', '雷暴': '⛈️' }

async function save(andStart) {
  if (saving.value || locked.value) return
  saving.value = true
  const r = await store.saveLineup({ pilotId: pilotId.value, mechId: mechId.value, rentalId: rentalId.value })
  saving.value = false
  if (!r.ok) { store.tip(r.msg || '排班保存失败'); return }
  if (andStart) emit('start', next.value)
  else { store.tip('赛事排班已更新'); emit('close') }
}
function pickShip(rId) { if (!locked.value) rentalId.value = rId }
</script>

<template>
  <div class="drawer-mask" @click.self="emit('close')">
    <aside class="drawer">
      <header class="d-h">
        <div>
          <h3>📋 赛事排班</h3>
          <div class="d-sub">
            {{ mode === 'start' ? '确认本场出场阵容后开赛' : '安排下一站分站赛的机师、技工与出赛艇' }}
          </div>
        </div>
        <button class="d-x" @click="emit('close')">✕</button>
      </header>

      <div class="d-body">
        <!-- 目标赛站简报 -->
        <div class="lu-brief">
          <template v-if="next">
            <div class="lb-name">{{ next.name }}</div>
            <div class="lb-tags">
              <span class="tag b">{{ wIco[next.weather] }} {{ next.weather }}</span>
              <span class="tag o">难度 {{ '★'.repeat(next.diff) }}</span>
              <span class="tag gray">名次奖金加成 {{ next.diff * 5 }}%</span>
            </div>
          </template>
          <div v-else class="d-sub">本赛季 6 站已全部完赛 🏆</div>
          <div v-if="locked" class="lu-lock">🔒 比赛进行中，完赛结算后方可调整排班</div>
        </div>

        <!-- 出赛艇：自有艇 / 在履租约艇二选一 -->
        <section>
          <div class="sec-h"><b>🛸 出赛飞艇</b><span class="d-sub">磨损与租约场次按所选艇结算</span></div>
          <div class="lu-ships">
            <button v-for="s in store.lineup?.ships || []" :key="s.kind + (s.rentalId ?? 0)"
              class="lu-ship" :class="{ on: rentalId === s.rentalId, own: s.kind === 'own' }"
              :disabled="locked" @click="pickShip(s.rentalId)">
              <div class="ls-top">
                <b>{{ s.kind === 'rental' ? '🛟 ' : '🛠️ ' }}{{ s.name }}</b>
                <span class="tag" :class="s.kind === 'rental' ? 'rose' : 'b'">
                  {{ s.kind === 'rental' ? `租约艇 · ${s.maxRaces - s.racesUsed}/${s.maxRaces} 场` : '自有艇' }}
                </span>
              </div>
              <div class="ls-perf">
                <span v-for="([k, l]) in [['speed','速度'],['turn','转向'],['acc','加速'],['dur','耐久']]" :key="k">
                  {{ l }}<em class="mono">{{ s[k] }}</em>
                </span>
                <span>健康<em class="mono" :class="{ low: s.parts_dur < 40 }">{{ s.parts_dur }}%</em></span>
              </div>
              <div v-if="s.kind === 'rental'" class="ls-sub">本场磨损记入租约 · 已累计 {{ s.wearTotal }} 点</div>
              <div v-else class="ls-sub">赛后部件磨损由车队承担，可在机库维护</div>
            </button>
          </div>
        </section>

        <!-- 机师 -->
        <section>
          <div class="sec-h">
            <b>🧑‍✈️ 出赛机师</b>
            <span class="d-sub">带队 +{{ lead(selPilot).toFixed(1) }} · 天气抗性 {{ Math.round(grit(selPilot) * 100) }}%</span>
          </div>
          <div class="lu-list">
            <button class="lu-crew" :class="{ on: pilotId === null }" :disabled="locked" @click="pilotId = null">
              <div class="crew-ava none">∅</div>
              <div class="crew-m"><div class="cm-name">不派机师</div><div class="cm-sub">带队按基础值 +20、天气抗性 75%</div></div>
            </button>
            <button v-for="p in store.lineup?.pilots || []" :key="p.id" class="lu-crew"
              :class="{ on: pilotId === p.id }" :disabled="locked" @click="pilotId = p.id">
              <div class="crew-ava" :style="{ background: 'linear-gradient(135deg,var(--gold2),var(--violet))' }">{{ p.name[0] }}</div>
              <div class="crew-m">
                <div class="cm-name">{{ p.name }}<span class="tag b sm-tag">技巧 {{ p.skill }}</span></div>
                <div class="cm-sub">胆识 {{ p.courage }} · 经验 {{ p.exp }} · 心情 {{ p.mood }}</div>
              </div>
              <span class="lu-pick">{{ pilotId === p.id ? '✔' : '' }}</span>
            </button>
          </div>
        </section>

        <!-- 技工 -->
        <section>
          <div class="sec-h">
            <b>🔧 随队技工</b>
            <span class="d-sub">调校加成 +{{ mechB(selMech).toFixed(1) }}</span>
          </div>
          <div class="lu-list">
            <button class="lu-crew" :class="{ on: mechId === null }" :disabled="locked" @click="mechId = null">
              <div class="crew-ava none">∅</div>
              <div class="crew-m"><div class="cm-name">不派技工</div><div class="cm-sub">调校按基础值 +10</div></div>
            </button>
            <button v-for="m in store.lineup?.mechanics || []" :key="m.id" class="lu-crew"
              :class="{ on: mechId === m.id }" :disabled="locked" @click="mechId = m.id">
              <div class="crew-ava" style="background:linear-gradient(135deg,var(--mint),var(--sky))">{{ m.name[0] }}</div>
              <div class="crew-m">
                <div class="cm-name">{{ m.name }}<span class="tag m sm-tag">技能 {{ m.skill }}</span></div>
                <div class="cm-sub">心情 {{ m.mood }}</div>
              </div>
              <span class="lu-pick">{{ mechId === m.id ? '✔' : '' }}</span>
            </button>
          </div>
        </section>
      </div>

      <!-- 底部操作条 -->
      <footer class="lu-foot">
        <button class="btn ghost" @click="emit('close')">取消</button>
        <button v-if="mode === 'plan'" class="btn primary" :disabled="locked || saving || !dirty" @click="save(false)">
          保存排班
        </button>
        <template v-else>
          <button class="btn ghost" :disabled="locked || saving || !dirty" @click="save(false)">仅保存</button>
          <button class="btn primary" :disabled="locked || saving || !next" @click="save(true)">
            {{ dirty ? '保存并开赛 🚀' : '排班开赛 🚀' }}
          </button>
        </template>
      </footer>
    </aside>
  </div>
</template>
