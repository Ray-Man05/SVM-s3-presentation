<!-- ::title:: ::default:: (one concise line) ::figure:: -->
<template>
  <div class="slidev-layout figure-layout">
    <header><slot name="title" /></header>
    <p class="lead"><slot name="default" /></p>
    <div class="fig"><slot name="figure" /></div>
  </div>
</template>
<style scoped>
.figure-layout { display: flex; flex-direction: column; }
.lead { text-align: center; font-size: 1.15rem; margin: 0 0 .8rem; }
.fig { flex: 1; min-height: 0; display: flex; justify-content: center; align-items: center; }
.fig :deep(img), .fig :deep(video) { max-width: 100%; max-height: 340px; }
</style>
