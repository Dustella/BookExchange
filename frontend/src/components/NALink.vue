<template>
  <RouterLink :to="to" #="{ navigate, href }">
    <NA :href="href" @click="navigate">
      <slot />
    </NA>
  </RouterLink>
</template>
<script setup lang="ts">
import { NA } from 'naive-ui';
import { RouterLink } from 'vue-router';

defineProps<{ to: string }>();
</script>
