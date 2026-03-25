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
            <v-select
              v-model="pkt.network"
              :items="['tcp', 'udp', 'icmp']"
              :label="$t('network')"
              hide-details density="compact" clearable
              @click:clear="pkt.network = ''"
            ></v-select>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-select
              v-model="pkt.protocol"
              :items="protocols"
              :label="$t('protocol')"
              hide-details density="compact" clearable
              @click:clear="pkt.protocol = ''"
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
          <v-col cols="12" sm="6" md="4">
            <v-text-field
              v-model="pkt.source_ip_cidr"
              :label="$t('routeCheck.srcIp')"
              hide-details density="compact"
              placeholder="192.168.1.1"
              spellcheck="false"
            ></v-text-field>
          </v-col>
          <v-col cols="12" sm="6" md="2">
            <v-text-field
              v-model.number="pkt.port"
              :label="$t('routeCheck.destPort')"
              hide-details density="compact"
              type="number" min="1" max="65535"
            ></v-text-field>
          </v-col>
          <v-col cols="12" sm="6" md="2">
            <v-text-field
              v-model.number="pkt.source_port"
              :label="$t('routeCheck.srcPort')"
              hide-details density="compact"
              type="number" min="1" max="65535"
            ></v-text-field>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-checkbox
              v-model="pkt.ip_is_private"
              :label="$t('rule.privateIp')"
              hide-details density="compact"
            ></v-checkbox>
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
import { useI18n } from 'vue-i18n'
const { t } = useI18n()
import { ref, computed, reactive } from 'vue'

// ── Props ──────────────────────────────────────────────────
const props = defineProps<{
  rules: any[]
  finalOutbound: string
  inboundTags: string[]
  clients: string[]
}>()

// ── State ──────────────────────────────────────────────────
const dialog = ref(false)

const protocols = ['http', 'tls', 'quic', 'stun', 'dns', 'bittorrent', 'dtls', 'ssh', 'rdp', 'ntp']

const NON_TERMINATING = ['sniff', 'hijack-dns', 'resolve']

interface Packet {
  inbound: string
  auth_user: string
  network: string
  protocol: string
  domain: string
  ip_cidr: string
  source_ip_cidr: string
  port: number | null
  source_port: number | null
  ip_is_private: boolean
}

const defaultPkt = (): Packet => ({
  inbound: '', auth_user: '', network: '', protocol: '',
  domain: '', ip_cidr: '', source_ip_cidr: '',
  port: null, source_port: null, ip_is_private: false,
})

const pkt = reactive<Packet>(defaultPkt())

// rule_set checkboxes
const ruleSetMatches = reactive<Record<string, boolean>>({})

// All rule_set tags referenced in rules (recursive)
const referencedRuleSets = computed((): string[] => {
  const tags = new Set<string>()
  const collect = (ruleList: any[]) => {
    for (const r of ruleList) {
      if (r.rule_set) r.rule_set.forEach((t: string) => tags.add(t))
      if (r.rules) collect(r.rules)
    }
  }
  collect(props.rules)
  return [...tags]
})

// ── Result types ───────────────────────────────────────────
type Status = 'winner' | 'non_terminating' | 'no_match' | 'skipped'

interface RuleResult {
  index: number
  ruleType: string
  action: string
  outbound: string
  status: Status
  reason: string
}

const results = ref<RuleResult[]>([])
const winner = computed(() => results.value.find(r => r.status === 'winner') ?? null)

// ── Matching engine ────────────────────────────────────────

function matchesDomain(domain: string, rule: any): { matched: boolean; reason: string } {
  if (!domain) return { matched: false, reason: 'no domain in packet' }

  // exact
  if (rule.domain?.length) {
    const hit = rule.domain.find((d: string) => d.toLowerCase() === domain.toLowerCase())
    if (hit) return { matched: true, reason: `domain exact: "${hit}"` }
    if (!rule.domain_suffix && !rule.domain_keyword && !rule.domain_regex)
      return { matched: false, reason: `domain: no exact match in [${rule.domain.slice(0, 3).join(', ')}...]` }
  }
  // suffix
  if (rule.domain_suffix?.length) {
    const d = domain.toLowerCase()
    const hit = rule.domain_suffix.find((s: string) => {
      const suffix = s.toLowerCase().replace(/^\*\./, '')
      return d === suffix || d.endsWith('.' + suffix)
    })
    if (hit) return { matched: true, reason: `domain_suffix: "${hit}"` }
    return { matched: false, reason: `domain_suffix: "${domain}" doesn't match [${rule.domain_suffix.slice(0, 2).join(', ')}...]` }
  }
  // keyword
  if (rule.domain_keyword?.length) {
    const hit = rule.domain_keyword.find((k: string) => domain.toLowerCase().includes(k.toLowerCase()))
    if (hit) return { matched: true, reason: `domain_keyword: "${hit}"` }
    return { matched: false, reason: `domain_keyword: no match` }
  }
  // regex
  if (rule.domain_regex?.length) {
    const hit = rule.domain_regex.find((r: string) => { try { return new RegExp(r).test(domain) } catch { return false } })
    if (hit) return { matched: true, reason: `domain_regex: "${hit}"` }
    return { matched: false, reason: `domain_regex: no match` }
  }
  return { matched: true, reason: '' }
}

function matchesIpCidr(ip: string, cidrs: string[]): boolean {
  if (!cidrs?.length) return true
  if (!ip) return false

  return cidrs.some(cidr => {
    if (!cidr.includes('/')) return cidr === ip

    const [base, mask] = cidr.split('/')
    const octets = Math.floor(Number(mask) / 8)

    return ip.split('.').slice(0, octets).join('.') ===
           base.split('.').slice(0, octets).join('.')
  })
}

function matchesPortRange(port: number | null, ranges: string[]): boolean {
  if (!ranges?.length) return true
  if (port === null) return false

  return ranges.some(r => {
    const [a, b] = r.split(':').map(Number)
    return b ? port >= a && port <= b : port === a
  })
}

// Match a simple rule object against packet — returns { matched, reasons[] }
function matchSimpleRule(rule: any): { matched: boolean; reasons: string[] } {
  const reasons: string[] = []
  let matched = true

  reasons.push(`ruleCheck.start`)
console.log('rules in dialog:', props.rules)
// inbound
if (rule.inbound?.length) {
  if (!pkt.inbound) {
    matched = false
    reasons.push(t('routeCheck.inbound_missing'))
  } else if (!rule.inbound.includes(pkt.inbound)) {
    matched = false
    reasons.push(t('routeCheck.inbound_mismatch', {
      value: pkt.inbound,
      list: rule.inbound.join(', ')
    }))
  } else {
    reasons.push(t('routeCheck.inbound_ok'))
  }
} else {
  reasons.push(t('routeCheck.inbound_skipped'))
}

// auth_user
if (rule.auth_user?.length) {
  if (!pkt.auth_user) {
    matched = false
    reasons.push(`auth_user: packet has no value`)
  } else if (!rule.auth_user.includes(pkt.auth_user)) {
    matched = false
    reasons.push(`auth_user "${pkt.auth_user}" not in [${rule.auth_user.join(', ')}]`)
  } else {
    reasons.push(`auth_user ✓`)
  }
}

// network
if (rule.network?.length) {
  if (!pkt.network) {
    matched = false
    reasons.push(`network: packet has no value`)
  } else if (!rule.network.includes(pkt.network)) {
    matched = false
    reasons.push(`network "${pkt.network}" not in [${rule.network.join(', ')}]`)
  } else {
    reasons.push(`network ✓`)
  }
}

// protocol
if (rule.protocol?.length) {
  if (!pkt.protocol) {
    matched = false
    reasons.push(`protocol: packet has no value`)
  } else if (!rule.protocol.includes(pkt.protocol)) {
    matched = false
    reasons.push(`protocol "${pkt.protocol}" not in [${rule.protocol.join(', ')}]`)
  } else {
    reasons.push(`protocol ✓`)
  }
}

// ip_is_private
if (rule.ip_is_private !== undefined) {
  if (pkt.ip_is_private !== rule.ip_is_private) {
    matched = false
    reasons.push(`ip_is_private: packet=${pkt.ip_is_private}, rule=${rule.ip_is_private}`)
  } else {
    reasons.push(`ip_is_private ✓`)
  }
}

// domain
if (rule.domain?.length || rule.domain_suffix?.length || rule.domain_keyword?.length || rule.domain_regex?.length) {
  if (!pkt.domain) {
    matched = false
    reasons.push(`domain: packet has no value`)
  } else {
    const { matched: dm, reason } = matchesDomain(pkt.domain, rule)
    if (!dm) {
      matched = false
      reasons.push(reason)
    } else {
      reasons.push(reason || 'domain ✓')
    }
  }
}

// ip_cidr
if (rule.ip_cidr?.length) {
  if (!pkt.ip_cidr) {
    matched = false
    reasons.push(`ip_cidr: packet has no value`)
  } else if (!matchesIpCidr(pkt.ip_cidr, rule.ip_cidr)) {
    matched = false
    reasons.push(`ip_cidr: "${pkt.ip_cidr}" not in [${rule.ip_cidr.slice(0, 2).join(', ')}...]`)
  } else {
    reasons.push(`ip_cidr ✓`)
  }
}

// source_ip_cidr
if (rule.source_ip_cidr?.length) {
  if (!pkt.source_ip_cidr) {
    matched = false
    reasons.push(`source_ip_cidr: packet has no value`)
  } else if (!matchesIpCidr(pkt.source_ip_cidr, rule.source_ip_cidr)) {
    matched = false
    reasons.push(`source_ip_cidr: no match`)
  } else {
    reasons.push(`source_ip_cidr ✓`)
  }
}

// port
if (rule.port?.length) {
  if (pkt.port === null) {
    matched = false
    reasons.push(`port: packet has no value`)
  } else if (!rule.port.includes(pkt.port)) {
    matched = false
    reasons.push(`port ${pkt.port} not in [${rule.port.join(', ')}]`)
  } else {
    reasons.push(`port ✓`)
  }
}

// port_range
if (rule.port_range?.length) {
  if (pkt.port === null) {
    matched = false
    reasons.push(`port_range: packet has no value`)
  } else if (!matchesPortRange(pkt.port, rule.port_range)) {
    matched = false
    reasons.push(`port_range: no match`)
  } else {
    reasons.push(`port_range ✓`)
  }
}

// source_port
if (rule.source_port?.length) {
  if (pkt.source_port === null) {
    matched = false
    reasons.push(`source_port: packet has no value`)
  } else if (!rule.source_port.includes(pkt.source_port)) {
    matched = false
    reasons.push(`source_port ${pkt.source_port} not in [${rule.source_port.join(', ')}]`)
  } else {
    reasons.push(`source_port ✓`)
  }
}

// source_port_range
if (rule.source_port_range?.length) {
  if (pkt.source_port === null) {
    matched = false
    reasons.push(`source_port_range: packet has no value`)
  } else if (!matchesPortRange(pkt.source_port, rule.source_port_range)) {
    matched = false
    reasons.push(`source_port_range: no match`)
  } else {
    reasons.push(`source_port_range ✓`)
  }
}

// rule_set
if (rule.rule_set?.length) {
  const missing = rule.rule_set.filter((tag: string) => ruleSetMatches[tag] !== true)

  if (missing.length) {
    matched = false
    reasons.push(`rule_set: not matched [${missing.join(', ')}]`)
  } else {
    reasons.push(`rule_set ✓`)
  }
}

  return { matched, reasons }
}

// Match logical rule (recursive)
function matchLogicalRule(rule: any): { matched: boolean; reasons: string[] } {
  const subRules: any[] = rule.rules ?? []
  const mode: string = rule.mode ?? 'and'
  const subResults = subRules.map(r => matchSimpleRule(r))

  let matched: boolean
  if (mode === 'or') {
    matched = subResults.some(r => r.matched)
  } else {
    matched = subResults.every(r => r.matched)
  }

  const reasons = [`logical (${mode}): ` + subResults.map((r, i) => `rule${i + 1}=${r.matched ? '✓' : '✗'}`).join(', ')]
  return { matched, reasons }
}

function matchRule(rule: any): { matched: boolean; reasons: string[] } {
  if (rule.type === 'logical') {
    return matchLogicalRule(rule)
  }
  return matchSimpleRule(rule)
}

// ── Run ────────────────────────────────────────────────────

function runCheck() {
  results.value = []
  let terminated = false

  for (let i = 0; i < props.rules.length; i++) {
    const rule = props.rules[i]
    const action: string = rule.action ?? 'route'
    const outbound: string = rule.outbound ?? ''
    const ruleType = rule.type === 'logical' ? `logical (${rule.mode ?? 'and'})` : 'simple'

    if (terminated) {
      results.value.push({
        index: i, ruleType, action, outbound,
        status: 'skipped',
        reason: 'earlier rule already matched',
      })
      continue
    }

    const { matched, reasons } = matchRule(rule)
    let effectiveMatch = rule.invert ? !matched : matched

    if (effectiveMatch) {
      if (NON_TERMINATING.includes(action)) {
        results.value.push({
          index: i, ruleType, action, outbound,
          status: 'non_terminating',
          reason: reasons.join('; ') + (rule.invert ? ' [inverted]' : ''),
        })
        // continue — don't terminate
      } else {
        results.value.push({
          index: i, ruleType, action, outbound,
          status: 'winner',
          reason: reasons.join('; ') + (rule.invert ? ' [inverted]' : ''),
        })
        terminated = true
      }
    } else {
      results.value.push({
        index: i, ruleType, action, outbound,
        status: 'no_match',
        reason: reasons.join('; ') + (rule.invert ? ' [inverted]' : ''),
      })
    }
  }
}

// ── Helpers ────────────────────────────────────────────────

function statusColor(status: Status): string {
  return { winner: 'success', non_terminating: 'info', no_match: 'error', skipped: 'default' }[status]
}

function rowStyle(r: RuleResult): string {
  if (r.status === 'winner') return 'background: rgba(var(--v-theme-success), 0.06);'
  if (r.status === 'skipped') return 'opacity: 0.4;'
  return ''
}

function reset() {
  Object.assign(pkt, defaultPkt())
  results.value = []
  Object.keys(ruleSetMatches).forEach(k => delete ruleSetMatches[k])
}

function open() {
  reset()
  dialog.value = true
}

defineExpose({ open })
</script>
