<template>
  <v-dialog v-model="dialog" max-width="620" scrollable>
    <template #activator="{ props }">
      <v-btn
        v-bind="props"
        icon="mdi-arrow-expand"
        size="small"
        variant="text"
        :title="$t('rule.bulkEdit')"
      ></v-btn>
    </template>
    <v-card>
      <v-card-title class="d-flex align-center pa-4">
        <span>{{ label }} — {{ $t('rule.bulkEdit') }}</span>
        <v-spacer />
        <v-chip size="small" color="primary" variant="tonal" class="mr-2">
          {{ lineCount }} {{ $t('rule.bulkItems') }}
        </v-chip>
      </v-card-title>
      <v-divider />
      <v-card-text class="pa-4">
        <p class="text-caption text-medium-emphasis mb-3">
          {{ $t('rule.bulkHint') }}
        </p>
        <v-textarea
          v-model="localText"
          :label="label"
          variant="outlined"
          rows="16"
          hide-details
          spellcheck="false"
          style="font-family: monospace; font-size: 13px;"
        ></v-textarea>
      </v-card-text>
      <v-divider />
      <v-card-actions class="pa-3">
        <v-btn @click="dialog = false" variant="text">{{ $t('actions.close') }}</v-btn>
        <v-spacer />
        <v-btn @click="clearAll" color="error" variant="text">{{ $t('rule.bulkClear') }}</v-btn>
        <v-btn @click="apply" color="primary" variant="flat">{{ $t('actions.apply') }}</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts">
export default {
  props: {
    modelValue: {
      type: String,
      default: '',
    },
    label: {
      type: String,
      default: '',
    },
  },
  emits: ['update:modelValue'],
  data() {
    return {
      dialog: false,
      localText: '',
    }
  },
  watch: {
    dialog(open: boolean) {
      if (open) {
        this.localText = this.modelValue
      }
    },
  },
  computed: {
    lineCount(): number {
      return this.localText
        .split('\n')
        .filter((l: string) => l.trim().length > 0)
        .length
    },
  },
  methods: {
    apply() {
      const unique = [
        ...new Set(
          this.localText
            .split('\n')
            .map((l: string) => l.trim())
            .filter((l: string) => l.length > 0)
        ),
      ]
      this.$emit('update:modelValue', unique.join('\n'))
      this.dialog = false
    },
    clearAll() {
      this.localText = ''
    },
  },
}
</script>
