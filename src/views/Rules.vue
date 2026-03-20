<template>
  <RuleVue
    v-model="ruleModal.visible"
    :visible="ruleModal.visible"
    :index="ruleModal.index"
    :data="ruleModal.data"
    :clients="clients"
    :inTags="inboundTags"
    :outTags="outboundTags"
    :rsTags="rulesetTags"
    @close="closeRuleModal"
    @save="saveRuleModal"
  />
  <RulesetVue
    v-model="rulesetModal.visible"
    :visible="rulesetModal.visible"
    :index="rulesetModal.index"
    :data="rulesetModal.data"
    :outTags="outboundTags"
    @close="closeRulesetModal"
    @save="saveRulesetModal"
  />

  <!-- ───────────── Bulk Import Rulesets Dialog ───────────── -->
  <v-dialog v-model="importDialog" max-width="700" scrollable>
    <v-card>
      <v-card-title class="d-flex align-center pa-4">
        <v-icon icon="mdi-download-multiple" class="mr-2" />
        Bulk import rulesets
        <v-spacer />
        <v-chip v-if="importPreview.length > 0" size="small" color="primary" variant="tonal">
          {{ importPreview.length }} URLs
        </v-chip>
      </v-card-title>
      <v-divider />
      <v-card-text class="pa-4">
        <v-tabs v-model="importTab" class="mb-4">
          <v-tab value="text"><v-icon icon="mdi-text" class="mr-1" size="small" />Paste URLs</v-tab>
          <v-tab value="file"><v-icon icon="mdi-file-upload" class="mr-1" size="small" />Upload .txt file</v-tab>
        </v-tabs>
        <v-window v-model="importTab">
          <v-window-item value="text">
            <p class="text-caption text-medium-emphasis mb-2">One URL per line. Tag is parsed from filename without extension.</p>
            <v-textarea v-model="importRawText" label="URLs" variant="outlined" rows="10" auto-grow hide-details spellcheck="false"
              placeholder="https://github.com/.../geoip-telegram.srs&#10;https://github.com/.../geosite-youtube.srs"
              style="font-family: monospace; font-size: 13px;"></v-textarea>
          </v-window-item>
          <v-window-item value="file">
            <p class="text-caption text-medium-emphasis mb-2">Upload a .txt file with one URL per line.</p>
            <v-file-input label="Select .txt file" accept=".txt" variant="outlined" hide-details prepend-icon="mdi-file-document" @change="onFileUpload"></v-file-input>
          </v-window-item>
        </v-window>
        <v-row class="mt-4">
          <v-col cols="12" sm="6" md="4">
            <v-select v-model="importFormat" :items="['binary', 'source']" label="Format" variant="outlined" hide-details density="compact"></v-select>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-select v-model="importDetour" :items="[{ title: '— none —', value: '' }, ...outboundTags]" label="Download detour" variant="outlined" hide-details density="compact" clearable></v-select>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-text-field v-model.number="importInterval" label="Update interval (days)" type="number" min="0" variant="outlined" hide-details density="compact"></v-text-field>
          </v-col>
        </v-row>
        <template v-if="importPreview.length > 0">
          <v-divider class="my-4" />
          <p class="text-caption text-medium-emphasis mb-2">Preview — {{ importPreview.length }} rulesets to import ({{ importSkipped }} already exist, shown in grey)</p>
          <v-table density="compact" style="font-size: 13px;">
            <thead><tr><th>Tag</th><th>Format</th><th>URL</th><th style="width:36px;"></th></tr></thead>
            <tbody>
              <tr v-for="(item, i) in importPreview" :key="i" :style="item.exists ? 'opacity:0.4' : ''">
                <td><v-text-field v-model="item.tag" variant="plain" hide-details density="compact" style="font-family: monospace; min-width: 140px;"></v-text-field></td>
                <td style="white-space: nowrap;">{{ item.format }}</td>
                <td style="font-family: monospace; word-break: break-all; font-size: 11px;">{{ item.url }}</td>
                <td><v-btn icon="mdi-close" size="x-small" variant="text" color="error" @click="importPreview.splice(i, 1)" /></td>
              </tr>
            </tbody>
          </v-table>
        </template>
      </v-card-text>
      <v-divider />
      <v-card-actions class="pa-3">
        <v-btn @click="importDialog = false" variant="text">Cancel</v-btn>
        <v-spacer />
        <v-btn @click="parseImport" variant="tonal" :disabled="importRawText.trim().length === 0">
          <v-icon icon="mdi-magnify" class="mr-1" />Parse
        </v-btn>
        <v-btn @click="applyImport" color="primary" variant="flat" :disabled="importPreview.filter(i => !i.exists).length === 0">
          <v-icon icon="mdi-download-multiple" class="mr-1" />
          Import{{ importCount }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>

  <!-- ───────────── Import Route Dialog ───────────── -->
  <v-dialog v-model="routeImportDialog" max-width="750" scrollable>
    <v-card>
      <v-card-title class="d-flex align-center pa-4">
        <v-icon icon="mdi-routes" class="mr-2" />
        Import rules
        <v-spacer />
        <v-chip v-if="routeImportParsed" size="small" color="primary" variant="tonal">
          {{ routeImportParsed.rules?.length ?? 0 }} rules · {{ routeImportParsed.rule_set?.length ?? 0 }} rulesets
        </v-chip>
      </v-card-title>
      <v-divider />
      <v-card-text class="pa-4">
        <v-tabs v-model="routeImportTab" class="mb-4">
          <v-tab value="json"><v-icon icon="mdi-code-json" class="mr-1" size="small" />Paste JSON</v-tab>
          <v-tab value="file"><v-icon icon="mdi-file-upload" class="mr-1" size="small" />Upload file</v-tab>
          <v-tab value="url"><v-icon icon="mdi-link" class="mr-1" size="small" />From URL</v-tab>
        </v-tabs>

        <v-window v-model="routeImportTab">
          <!-- Paste JSON -->
          <v-window-item value="json">
            <p class="text-caption text-medium-emphasis mb-2">
              Paste a JSON object with <code>rules</code> and/or <code>rule_set</code> arrays.
              You can paste the whole <code>"route": {...}</code> block or just the inner object.
            </p>
            <v-textarea v-model="routeImportRaw" label="JSON" variant="outlined" rows="12" hide-details spellcheck="false"
              style="font-family: monospace; font-size: 12px;" :error="!!routeImportError"></v-textarea>
            <p v-if="routeImportError" class="text-caption text-error mt-1">{{ routeImportError }}</p>
            <div class="d-flex justify-end mt-3">
              <v-btn @click="parseRouteImport" variant="tonal" :disabled="routeImportRaw.trim().length === 0">
                <v-icon icon="mdi-magnify" class="mr-1" />Parse
              </v-btn>
            </div>
          </v-window-item>

          <!-- Upload file -->
          <v-window-item value="file">
            <p class="text-caption text-medium-emphasis mb-2">Upload a <code>.json</code> file with the route block.</p>
            <v-file-input label="Select .json file" accept=".json" variant="outlined" hide-details prepend-icon="mdi-file-code" @change="onRouteFileUpload"></v-file-input>
            <p v-if="routeImportError" class="text-caption text-error mt-2">{{ routeImportError }}</p>
          </v-window-item>

          <!-- From URL -->
          <v-window-item value="url">
            <p class="text-caption text-medium-emphasis mb-2">Fetch a raw JSON file from a URL (e.g. GitHub raw link).</p>
            <div class="d-flex gap-2 align-start">
              <v-text-field v-model="routeImportUrl" label="URL" variant="outlined" hide-details spellcheck="false"
                placeholder="https://raw.githubusercontent.com/..." class="flex-grow-1" :error="!!routeImportError"></v-text-field>
              <v-btn @click="fetchRouteFromUrl" variant="tonal" :loading="routeImportFetching" style="height: 56px;">Fetch</v-btn>
            </div>
            <p v-if="routeImportError" class="text-caption text-error mt-1">{{ routeImportError }}</p>
          </v-window-item>
        </v-window>

        <!-- Preview -->
        <template v-if="routeImportParsed">
          <v-divider class="my-4" />

          <!-- Conflict warning -->
          <v-alert v-if="routeImportHasConflicts" type="warning" variant="tonal" class="mb-4" title="Conflicts detected">
            Your config already has {{ rules.length }} rules and {{ rulesets.length }} rulesets. Choose how to handle them:
            <v-radio-group v-model="routeImportMode" hide-details class="mt-2">
              <v-radio value="merge" label="Merge — append imported rules/rulesets (skip duplicate ruleset tags)" />
              <v-radio value="replace" label="Replace — remove existing rules and rulesets, import fresh" />
            </v-radio-group>
          </v-alert>

          <!-- final -->
          <div v-if="routeImportParsed.final" class="d-flex align-center mb-3 gap-2">
            <span class="text-caption text-medium-emphasis">Default outbound (final):</span>
            <v-chip size="small" color="secondary" variant="tonal">{{ routeImportParsed.final }}</v-chip>
            <v-checkbox v-model="routeImportApplyFinal" label="Apply as default outbound" hide-details density="compact" />
          </div>

          <!-- Rules preview -->
          <p class="text-caption text-medium-emphasis mb-1">Rules: <strong>{{ routeImportParsed.rules?.length ?? 0 }}</strong></p>
          <v-table v-if="routeImportParsed.rules?.length" density="compact" style="font-size: 12px;" class="mb-4">
            <thead><tr><th>#</th><th>Type</th><th>Action</th><th>Outbound</th></tr></thead>
            <tbody>
              <tr v-for="(r, i) in routeImportParsed.rules" :key="i">
                <td>{{ (i as number) + 1 }}</td>
                <td>{{ r.type ?? 'simple' }}</td>
                <td>{{ r.action }}</td>
                <td>{{ r.outbound ?? '-' }}</td>
              </tr>
            </tbody>
          </v-table>

          <!-- Rulesets preview -->
          <p class="text-caption text-medium-emphasis mb-1">
            Rulesets: <strong>{{ routeImportParsed.rule_set?.length ?? 0 }}</strong>
            <span v-if="routeImportSkippedRulesets > 0" class="text-warning ml-2">
              ({{ routeImportSkippedRulesets }} duplicate tags skipped in merge mode)
            </span>
          </p>
          <v-table v-if="routeImportParsed.rule_set?.length" density="compact" style="font-size: 12px;">
            <thead><tr><th>Tag</th><th>Format</th><th>Type</th><th>Interval</th></tr></thead>
            <tbody>
              <tr v-for="(rs, i) in routeImportParsed.rule_set" :key="i"
                :style="routeImportMode === 'merge' && rulesetTags.includes(rs.tag) ? 'opacity:0.4' : ''">
                <td style="font-family: monospace;">{{ rs.tag }}</td>
                <td>{{ rs.format }}</td>
                <td>{{ rs.type }}</td>
                <td>{{ rs.update_interval ?? '-' }}</td>
              </tr>
            </tbody>
          </v-table>
        </template>
      </v-card-text>
      <v-divider />
      <v-card-actions class="pa-3">
        <v-btn @click="closeRouteImport" variant="text">Cancel</v-btn>
        <v-spacer />
        <v-btn @click="applyRouteImport" color="primary" variant="flat" :disabled="!routeImportParsed">
          <v-icon icon="mdi-check" class="mr-1" />
          {{ routeImportMode === 'replace' ? 'Replace & import' : 'Merge & import' }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
  <!-- ──────────────────────────────────────────── -->

  <v-row>
    <v-col cols="12" justify="center" align="center">
      <v-btn color="primary" @click="showRuleModal(-1)" style="margin: 0 5px;">{{ $t('rule.add') }}</v-btn>
      <v-btn color="primary" @click="showRulesetModal(-1)" style="margin: 0 5px;">{{ $t('ruleset.add') }}</v-btn>
      <v-btn color="secondary" @click="openImportDialog" style="margin: 0 5px;">
        <v-icon icon="mdi-download-multiple" class="mr-1" />Import rulesets
      </v-btn>
      <v-btn color="secondary" @click="openRouteImportDialog" style="margin: 0 5px;">
        <v-icon icon="mdi-routes" class="mr-1" />Import rules
      </v-btn>
      <v-btn variant="outlined" color="warning" @click="saveConfig" :loading="loading" :disabled="stateChange">
        {{ $t('actions.save') }}
      </v-btn>
    </v-col>
  </v-row>
  <v-row>
    <v-col class="v-card-subtitle" cols="12">{{ $t('basic.routing.title') }}</v-col>
    <v-col cols="12">
      <v-row>
        <v-col cols="12" sm="6" md="3" lg="2">
          <v-select hide-details :label="$t('basic.routing.defaultOut')" clearable @click:clear="delete route.final" :items="outboundTags" v-model="route.final"></v-select>
        </v-col>
        <v-col cols="12" sm="6" md="3" lg="2">
          <v-text-field v-model="route.default_interface" hide-details clearable @click:clear="delete route.default_interface" :label="$t('basic.routing.defaultIf')"></v-text-field>
        </v-col>
        <v-col cols="12" sm="6" md="3" lg="2">
          <v-text-field v-model.number="routeMark" hide-details type="number" min="0" :label="$t('basic.routing.defaultRm')"></v-text-field>
        </v-col>
        <v-col cols="12" sm="6" md="3" lg="2">
          <v-switch v-model="route.auto_detect_interface" color="primary" :label="$t('basic.routing.autoBind')" hide-details></v-switch>
        </v-col>
      </v-row>
    </v-col>
  </v-row>
  <v-row>
    <v-col class="v-card-subtitle" cols="12">{{ $t('rule.ruleset') }}</v-col>
    <v-col cols="12" sm="4" md="3" lg="2" v-for="(item, index) in <any[]>rulesets" :key="item.tag">
      <v-card rounded="xl" elevation="5" min-width="200" :title="item.tag">
        <v-card-subtitle style="margin-top: -15px;">
          <v-row><v-col>{{ $t('ruleset.' + item.type) }}</v-col></v-row>
        </v-card-subtitle>
        <v-card-text>
          <v-row><v-col>{{ $t('ruleset.format') }}</v-col><v-col>{{ item.format }}</v-col></v-row>
          <v-row><v-col>{{ $t('actions.update') }}</v-col><v-col>{{ item.update_interval ?? '-' }}</v-col></v-row>
        </v-card-text>
        <v-divider></v-divider>
        <v-card-actions style="padding: 0;">
          <v-btn icon="mdi-file-edit" @click="showRulesetModal(index)"><v-icon /><v-tooltip activator="parent" location="top" :text="$t('actions.edit')"></v-tooltip></v-btn>
          <v-btn icon="mdi-file-remove" style="margin-inline-start:0;" color="warning" @click="delRulesetOverlay[index] = true"><v-icon /><v-tooltip activator="parent" location="top" :text="$t('actions.del')"></v-tooltip></v-btn>
          <v-overlay v-model="delRulesetOverlay[index]" contained class="align-center justify-center">
            <v-card :title="$t('actions.del')" rounded="lg">
              <v-divider></v-divider>
              <v-card-text>{{ $t('confirm') }}</v-card-text>
              <v-card-actions>
                <v-btn color="error" variant="outlined" @click="delRuleset(index)">{{ $t('yes') }}</v-btn>
                <v-btn color="success" variant="outlined" @click="delRulesetOverlay[index] = false">{{ $t('no') }}</v-btn>
              </v-card-actions>
            </v-card>
          </v-overlay>
        </v-card-actions>
      </v-card>
    </v-col>
  </v-row>
  <v-row>
    <v-col class="v-card-subtitle" cols="12">{{ $t('pages.rules') }}</v-col>
    <v-col cols="12" sm="4" md="3" lg="2" v-for="(item, index) in <any[]>rules"
        :key="item.id" :draggable="true"
        @dragstart="onDragStart(index)" @dragover.prevent @drop="onDrop(index)">
      <v-card rounded="xl" elevation="5" min-width="200" :title="index+1">
        <v-card-subtitle style="margin-top: -15px;">
          <v-row><v-col>{{ item.type != undefined ? $t('rule.logical') + ' (' + item.mode + ')' : $t('rule.simple') }}</v-col></v-row>
        </v-card-subtitle>
        <v-card-text>
          <v-row><v-col>{{ $t('admin.action') }}</v-col><v-col>{{ item.action }}</v-col></v-row>
          <v-row><v-col>{{ $t('objects.outbound') }}</v-col><v-col>{{ item.outbound ?? '-' }}</v-col></v-row>
          <v-row><v-col>{{ $t('pages.rules') }}</v-col><v-col>{{ item.rules ? item.rules.length : Object.keys(item).filter(r => !actionKeys.includes(r)).length }}</v-col></v-row>
          <v-row><v-col>{{ $t('rule.invert') }}</v-col><v-col>{{ $t((item.invert ?? false) ? 'yes' : 'no') }}</v-col></v-row>
        </v-card-text>
        <v-divider></v-divider>
        <v-card-actions style="padding: 0;">
          <v-btn icon="mdi-file-edit" @click="showRuleModal(index)"><v-icon /><v-tooltip activator="parent" location="top" :text="$t('actions.edit')"></v-tooltip></v-btn>
          <v-btn icon="mdi-file-remove" style="margin-inline-start:0;" color="warning" @click="delRuleOverlay[index] = true"><v-icon /><v-tooltip activator="parent" location="top" :text="$t('actions.del')"></v-tooltip></v-btn>
          <v-overlay v-model="delRuleOverlay[index]" contained class="align-center justify-center">
            <v-card :title="$t('actions.del')" rounded="lg">
              <v-divider></v-divider>
              <v-card-text>{{ $t('confirm') }}</v-card-text>
              <v-card-actions>
                <v-btn color="error" variant="outlined" @click="delRule(index)">{{ $t('yes') }}</v-btn>
                <v-btn color="success" variant="outlined" @click="delRuleOverlay[index] = false">{{ $t('no') }}</v-btn>
              </v-card-actions>
            </v-card>
          </v-overlay>
        </v-card-actions>
      </v-card>
    </v-col>
  </v-row>
</template>

<script lang="ts" setup>
import Data from '@/store/modules/data'
import { computed, ref, onBeforeMount } from 'vue'
import RuleVue from '@/layouts/modals/Rule.vue'
import RulesetVue from '@/layouts/modals/Ruleset.vue'
import { Config } from '@/types/config'
import { actionKeys, ruleset } from '@/types/rules'
import { FindDiff } from '@/plugins/utils'

const oldConfig = ref({})
const loading = ref(false)
const appConfig = computed((): Config => {
  return <Config> Data().config
})

onBeforeMount(async () => {
  loading.value = true
  while (Data().lastLoad == 0) {
    await new Promise(resolve => setTimeout(resolve, 100))
  }
  oldConfig.value = JSON.parse(JSON.stringify(Data().config))
  loading.value = false
})

const routeMark = computed({
  get() { return route.value.default_mark ?? 0 },
  set(v:number) { v>0 ? route.value.default_mark = v : delete appConfig.value.route.default_mark }
})

const stateChange = computed(() => FindDiff.deepCompare(appConfig.value, oldConfig.value))

const saveConfig = async () => {
  loading.value = true
  const success = await Data().save("config", "set", appConfig.value)
  if (success) {
    oldConfig.value = JSON.parse(JSON.stringify(Data().config))
    loading.value = false
  }
}

const clients = computed((): string[] => Data().clients.map((c:any) => c.name))
const route = computed((): any => appConfig.value.route ?? {})

const rules = computed((): any[] => {
  const data = route.value
  if (!data) return []
  if (!('rules' in data) || !Array.isArray(data.rules)) data.rules = []
  return data.rules
})

const rulesets = computed((): any[] => {
  const data = route.value
  if (!data) return []
  if (!('rule_set' in data) || !Array.isArray(data.rule_set)) data.rule_set = []
  return data.rule_set
})

const rulesetTags = computed((): string[] => rulesets.value.map((rs:any) => rs.tag))

const outboundTags = computed((): string[] => [
  ...Data().outbounds?.map((o:any) => o.tag),
  ...Data().endpoints?.map((e:any) => e.tag)
])

const inboundTags = computed((): string[] => [
  ...Data().inbounds?.map((o:any) => o.tag),
  ...Data().endpoints?.filter((e:any) => e.listen_port > 0).map((e:any) => e.tag)
])

let delRuleOverlay = ref(new Array<boolean>)
let delRulesetOverlay = ref(new Array<boolean>)

const ruleModal = ref({ visible: false, index: -1, data: "" })
const showRuleModal = (index: number) => {
  ruleModal.value.index = index
  ruleModal.value.data = index == -1 ? '' : JSON.stringify(rules.value[index])
  ruleModal.value.visible = true
}
const closeRuleModal = () => { ruleModal.value.visible = false }
const saveRuleModal = (data:any) => {
  if (ruleModal.value.index == -1) rules.value.push(data)
  else rules.value[ruleModal.value.index] = data
  ruleModal.value.visible = false
}
const delRule = (index: number) => { rules.value.splice(index, 1); delRuleOverlay.value[index] = false }

const rulesetModal = ref({ visible: false, index: -1, data: "" })
const showRulesetModal = (index: number) => {
  rulesetModal.value.index = index
  rulesetModal.value.data = index == -1 ? '' : JSON.stringify(rulesets.value[index])
  rulesetModal.value.visible = true
}
const closeRulesetModal = () => { rulesetModal.value.visible = false }
const saveRulesetModal = (data:ruleset) => {
  if (rulesetModal.value.index == -1) rulesets.value.push(data)
  else rulesets.value[rulesetModal.value.index] = data
  rulesetModal.value.visible = false
}
const delRuleset = (index: number) => { rulesets.value.splice(index, 1); delRulesetOverlay.value[index] = false }

const draggedItemIndex = ref(null)
const onDragStart = (index: any) => { draggedItemIndex.value = index }
const onDrop = (index: any) => {
  if (draggedItemIndex.value !== null) {
    const draggedItem = rules.value[draggedItemIndex.value]
    rules.value.splice(draggedItemIndex.value, 1)
    rules.value.splice(index, 0, draggedItem)
    draggedItemIndex.value = null
  }
}

// ─────────────────────────────────────────────
// Bulk Import Rulesets
// ─────────────────────────────────────────────

const importDialog = ref(false)
const importTab = ref('text')
const importRawText = ref('')
const importFormat = ref('binary')
const importDetour = ref('')
const importInterval = ref(1)

interface ImportItem { tag: string; url: string; format: string; exists: boolean }
const importPreview = ref<ImportItem[]>([])
const importCount = computed((): string => {
  const n = importPreview.value.filter((i: ImportItem) => !i.exists).length
  return n > 0 ? ` ${n}` : ''
})
const importSkipped = computed(() => importPreview.value.filter(i => i.exists).length)

function urlToTag(url: string): string {
  try {
    const filename = new URL(url).pathname.split('/').pop() ?? ''
    return filename.replace(/\.[^.]+$/, '')
  } catch {
    const parts = url.split('/')
    return parts[parts.length - 1].replace(/\.[^.]+$/, '') || url
  }
}

function openImportDialog() {
  importRawText.value = ''; importPreview.value = []; importTab.value = 'text'; importDialog.value = true
}

function parseImport() {
  const existingTags = new Set(rulesetTags.value)
  const seen = new Set<string>()
  importPreview.value = importRawText.value
    .split('\n').map(l => l.trim()).filter(l => l.length > 0 && l.startsWith('http'))
    .filter(url => { if (seen.has(url)) return false; seen.add(url); return true })
    .map(url => ({ tag: urlToTag(url), url, format: importFormat.value, exists: existingTags.has(urlToTag(url)) }))
}

function applyImport() {
  for (const item of importPreview.value.filter(i => !i.exists)) {
    const rs: any = { type: 'remote', tag: item.tag, format: item.format, url: item.url }
    if (importDetour.value) rs.download_detour = importDetour.value
    if (importInterval.value > 0) rs.update_interval = importInterval.value + 'd'
    rulesets.value.push(rs)
  }
  importDialog.value = false
}

async function onFileUpload(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return
  importRawText.value = await file.text()
  parseImport()
}

// ─────────────────────────────────────────────
// Import Route (rules + rule_set)
// ─────────────────────────────────────────────

const routeImportDialog = ref(false)
const routeImportTab = ref('json')
const routeImportRaw = ref('')
const routeImportUrl = ref('')
const routeImportFetching = ref(false)
const routeImportError = ref('')
const routeImportParsed = ref<any>(null)
const routeImportMode = ref<'merge' | 'replace'>('merge')
const routeImportApplyFinal = ref(false)

const routeImportHasConflicts = computed(() => rules.value.length > 0 || rulesets.value.length > 0)

const routeImportSkippedRulesets = computed(() => {
  if (!routeImportParsed.value?.rule_set) return 0
  const existing = new Set(rulesetTags.value)
  return routeImportParsed.value.rule_set.filter((rs: any) => existing.has(rs.tag)).length
})

function openRouteImportDialog() {
  routeImportRaw.value = ''; routeImportUrl.value = ''; routeImportError.value = ''
  routeImportParsed.value = null; routeImportTab.value = 'json'
  routeImportMode.value = routeImportHasConflicts.value ? 'merge' : 'replace'
  routeImportApplyFinal.value = false; routeImportDialog.value = true
}

function closeRouteImport() { routeImportDialog.value = false }

function extractRouteBlock(obj: any): any {
  if (obj?.route && (obj.route.rules || obj.route.rule_set)) return obj.route
  if (obj?.rules || obj?.rule_set) return obj
  return null
}

function parseRouteImport() {
  routeImportError.value = ''; routeImportParsed.value = null
  try {
    const block = extractRouteBlock(JSON.parse(routeImportRaw.value))
    if (!block) { routeImportError.value = 'No "rules" or "rule_set" arrays found in the pasted JSON.'; return }
    routeImportParsed.value = block
    if (!routeImportHasConflicts.value) routeImportMode.value = 'replace'
  } catch (e: any) {
    routeImportError.value = 'JSON parse error: ' + e.message
  }
}

async function fetchRouteFromUrl() {
  routeImportError.value = ''; routeImportParsed.value = null; routeImportFetching.value = true
  try {
    const resp = await fetch(routeImportUrl.value)
    if (!resp.ok) throw new Error(`HTTP ${resp.status}`)
    const block = extractRouteBlock(await resp.json())
    if (!block) { routeImportError.value = 'No "rules" or "rule_set" found in fetched JSON.' }
    else { routeImportParsed.value = block; if (!routeImportHasConflicts.value) routeImportMode.value = 'replace' }
  } catch (e: any) {
    routeImportError.value = 'Fetch error: ' + e.message
  } finally {
    routeImportFetching.value = false
  }
}

async function onRouteFileUpload(event: Event) {
  routeImportError.value = ''; routeImportParsed.value = null
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return
  try {
    const block = extractRouteBlock(JSON.parse(await file.text()))
    if (!block) { routeImportError.value = 'No "rules" or "rule_set" found in the file.' }
    else { routeImportParsed.value = block; if (!routeImportHasConflicts.value) routeImportMode.value = 'replace' }
  } catch (e: any) {
    routeImportError.value = 'JSON parse error: ' + e.message
  }
}

function applyRouteImport() {
  const block = routeImportParsed.value
  if (!block) return
  if (routeImportMode.value === 'replace') {
    route.value.rules = block.rules ?? []
    route.value.rule_set = block.rule_set ?? []
  } else {
    const existingTags = new Set(rulesetTags.value)
    if (block.rules) rules.value.push(...block.rules)
    if (block.rule_set) {
      for (const rs of block.rule_set) {
        if (!existingTags.has(rs.tag)) rulesets.value.push(rs)
      }
    }
  }
  if (routeImportApplyFinal.value && block.final) route.value.final = block.final
  routeImportDialog.value = false
}
</script>
