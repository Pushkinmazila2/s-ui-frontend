<template>
  <v-dialog v-model="dialog" max-width="750" scrollable>
    <v-card>
      <v-card-title class="d-flex align-center pa-4">
        <v-icon icon="mdi-routes" class="mr-2" />
        {{ $t('ruleset.importRulesTitle') }}
        <v-spacer />
        <v-chip v-if="parsed" size="small" color="primary" variant="tonal">
          {{ parsed.rules?.length ?? 0 }} {{ $t('pages.rules') }} · {{ parsed.rule_set?.length ?? 0 }} {{ $t('rule.ruleset') }}
        </v-chip>
      </v-card-title>
      <v-divider />
      <v-card-text class="pa-4">
        <v-tabs v-model="tab" class="mb-4">
          <v-tab value="json">
            <v-icon icon="mdi-code-json" class="mr-1" size="small" />{{ $t('ruleset.importPasteJson') }}
          </v-tab>
          <v-tab value="file">
            <v-icon icon="mdi-file-upload" class="mr-1" size="small" />{{ $t('ruleset.importUploadFile') }}
          </v-tab>
          <v-tab value="url">
            <v-icon icon="mdi-link" class="mr-1" size="small" />{{ $t('ruleset.importFromUrl') }}
          </v-tab>
        </v-tabs>

        <v-window v-model="tab">
          <v-window-item value="json">
            <p class="text-caption text-medium-emphasis mb-2">{{ $t('ruleset.importJsonHint') }}</p>
            <v-textarea v-model="rawJson" label="JSON" variant="outlined" rows="12" hide-details spellcheck="false"
              style="font-family: monospace; font-size: 12px;" :error="!!error"></v-textarea>
            <p v-if="error" class="text-caption text-error mt-1">{{ error }}</p>
            <div class="d-flex justify-end mt-3">
              <v-btn @click="parseJson" variant="tonal" :disabled="rawJson.trim().length === 0">
                <v-icon icon="mdi-magnify" class="mr-1" />{{ $t('ruleset.importParse') }}
              </v-btn>
            </div>
          </v-window-item>

          <v-window-item value="file">
            <p class="text-caption text-medium-emphasis mb-2">{{ $t('ruleset.importFileJsonHint') }}</p>
            <v-file-input :label="$t('ruleset.importSelectJson')" accept=".json" variant="outlined"
              hide-details prepend-icon="mdi-file-code" @change="onFileUpload"></v-file-input>
            <p v-if="error" class="text-caption text-error mt-2">{{ error }}</p>
          </v-window-item>

          <v-window-item value="url">
            <p class="text-caption text-medium-emphasis mb-2">{{ $t('ruleset.importUrlHint') }}</p>
            <div class="d-flex gap-2 align-start">
              <v-text-field v-model="fetchUrl" label="URL" variant="outlined" hide-details spellcheck="false"
                placeholder="https://raw.githubusercontent.com/..." class="flex-grow-1" :error="!!error"></v-text-field>
              <v-btn @click="fetchFromUrl" variant="tonal" :loading="fetching" style="height: 56px;">
                {{ $t('ruleset.importFetch') }}
              </v-btn>
            </div>
            <p v-if="error" class="text-caption text-error mt-1">{{ error }}</p>
          </v-window-item>
        </v-window>

        <template v-if="parsed">
          <v-divider class="my-4" />

          <v-alert v-if="hasConflicts" type="warning" variant="tonal" class="mb-4" :title="$t('ruleset.importConflict')">
            {{ $t('ruleset.importConflictDesc', { rules: existingRulesCount, rulesets: existingRulesetsCount }) }}
            <v-radio-group v-model="mode" hide-details class="mt-2">
              <v-radio value="merge" :label="$t('ruleset.importMerge')" />
              <v-radio value="replace" :label="$t('ruleset.importReplace')" />
            </v-radio-group>
          </v-alert>

          <div v-if="parsed.final" class="d-flex align-center mb-3 gap-2">
            <span class="text-caption text-medium-emphasis">{{ $t('ruleset.importFinalOutbound') }}:</span>
            <v-chip size="small" color="secondary" variant="tonal">{{ parsed.final }}</v-chip>
            <v-checkbox v-model="applyFinal" :label="$t('ruleset.importApplyFinal')" hide-details density="compact" />
          </div>

          <p class="text-caption text-medium-emphasis mb-1">
            {{ $t('pages.rules') }}: <strong>{{ parsed.rules?.length ?? 0 }}</strong>
          </p>
          <v-table v-if="parsed.rules?.length" density="compact" style="font-size: 12px;" class="mb-4">
            <thead>
              <tr><th>#</th><th>{{ $t('type') }}</th><th>{{ $t('admin.action') }}</th><th>{{ $t('objects.outbound') }}</th></tr>
            </thead>
            <tbody>
              <tr v-for="(r, i) in parsed.rules" :key="i">
                <td>{{ (i as number) + 1 }}</td>
                <td>{{ r.type ?? 'simple' }}</td>
                <td>{{ r.action }}</td>
                <td>{{ r.outbound ?? '-' }}</td>
              </tr>
            </tbody>
          </v-table>

          <p class="text-caption text-medium-emphasis mb-1">
            {{ $t('rule.ruleset') }}: <strong>{{ parsed.rule_set?.length ?? 0 }}</strong>
            <span v-if="skippedRulesets > 0" class="text-warning ml-2">
              ({{ skippedRulesets }} {{ $t('ruleset.importSkipped') }})
            </span>
          </p>
          <v-table v-if="parsed.rule_set?.length" density="compact" style="font-size: 12px;">
            <thead>
              <tr><th>Tag</th><th>{{ $t('ruleset.format') }}</th><th>{{ $t('type') }}</th><th>{{ $t('ruleset.interval') }}</th></tr>
            </thead>
            <tbody>
              <tr v-for="(rs, i) in parsed.rule_set" :key="i"
                :style="mode === 'merge' && existingRulesetTags.includes(rs.tag) ? 'opacity:0.4' : ''">
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
        <v-btn @click="dialog = false" variant="text">{{ $t('actions.close') }}</v-btn>
        <v-spacer />
        <v-btn @click="apply" color="primary" variant="flat" :disabled="!parsed">
          <v-icon icon="mdi-check" class="mr-1" />
          {{ mode === 'replace' ? $t('ruleset.importReplaceDo') : $t('ruleset.importMergeDo') }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts" setup>
import { ref, computed } from 'vue'

const props = defineProps<{
  existingRulesCount: number
  existingRulesetsCount: number
  existingRulesetTags: string[]
}>()

const emit = defineEmits<{
  'save': [block: any, mode: 'merge' | 'replace', applyFinal: boolean]
}>()

const dialog = ref(false)
const tab = ref('json')
const rawJson = ref('')
const fetchUrl = ref('')
const fetching = ref(false)
const error = ref('')
const parsed = ref<any>(null)
const mode = ref<'merge' | 'replace'>('merge')
const applyFinal = ref(false)

const hasConflicts = computed(() => props.existingRulesCount > 0 || props.existingRulesetsCount > 0)

const skippedRulesets = computed(() => {
  if (!parsed.value?.rule_set) return 0
  const existing = new Set(props.existingRulesetTags)
  return parsed.value.rule_set.filter((rs: any) => existing.has(rs.tag)).length
})

function extractRouteBlock(obj: any): any {
  if (obj?.route && (obj.route.rules || obj.route.rule_set)) return obj.route
  if (obj?.rules || obj?.rule_set) return obj
  return null
}

function setParsed(block: any) {
  parsed.value = block
  mode.value = hasConflicts.value ? 'merge' : 'replace'
  applyFinal.value = false
}

function open() {
  rawJson.value = ''; fetchUrl.value = ''; error.value = ''
  parsed.value = null; tab.value = 'json'; applyFinal.value = false
  mode.value = hasConflicts.value ? 'merge' : 'replace'
  dialog.value = true
}

function parseJson() {
  error.value = ''; parsed.value = null
  try {
    const block = extractRouteBlock(JSON.parse(rawJson.value))
    if (!block) { error.value = 'No "rules" or "rule_set" arrays found.'; return }
    setParsed(block)
  } catch (e: any) {
    error.value = 'JSON parse error: ' + e.message
  }
}

async function fetchFromUrl() {
  error.value = ''; parsed.value = null; fetching.value = true
  try {
    const resp = await fetch(fetchUrl.value)
    if (!resp.ok) throw new Error(`HTTP ${resp.status}`)
    const block = extractRouteBlock(await resp.json())
    if (!block) { error.value = 'No "rules" or "rule_set" found in fetched JSON.' }
    else setParsed(block)
  } catch (e: any) {
    error.value = 'Fetch error: ' + e.message
  } finally {
    fetching.value = false
  }
}

async function onFileUpload(event: Event) {
  error.value = ''; parsed.value = null
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return
  try {
    const block = extractRouteBlock(JSON.parse(await file.text()))
    if (!block) { error.value = 'No "rules" or "rule_set" found in the file.' }
    else setParsed(block)
  } catch (e: any) {
    error.value = 'JSON parse error: ' + e.message
  }
}

function apply() {
  if (!parsed.value) return
  emit('save', parsed.value, mode.value, applyFinal.value)
  dialog.value = false
}

defineExpose({ open })
</script>
