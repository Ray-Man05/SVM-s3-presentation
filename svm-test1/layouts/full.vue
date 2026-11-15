<!-- ::title:: ::default:: (large interactive component) -->
<template>
  <div class="slidev-layout full">
    <header><slot name="title" /></header>
    <div class="body"><slot name="default" /></div>
  </div>
</template>
<style scoped>
.full { display: flex; flex-direction: column; padding: 1rem 1.5rem; }
.body { flex: 1; min-height: 0; }
</style>
