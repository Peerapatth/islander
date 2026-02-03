<template>
  <div class="min-h-screen bg-gray-50 text-gray-900">
    <nav class="p-4 bg-white shadow">
      <a href="/" class="mr-4 font-medium">Home</a>
      <a href="/about" class="font-medium">About</a>
    </nav>
    <main class="p-6">
      <router-view />
    </main>
  </div>
</template>

<script>
export default {
  name: 'App'
}
</script>
