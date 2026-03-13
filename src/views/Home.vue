<template>
  <section :style="{ maxHeight: 'calc(100vh - var(--header-height))' }" class="flex flex-col overflow-hidden bg-black">
    <HomeCarousel />
  </section>
</template>

<script setup>
import HomeCarousel from "@/components/HomeCarousel.vue";
import { useI18n } from "vue-i18n";

const { t } = useI18n();
</script>