<template>
  <v-dialog v-model="dialog" max-width="700" scrollable>
    <v-card>
      <v-card-title class="d-flex align-center pa-4">
        <v-icon icon="mdi-download-multiple" class="mr-2" />
        {{ $t('ruleset.importTitle') }}
        <v-spacer />
        <v-chip v-if="importPreview.length > 0" size="small" color="primary" variant="tonal">
          {{ importPreview.length }} URLs
        </v-chip>
      </v-card-title>
      <v-divider />
      <v-card-text class="pa-4">
        <v-tabs v-model="importTab" class="mb-4">
          <v-tab value="text">
            <v-icon icon="mdi-text" class="mr-1" size="small" />
            {{ $t('ruleset.importPasteUrls') }}
          </v-tab>
          <v-tab value="file">
            <v-icon icon="mdi-file-upload" class="mr-1" size="small" />
            {{ $t('ruleset.importUploadTxt') }}
          </v-tab>
        </v-tabs>
        <v-window v-model="importTab">
          <v-window-item value="text">
            <p class="text-caption text-medium-emphasis mb-2">{{ $t('ruleset.importUrlsHint') }}</p>
            <v-textarea
              v-model="importRawText"
              label="URLs"
              variant="outlined"
              rows="10"
              auto-grow
              hide-details
              spellcheck="false"
              placeholder="https://github.com/.../geoip-telegram.srs&#10;https://github.com/.../geosite-youtube.srs"
              style="font-family: monospace; font-size: 13px;"
            ></v-textarea>
          </v-window-item>
          <v-window-item value="file">
            <p class="text-caption text-medium-emphasis mb-2">{{ $t('ruleset.importFileHint') }}</p>
            <v-file-input
              :label="$t('ruleset.importSelectTxt')"
              accept=".txt"
              variant="outlined"
              hide-details
              prepend-icon="mdi-file-document"
              @change="onFileUpload"
            ></v-file-input>
          </v-window-item>
        </v-window>
        <v-row class="mt-4">
          <v-col cols="12" sm="6" md="4">
            <v-select v-model="importFormat" :items="['binary', 'source']" :label="$t('ruleset.format')" variant="outlined" hide-details density="compact"></v-select>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-select v-model="importDetour" :items="[{ title: '— ' + $t('ruleset.importNone') + ' —', value: '' }, ...outboundTags]"
              :label="$t('ruleset.importDetour')" variant="outlined" hide-details density="compact" clearable></v-select>
          </v-col>
          <v-col cols="12" sm="6" md="4">
            <v-text-field v-model.number="importInterval" :label="$t('ruleset.importIntervalDays')"
              type="number" min="0" variant="outlined" hide-details density="compact"></v-text-field>
          </v-col>
        </v-row>
        <template v-if="importPreview.length > 0">
          <v-divider class="my-4" />
          <p class="text-caption text-medium-emphasis mb-2">
            {{ $t('ruleset.importPreview') }} — {{ importPreview.length }}
            ({{ importSkipped }} {{ $t('ruleset.importSkipped') }})
          </p>
          <v-table density="compact" style="font-size: 13px;">
            <thead>
              <tr><th>Tag</th><th>{{ $t('ruleset.format') }}</th><th>URL</th><th style="width:36px;"></th></tr>
            </thead>
            <tbody>
              <tr v-for="(item, i) in importPreview" :key="i" :style="item.exists ? 'opacity:0.4' : ''">
                <td>
                  <v-text-field v-model="item.tag" variant="plain" hide-details density="compact"
                    style="font-family: monospace; min-width: 140px;"></v-text-field>
                </td>
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
        <v-btn @click="dialog = false" variant="text">{{ $t('actions.close') }}</v-btn>
        <v-spacer />
        <v-btn @click="parseImport" variant="tonal" :disabled="importRawText.trim().length === 0">
          <v-icon icon="mdi-magnify" class="mr-1" />{{ $t('ruleset.importParse') }}
        </v-btn>
        <v-btn @click="applyImport" color="primary" variant="flat" :disabled="newCount === 0">
          <v-icon icon="mdi-download-multiple" class="mr-1" />
          {{ $t('actions.import') }}{{ newCount > 0 ? ' ' + newCount : '' }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts" setup>
import { ref, computed } from 'vue'

const props = defineProps<{
  outboundTags: string[]
  rulesetTags: string[]
}>()

const emit = defineEmits<{
  'save': [items: any[]]
}>()

const dialog = ref(false)
const importTab = ref('text')
const importRawText = ref('')
const importFormat = ref('binary')
const importDetour = ref('')
const importInterval = ref(1)

interface ImportItem { tag: string; url: string; format: string; exists: boolean }
const importPreview = ref<ImportItem[]>([])

const importSkipped = computed(() => importPreview.value.filter(i => i.exists).length)
const newCount = computed(() => importPreview.value.filter(i => !i.exists).length)

function urlToTag(url: string): string {
  try {
    const filename = new URL(url).pathname.split('/').pop() ?? ''
    return filename.replace(/\.[^.]+$/, '')
  } catch {
    const parts = url.split('/')
    return parts[parts.length - 1].replace(/\.[^.]+$/, '') || url
  }
}

function open() {
  importRawText.value = ''
  importPreview.value = []
  importTab.value = 'text'
  dialog.value = true
}

function parseImport() {
  const existingTags = new Set(props.rulesetTags)
  const seen = new Set<string>()
  importPreview.value = importRawText.value
    .split('\n').map(l => l.trim()).filter(l => l.length > 0 && l.startsWith('http'))
    .filter(url => { if (seen.has(url)) return false; seen.add(url); return true })
    .map(url => ({ tag: urlToTag(url), url, format: importFormat.value, exists: existingTags.has(urlToTag(url)) }))
}

function applyImport() {
  const toAdd = importPreview.value.filter(i => !i.exists).map(item => {
    const rs: any = { type: 'remote', tag: item.tag, format: item.format, url: item.url }
    if (importDetour.value) rs.download_detour = importDetour.value
    if (importInterval.value > 0) rs.update_interval = importInterval.value + 'd'
    return rs
  })
  emit('save', toAdd)
  dialog.value = false
}

async function onFileUpload(event: Event) {
  const file = (event.target as HTMLInputElement).files?.[0]
  if (!file) return
  importRawText.value = await file.text()
  parseImport()
}

defineExpose({ open })
</script>
