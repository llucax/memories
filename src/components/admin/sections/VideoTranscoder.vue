<template>
  <div class="admin-section">
    <h3>{{ t('memories', 'Transcoder configuration') }}</h3>
    <p>
      {{
        t(
          'memories',
          'Memories uses the go-vod transcoder. You can run go-vod exernally (e.g. in a separate Docker container for hardware acceleration) or use the built-in transcoder. To use an external transcoder, enable the following option and follow the instructions in the documentation:'
        )
      }}
      <a target="_blank" href="https://github.com/pulsejet/memories/wiki/HW-Transcoding">
        {{ t('memories', 'External Link') }}
      </a>

      <template v-if="status">
        <NcNoteCard :type="binaryStatusType(status.govod)">
          {{ binaryStatus('go-vod', status.govod) }}
        </NcNoteCard>
      </template>

      <NcCheckboxRadioSwitch
        :disabled="!enableTranscoding"
        :checked.sync="config['memories.vod.external']"
        @update:checked="update('memories.vod.external')"
        type="switch"
      >
        {{ t('memories', 'Enable external transcoder (go-vod)') }}
      </NcCheckboxRadioSwitch>

      <NcTextField
        :disabled="!enableTranscoding"
        :label="t('memories', 'Binary path (local only)')"
        :label-visible="true"
        :value="config['memories.vod.path']"
        @change="update('memories.vod.path', $event.target.value)"
      />

      <NcTextField
        :disabled="!enableTranscoding"
        :label="t('memories', 'Bind address (local only)')"
        :label-visible="true"
        :value="config['memories.vod.bind']"
        @change="update('memories.vod.bind', $event.target.value)"
      />

      <NcTextField
        :disabled="!enableTranscoding"
        :label="t('memories', 'Connection address (same as bind if local)')"
        :label-visible="true"
        :value="config['memories.vod.connect']"
        @change="update('memories.vod.connect', $event.target.value)"
      />
    </p>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';

import AdminMixin from '../AdminMixin';

export default defineComponent({
  name: 'VideoTranscoder',
  mixins: [AdminMixin],
});
</script>
