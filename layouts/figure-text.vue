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
.text-content { margin-bottom: 0.35rem; font-size: 0.9rem; line-height: 1.25; flex-shrink: 0; }
.text-content :deep(p),
.text-content :deep(ul),
.text-content :deep(ol),
.text-content :deep(pre),
.text-content :deep(.katex-display) { margin-top: 0.25rem; margin-bottom: 0.25rem; }
.text-content :deep(> :first-child) { margin-top: 0; }
.text-content :deep(> :last-child) { margin-bottom: 0; }
.text-content :deep(li) { margin-top: 0.1rem; margin-bottom: 0.1rem; }
.text-content :deep(.katex-display) { line-height: 1.1; }
.fig { flex: 1 1 auto; width: 100%; height: 100%; min-height: 0; display: flex; justify-content: center; align-items: center; }
.fig :deep(img), .fig :deep(video), .fig :deep(svg) { width: 100%; height: 100%; max-width: 100%; max-height: 85%; object-fit: contain; }
</style>