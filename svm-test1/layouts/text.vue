<!-- ::title:: ::default:: (prose, $$LaTeX$$, code) -->
<template>
  <div class="slidev-layout text-layout">
    <header><slot name="title" /></header>
    <div class="body"><slot name="default" /></div>
  </div>
</template>
<style scoped>
.text-layout { display: flex; flex-direction: column; }
.body { flex: 1; min-height: 0; display: flex; flex-direction: column; justify-content: center; font-size: 1.05rem; }
</style>
