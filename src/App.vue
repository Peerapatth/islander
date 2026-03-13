<template>
  <div class="min-h-screen bg-gray-50 text-gray-900">
    <Navbar />
    <main>
      <router-view />
    </main>
  </div>
</template>

<script setup>
import Navbar from '@/components/Navbar.vue'
</script>
