<template>
  <div>
    <FtLoader v-if="isLoading" />
    <FtFlexBox v-else-if="videoList.length === 0">
      <p class="message">
        {{ t('Subscriptions.Empty Local Videos') }}
      </p>
    </FtFlexBox>
    <FtElementList
      v-else
      :data="videoList"
      :use-channels-hidden-preference="false"
    />
  </div>
</template>

<script setup>
import { onMounted, ref, shallowRef } from 'vue'
import { useI18n } from 'vue-i18n'

import FtElementList from './FtElementList/FtElementList.vue'
import FtFlexBox from './ft-flex-box/ft-flex-box.vue'
import FtLoader from './FtLoader/FtLoader.vue'

import store from '../store/index'

const { t } = useI18n()

const isLoading = ref(true)
const videoList = shallowRef([])

onMounted(async () => {
  await loadLocalVideos()
})

async function loadLocalVideos() {
  const localVideoPath = store.getters.getLocalVideoPath

  if (process.env.IS_ELECTRON && typeof localVideoPath === 'string' && localVideoPath.length > 0) {
    try {
      const localVideos = await window.ftElectron.getLocalVideos(localVideoPath)
      videoList.value = localVideos.map((video) => ({
        ...video,
        isLocal: true,
      }))
    } catch (error) {
      console.error(error)
      videoList.value = []
    }
  } else {
    videoList.value = []
  }

  isLoading.value = false
}
</script>

<style scoped>
.message {
  margin-inline: auto;
}
</style>
