<!-- ::title:: ::media:: (image | video | component) ::default:: (text) -->
<template>
  <div class="slidev-layout media-text">
    <header><slot name="title" /></header>
    <section class="cols">
      <div class="text"><slot name="default" /></div>
      <div class="media"><slot name="media" /></div>
    </section>
  </div>
</template>
<style scoped>
.media-text { display: flex; flex-direction: column; padding: 0.5rem 1.25rem; height: 100%; box-sizing: border-box; overflow: hidden; }
.media-text header { margin-bottom: 0.15rem; flex-shrink: 0; }
.cols { flex: 1; min-height: 0; display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; align-items: center; grid-template-areas: 'text media'; }
.text { grid-area: text; }
.media { grid-area: media; display: flex; justify-content: center; align-items: center; width: 100%; height: 100%; }
.media :deep(img), .media :deep(video), .media :deep(svg) { max-width: 100%; max-height: 85%; object-fit: contain; }
</style>