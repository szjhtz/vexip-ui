<template>
  <Button type="primary" @click="Notice.open('提示的标题', '提示消息的具体内容')">
    打开提示
  </Button>
  <Button type="primary" @click="Notice.open('一条仅有标题的提示消息')">
    仅标题提示
  </Button>
  <Button type="primary" @click="$notice.open({ content: '一条仅有内容的提示消息' })">
    仅内容提示
  </Button>
</template>

<script setup lang="ts">
import { Notice } from 'vexip-ui'
</script>
