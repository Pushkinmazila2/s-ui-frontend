<template>
  <v-card>
    <v-row>
      <v-col cols="12" sm="6" md="4">
        <v-switch
          v-model="gitEnabled"
          color="primary"
          :label="$t('enable')"
          hide-details
        />
      </v-col>
    </v-row>
    
    <template v-if="gitEnabled">
      <v-row>
        <v-col cols="12" sm="6" md="4">
          <v-select
            v-model="gitConfig.provider"
            :items="providers"
            :label="$t('setting.gitProvider')"
            hide-details
          />
        </v-col>
        <v-col cols="12" sm="6" md="4">
          <v-text-field
            v-model="gitConfig.repoUrl"
            :label="$t('setting.gitRepoUrl')"
            hide-details
            placeholder="https://github.com/user/repo.git"
          />
        </v-col>
        <v-col cols="12" sm="6" md="4">
          <v-text-field
            v-model="gitConfig.branch"
            :label="$t('setting.gitBranch')"
            hide-details
            placeholder="main"
          />
        </v-col>
      </v-row>

      <v-row>
        <v-col cols="12" sm="6" md="4">
          <v-text-field
            v-model="gitConfig.token"
            :label="$t('setting.gitToken')"
            type="password"
            hide-details
          />
        </v-col>
        <v-col cols="12" sm="6" md="4">
          <v-switch
            v-model="gitConfig.autoSync"
            color="primary"
            :label="$t('setting.gitAutoSync')"
            hide-details
          />
        </v-col>
        <v-col cols="12" sm="6" md="4">
          <v-text-field
            v-model.number="gitConfig.syncInterval"
            type="number"
            min="1"
            :label="$t('setting.gitSyncInterval')"
            :suffix="$t('date.h')"
            hide-details
            :disabled="!gitConfig.autoSync"
          />
        </v-col>
      </v-row>

      <v-row>
        <v-col cols="12" sm="6" md="4">
          <v-switch
            v-model="gitConfig.syncConfig"
            color="primary"
            :label="$t('setting.gitSyncConfig')"
            hide-details
          />
        </v-col>
        <v-col cols="12" sm="6" md="4">
          <v-switch
            v-model="gitConfig.syncDb"
            color="primary"
            :label="$t('setting.gitSyncDb')"
            hide-details
          />
        </v-col>
      </v-row>

      <v-row v-if="gitConfig.lastSync">
        <v-col cols="12">
          <v-chip color="info" variant="outlined">
            {{ $t('setting.gitLastSync') }}: {{ formatDate(gitConfig.lastSync) }}
          </v-chip>
        </v-col>
      </v-row>

      <v-row>
        <v-col cols="auto">
          <v-btn
            color="primary"
            variant="outlined"
            @click="testConnection"
            :loading="testing"
          >
            {{ $t('setting.gitTestConnection') }}
          </v-btn>
        </v-col>
        <v-col cols="auto">
          <v-btn
            color="success"
            variant="outlined"
            @click="pushNow"
            :loading="pushing"
            :disabled="!gitEnabled"
          >
            {{ $t('setting.gitPushNow') }}
          </v-btn>
        </v-col>
        <v-col cols="auto">
          <v-btn
            color="info"
            variant="outlined"
            @click="pullNow"
            :loading="pulling"
            :disabled="!gitEnabled"
          >
            {{ $t('setting.gitPullNow') }}
          </v-btn>
        </v-col>
      </v-row>
    </template>
  </v-card>
</template>

<script lang="ts" setup>
import { ref, computed, watch } from 'vue'
import HttpUtils from '@/plugins/httputil'
import { push } from 'notivue'
import { i18n } from '@/locales'

const props = defineProps<{
  settings: any
}>()

const providers = [
  { title: 'GitHub', value: 'github' },
  { title: 'GitLab', value: 'gitlab' },
  { title: 'Gitea', value: 'gitea' }
]

const testing = ref(false)
const pushing = ref(false)
const pulling = ref(false)

const gitConfig = ref({
  enabled: false,
  provider: 'github',
  repoUrl: '',
  branch: 'main',
  token: '',
  autoSync: false,
  syncInterval: 24,
  syncConfig: true,
  syncDb: false,
  lastSync: ''
})

const gitEnabled = computed({
  get: () => gitConfig.value.enabled,
  set: (v: boolean) => { gitConfig.value.enabled = v }
})

const loadData = () => {
  if (props.settings?.gitSync?.length > 0) {
    try {
      gitConfig.value = JSON.parse(props.settings.gitSync)
    } catch (e) {
      gitConfig.value = {
        enabled: false,
        provider: 'github',
        repoUrl: '',
        branch: 'main',
        token: '',
        autoSync: false,
        syncInterval: 24,
        syncConfig: true,
        syncDb: false,
        lastSync: ''
      }
    }
  }
}

const testConnection = async () => {
  testing.value = true
  const msg = await HttpUtils.post('api/gitSyncTest', gitConfig.value)
  testing.value = false
  
  if (msg.success) {
    push.success({
      title: i18n.global.t('success'),
      message: i18n.global.t('setting.gitConnectionSuccess')
    })
  } else {
    push.error({
      title: i18n.global.t('failed'),
      message: i18n.global.t('setting.gitConnectionFailed')
    })
  }
}

const pushNow = async () => {
  pushing.value = true
  const msg = await HttpUtils.post('api/gitSyncPush', gitConfig.value)
  pushing.value = false
  
  if (msg.success) {
    gitConfig.value.lastSync = new Date().toISOString()
    push.success({
      title: i18n.global.t('success'),
      message: i18n.global.t('setting.gitSyncSuccess')
    })
  } else {
    push.error({
      title: i18n.global.t('failed'),
      message: i18n.global.t('setting.gitSyncFailed')
    })
  }
}

const pullNow = async () => {
  pulling.value = true
  const msg = await HttpUtils.post('api/gitSyncPull', gitConfig.value)
  pulling.value = false
  
  if (msg.success) {
    gitConfig.value.lastSync = new Date().toISOString()
    push.success({
      title: i18n.global.t('success'),
      message: i18n.global.t('setting.gitSyncSuccess')
    })
  } else {
    push.error({
      title: i18n.global.t('failed'),
      message: i18n.global.t('setting.gitSyncFailed')
    })
  }
}

const formatDate = (dateStr: string) => {
  if (!dateStr) return ''
  const date = new Date(dateStr)
  return date.toLocaleString()
}

watch(() => props.settings, () => {
  loadData()
}, { immediate: true })

watch(gitConfig, (newVal) => {
  props.settings.gitSync = JSON.stringify(newVal, null, 2)
}, { deep: true })
</script>
