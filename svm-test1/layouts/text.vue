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
.body { flex: 1; min-height: 0; display: flex; flex-direction: column; justify-content: center; font-size: 0.9rem; line-height: 1.25; }
.body :deep(p),
.body :deep(ul),
.body :deep(ol),
.body :deep(pre),
.body :deep(.katex-display) { margin-top: 0.25rem; margin-bottom: 0.25rem; }
.body :deep(> :first-child) { margin-top: 0; }
.body :deep(> :last-child) { margin-bottom: 0; }
.body :deep(li) { margin-top: 0.1rem; margin-bottom: 0.1rem; }
.body :deep(.katex-display) { line-height: 1.1; }
</style>