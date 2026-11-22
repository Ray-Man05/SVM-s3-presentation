<!-- ::title:: ::default:: (one concise line) ::figure:: -->
<template>
  <div class="slidev-layout figure-layout">
    <header><slot name="title" /></header>
    <p class="lead"><slot name="default" /></p>
    <div class="fig"><slot name="figure" /></div>
  </div>
</template>
<style scoped>
.figure-layout { display: flex; flex-direction: column; padding: 0.25rem 1.25rem; height: 100%; box-sizing: border-box; overflow: hidden; justify-content: space-between; }
.figure-layout header { margin: 0; padding: 0; flex-shrink: 0; line-height: 1.2; }
.lead { text-align: left; font-size: 0.95rem; margin: 0; flex-shrink: 0; line-height: 1.2; }
.fig { flex: 1; width: 100%; min-height: 0; display: flex; justify-content: center; align-items: center; }
.fig :deep(img), .fig :deep(video), .fig :deep(svg) { width: 100%; height: 100%; max-width: 100%; max-height: 100%; object-fit: contain; }
</style>