<!-- ::title:: ::default:: (prose, $$LaTeX$$, code) -->
<template>
  <div class="slidev-layout text-layout">
    <header><slot name="title" /></header>
    <div class="body"><slot name="default" /></div>
  </div>
</template>
<style scoped>
.text-layout { display: flex; flex-direction: column; padding: 0.5rem 1.25rem; height: 100%; box-sizing: border-box; overflow: hidden; }
.text-layout header { margin-bottom: 0.15rem; flex-shrink: 0; }
.body { flex: 1; min-height: 0; display: flex; flex-direction: column; justify-content: center; font-size: 0.9rem; line-height: 1.4; }
</style>