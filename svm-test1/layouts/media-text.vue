<!-- ::title:: ::media:: (image | video | component) ::default:: (text) -->
<template>
  <div class="slidev-layout media-text">
    <header><slot name="title" /></header>
    <section class="cols">
      <div class="media"><slot name="media" /></div>
      <div class="text"><slot name="default" /></div>
    </section>
  </div>
</template>
<style scoped>
.media-text { display: flex; flex-direction: column; }
.cols { flex: 1; min-height: 0; display: grid; grid-template-columns: 1fr 1fr; gap: 2rem; align-items: center; grid-template-areas: 'media text'; }
.text { grid-area: text; }
.media { grid-area: media; display: flex; justify-content: center; align-items: center; }
.media :deep(img), .media :deep(video) { max-width: 100%; max-height: 360px; }
</style>
