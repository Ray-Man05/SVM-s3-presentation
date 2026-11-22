<!-- ::title:: ::default:: (large interactive component) -->
<template>
  <div class="slidev-layout full">
    <header><slot name="title" /></header>
    <div class="body"><slot name="default" /></div>
  </div>
</template>
<style scoped>
.full { display: flex; flex-direction: column; padding: 0.5rem 1.25rem; height: 100%; box-sizing: border-box; overflow: hidden; }
.full header { margin-bottom: 0.15rem; flex-shrink: 0; }
.body { flex: 1; min-height: 0; width: 100%; height: 100%; display: flex; flex-direction: column; }
</style>