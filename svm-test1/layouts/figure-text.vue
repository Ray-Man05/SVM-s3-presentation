<!-- ::title:: ::default:: (substantial text) ::figure:: (image | video | component) -->
<template>
  <div class="slidev-layout figure-layout">
    <header><slot name="title" /></header>
    <div class="text-content"><slot name="default" /></div>
    <div class="fig"><slot name="figure" /></div>
  </div>
</template>
<style scoped>
.figure-layout { display: flex; flex-direction: column; padding: 0.5rem 1.25rem; height: 100%; box-sizing: border-box; overflow: hidden; }
.figure-layout header { margin-bottom: 0.15rem; flex-shrink: 0; }
.text-content { margin-bottom: 0.35rem; font-size: 0.9rem; line-height: 1.35; flex-shrink: 0; }
.fig { flex: 1 1 auto; width: 100%; height: 100%; min-height: 0; display: flex; justify-content: center; align-items: center; }
.fig :deep(img), .fig :deep(video), .fig :deep(svg) { width: 100%; height: 100%; max-width: 100%; max-height: 85%; object-fit: contain; }
</style>