<template>
  <v-dialog v-model="dialog" max-width="900" scrollable>
    <v-card>
      <v-card-title class="d-flex align-center pa-4">
        <v-icon icon="mdi-routes" class="mr-2" />
        {{ $t('routeCheck.title') }}
        <v-spacer />
      </v-card-title>
      <v-divider />
      <v-card-text class="pa-4">

        <!-- ── Step 1: Packet params ── -->
        <p class="text-subtitle-2 mb-3">{{ $t('routeCheck.packetParams') }}</p>
        <v-row dense>
          <v-col cols="12" sm="6" md="4">
            <v-select
              v-model="pkt.inbound"
              :items="[{ title: '—', value: '' }, ...inboundTags]"
              :label="$t('pages.inbounds')"
              hide-details density="compact" clearable
              @click:clear="pkt.inbound = ''"
            ></v-select>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-select
              v-model="pkt.auth_user"
              :items="[{ title: '—', value: '' }, ...clients]"
              :label="$t('pages.clients')"
              hide-details density="compact" clearable
              @click:clear="pkt.auth_user = ''"
            ></v-select>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-text-field
              v-model="pkt.domain"
              :label="$t('routeCheck.domain')"
              hide-details density="compact"
              placeholder="test.petrovich.com"
              spellcheck="false"
            ></v-text-field>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-text-field
              v-model="pkt.ip_cidr"
              :label="$t('routeCheck.destIp')"
              hide-details density="compact"
              placeholder="1.2.3.4"
              spellcheck="false"
            ></v-text-field>
          </v-col>
        </v-row>

        <!-- ── Step 2: rule_set checkboxes (if any rules reference rule_set) ── -->
        <template v-if="referencedRuleSets.length > 0">
          <v-divider class="my-4" />
          <p class="text-subtitle-2 mb-1">{{ $t('routeCheck.ruleSetHint') }}</p>
          <p class="text-caption text-medium-emphasis mb-3">{{ $t('routeCheck.ruleSetHintSub') }}</p>
          <v-row dense>
            <v-col v-for="tag in referencedRuleSets" :key="tag" cols="12" sm="6" md="4" lg="3">
              <v-checkbox
                v-model="ruleSetMatches[tag]"
                :label="tag"
                hide-details density="compact"
              ></v-checkbox>
            </v-col>
          </v-row>
        </template>

        <!-- ── Check button ── -->
        <div class="d-flex justify-end mt-4">
          <v-btn @click="runCheck" color="primary" variant="flat" prepend-icon="mdi-play">
            {{ $t('routeCheck.check') }}
          </v-btn>
        </div>

        <!-- ── Results ── -->
        <template v-if="results.length > 0">
          <v-divider class="my-4" />
          <p class="text-subtitle-2 mb-3">{{ $t('routeCheck.result') }}</p>

          <!-- Winner banner -->
          <v-alert
            v-if="winner"
            :type="winner.action === 'reject' ? 'error' : 'success'"
            variant="tonal"
            class="mb-4"
            :title="$t('routeCheck.matched', { index: winner.index + 1 })"
          >
            <strong>{{ $t('admin.action') }}:</strong> {{ winner.action }}
            <span v-if="winner.outbound"> → <strong>{{ winner.outbound }}</strong></span>
          </v-alert>
          <v-alert v-else type="warning" variant="tonal" class="mb-4" :title="$t('routeCheck.noMatch')">
            {{ $t('routeCheck.noMatchDesc', { final: finalOutbound || '—' }) }}
          </v-alert>

          <!-- Full trace table -->
          <v-table density="compact" style="font-size: 12px;">
            <thead>
              <tr>
                <th style="width:40px;">#</th>
                <th>{{ $t('type') }}</th>
                <th>{{ $t('admin.action') }}</th>
                <th>{{ $t('objects.outbound') }}</th>
                <th>{{ $t('routeCheck.status') }}</th>
                <th>{{ $t('routeCheck.reason') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="r in results" :key="r.index"
                :style="rowStyle(r)"
              >
                <td>{{ r.index + 1 }}</td>
                <td>{{ r.ruleType }}</td>
                <td>{{ r.action }}</td>
                <td>{{ r.outbound || '—' }}</td>
                <td>
                  <v-chip size="x-small" :color="statusColor(r.status)" variant="tonal">
                    {{ $t('routeCheck.status_' + r.status) }}
                  </v-chip>
                </td>
                <td style="font-size: 11px; color: rgba(var(--v-theme-on-surface), 0.6);">
                  {{ r.reason }}
                </td>
              </tr>
            </tbody>
          </v-table>
        </template>

      </v-card-text>
      <v-divider />
      <v-card-actions class="pa-3">
        <v-btn @click="reset" variant="text">{{ $t('routeCheck.reset') }}</v-btn>
        <v-spacer />
        <v-btn @click="dialog = false" variant="text">{{ $t('actions.close') }}</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts" setup>
import { ref, reactive, computed } from 'vue'
import { useI18n } from 'vue-i18n'

const { t } = useI18n()

// ── Props ──────────────────────────────────────────────────
const props = defineProps<{
  rules: any[]
  rulesets?: any[]
  finalOutbound: string
  inboundTags: string[]
  clients: string[]
}>()

// ── State ──────────────────────────────────────────────────
const dialog = ref(false)
const pkt = reactive({ inbound: '', auth_user: '', domain: '', ip_cidr: '' })
const ruleSetMatches = reactive<Record<string, boolean>>({})
const results = ref<any[]>([])

const referencedRuleSets = computed((): string[] => {
  const tags = new Set<string>()
  const collect = (rules: any[]) => {
    for (const r of rules) {
      if (r.rule_set) r.rule_set.forEach((t: string) => tags.add(t))
      if (r.rules) collect(r.rules)
    }
  }
  collect(props.rules)
  return [...tags]
})

const winner = computed(() => results.value.find(r => r.status === 'winner') ?? null)
const NON_TERMINATING = ['sniff', 'hijack-dns', 'resolve']
const remoteCache = reactive<Record<string, { domains: string[]; cidrs: string[] }>>({})

// ── Helpers ────────────────────────────────────────────────
function reset() {
  pkt.inbound = ''
  pkt.auth_user = ''
  pkt.domain = ''
  pkt.ip_cidr = ''
  results.value = []
  Object.keys(ruleSetMatches).forEach(k => delete ruleSetMatches[k])
}

function rowStyle(r: any) {
  if (r.status === 'winner') return 'background: rgba(var(--v-theme-success), 0.06);'
  if (r.status === 'skipped') return 'opacity: 0.4;'
  return ''
}

function statusColor(status: string) {
  return { winner: 'success', non_terminating: 'info', no_match: 'error', skipped: 'default' }[status]
}

// ── Remote rule_set loader ──
async function loadRemoteRuleSet(tag: string, url: string) {
  if (remoteCache[tag]) return remoteCache[tag]
  try {
    const res = await fetch(url)
    const text = await res.text()
    const lines = text.split(/\r?\n/).filter(Boolean)
    const domains: string[] = []
    const cidrs: string[] = []
    for (const l of lines) {
      if (l.match(/^\d/)) cidrs.push(l)
      else domains.push(l)
    }
    remoteCache[tag] = { domains, cidrs }
    return remoteCache[tag]
  } catch (e) {
    console.error(`Failed to load rule_set ${tag}`, e)
    return { domains: [], cidrs: [] }
  }
}

// ── Domain matcher ──
function matchesDomain(domain: string, rule: any) {
  if (!domain) return { matched: false, reason: 'no domain in packet' }
  if (rule.domain?.includes(domain)) return { matched: true, reason: `domain: exact "${domain}"` }
  return { matched: false, reason: `domain: "${domain}" not matched` }
}

// ── IP matcher ──
function matchesIpCidr(ip: string, cidrs?: string[]) {
  if (!cidrs?.length) return true
  return cidrs.includes(ip)
}

// ── Match a single rule ──
async function matchRule(rule: any) {
  const reasons: string[] = []

  const empty = !rule.domain?.length && !rule.ip_cidr?.length && !rule.rule_set?.length
  if (empty) {
    reasons.push('empty rule — auto matched')
    return { matched: true, reasons }
  }

  let matched = true

  // Domain
  if (pkt.domain) {
    const { matched: dm, reason } = matchesDomain(pkt.domain, rule)
    if (!dm) { matched = false; reasons.push(reason) }
    else reasons.push(reason)
  }

  // IP
  if (pkt.ip_cidr) {
    if (!matchesIpCidr(pkt.ip_cidr, rule.ip_cidr)) {
      matched = false
      reasons.push(`ip_cidr "${pkt.ip_cidr}" not in [${rule.ip_cidr?.join(', ')}]`)
    } else reasons.push(`ip_cidr ✓`)
  }

  // rule_set
  if (rule.rule_set?.length) {
    for (const tag of rule.rule_set) {
      let rs: any = ruleSetMatches[tag] ? { domains: [], cidrs: [] } : null
      const remote = props.rulesets?.find(r => r.tag === tag && r.type === 'remote')
      if (remote) rs = await loadRemoteRuleSet(tag, remote.url)
      if (rs) {
        let rsMatched = false
        if (pkt.domain && rs.domains.includes(pkt.domain)) rsMatched = true
        if (pkt.ip_cidr && rs.cidrs.includes(pkt.ip_cidr)) rsMatched = true
        if (!rsMatched) { matched = false; reasons.push(`rule_set: ${tag} no match`) }
        else reasons.push(`rule_set: ${tag} ✓`)
      }
    }
  }

  return { matched, reasons }
}

// ── Run check ──
async function runCheck() {
  results.value = []
  let terminated = false

  for (let i = 0; i < props.rules.length; i++) {
    const rule = props.rules[i]
    const action: string = rule.action ?? 'route'
    const outbound: string = rule.outbound ?? ''
    const ruleType = rule.type === 'logical' ? `logical (${rule.mode ?? 'and'})` : 'simple'

    if (terminated) {
      results.value.push({ index: i, ruleType, action, outbound, status: 'skipped', reason: 'earlier rule matched' })
      continue
    }

    const { matched, reasons } = await matchRule(rule)
    const effectiveMatch = rule.invert ? !matched : matched

    if (effectiveMatch) {
      const status = NON_TERMINATING.includes(action) ? 'non_terminating' : 'winner'
      results.value.push({ index: i, ruleType, action, outbound, status, reason: reasons.join('; ') })
      if (status === 'winner') terminated = true
    } else {
      results.value.push({ index: i, ruleType, action, outbound, status: 'no_match', reason: reasons.join('; ') })
    }
  }
}

// ── Expose ──
function open() { reset(); dialog.value = true }
defineExpose({ open })
</script>
