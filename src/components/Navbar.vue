<template>
  <nav ref="nav" class="p-4 bg-white shadow flex items-center justify-between">
    <div>
      <router-link to="/" class="mr-4 font-medium">{{
        t("message.home")
      }}</router-link>
      <router-link to="/about" class="font-medium">{{
        t("message.about")
      }}</router-link>
    </div>
    <div>
      <button @click="toggleLang" class="px-3 py-1 bg-gray-100 rounded">
        {{ locale === "en" ? "TH" : "EN" }}
      </button>
    </div>
  </nav>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from "vue";
import { useI18n } from "vue-i18n";

const { t, locale } = useI18n();

function toggleLang() {
  locale.value = locale.value === "en" ? "th" : "en";
  try {
    localStorage.setItem("locale", locale.value);
  } catch (e) {}
}


const nav = ref(null);

function setHeaderHeight() {
  try {
    const h = nav.value ? nav.value.offsetHeight : 0;
    document.documentElement.style.setProperty("--header-height", `${h}px`);
  } catch (e) {}
}

onMounted(() => {
  nextTick(setHeaderHeight);
  window.addEventListener("resize", setHeaderHeight);
});

onUnmounted(() => {
  window.removeEventListener("resize", setHeaderHeight);
});
</script>

<style lang="scss" scoped></style>
