
<script setup lang="ts">
import { useNav } from '@slidev/client'

withDefaults(defineProps<{ nav?: boolean }>(), { nav: true })

const dropFocus = (e: Event) => (e.currentTarget as HTMLElement).blur()
const { nextSlide, prevSlide } = useNav()
</script>

<template>
  <div class="wrap">
    <div class="cards-container">

      <!-- Explicit Mapping -->
      <div class="card explicit-card">
        <div class="card-header">
          <span class="badge explicit-badge">Explicit Mapping</span>
          <h3>
            <i>ϕ</i>(<span class="x-col"><i>x</i></span>)
            ·
            <i>ϕ</i>(<span class="z-col"><i>z</i></span>)
            in ℝ⁶
          </h3>
        </div>

        <div class="card-body">

          <div class="phi-vectors">
            <div class="phi-box">
              <span class="phi-title">
                <span class="x-col">ϕ(x)</span>
                = (1, 1, 1, √2, √2, √2)
              </span>
              <span class="phi-cost">9m</span>
            </div>

            <div class="phi-box">
              <span class="phi-title">
                <span class="z-col">ϕ(z)</span>
                = (0, 1, 1, 0, 0, √2)
              </span>
              <span class="phi-cost">9m</span>
            </div>
          </div>

          <div class="dot-breakdown">
            <div class="dot-heading">
              <span class="section-label">6D Dot Product</span>
              <span class="phi-cost">6m + 5a</span>
            </div>

            <div class="dot-eq">
              <span class="x-col">ϕ(x)</span><sup>⊤</sup>
              <span class="z-col">ϕ(z)</span>
              =
              <span class="dot-terms">
                <span>1(0)</span> +
                <span>1(1)</span> +
                <span>1(1)</span> +
                <span>√2(0)</span> +
                <span>√2(0)</span> +
                <span>√2(√2)</span>
              </span>
            </div>

            <div class="dot-result">
              = 0 + 1 + 1 + 0 + 0 + 2 = <strong>4</strong>
            </div>
          </div>

          <div class="summary-row kernel-summary">
            <span>Result: <strong>4</strong></span>
            <span class="cost-tag">
              Cost: <strong>29 ops</strong>
              <small>(24m, 5a)</small>
            </span>
          </div>
        </div>
      </div>

      <!-- Kernel Trick -->
      <div class="card kernel-card">
        <div class="card-header">
          <span class="badge kernel-badge">Kernel Trick</span>
          <h3><i>K</i>(<i>x</i>, <i>z</i>) = (<i>x</i><sup>⊤</sup><i>z</i>)² in the original ℝ³</h3>
        </div>

        <div class="card-body">

          <div class="k-rows">
            <div class="k-row">
              <div class="k-step">
                <span class="tag">01 · Dot Product</span>
                <span class="phi-cost">3m + 2a</span>
              </div>

              <div class="k-equation">
                <span class="x-col">x</span><sup>⊤</sup>
                <span class="z-col">z</span>
                = (1·0) + (1·1) + (1·1) = <strong>2</strong>
              </div>
            </div>

            <div class="k-row result">
              <div class="k-step">
                <span class="tag">02 · Square</span>
                <span class="phi-cost">1m</span>
              </div>

              <div class="k-equation">2² = <strong>4</strong></div>
            </div>
          </div>


          <div class="summary-row ">
            <span>Result: <strong>4</strong></span>
            <span class="cost-tag">
              Cost: <strong>6 ops</strong>
              <small>(4m, 2a)</small>
            </span>
          </div>
        </div>
      </div>

    </div>

    <div v-if="nav" class="nav-bar">
      <div class="spacer"></div>
      <div class="slide-nav">
        <button
          title="Previous slide"
          aria-label="Previous slide"
          @click="prevSlide(); dropFocus($event)"
        >
          <svg viewBox="0 0 24 24">
            <path d="M15 5l-7 7 7 7" />
          </svg>
        </button>

        <button
          title="Next slide"
          aria-label="Next slide"
          @click="nextSlide(); dropFocus($event)"
        >
          <svg viewBox="0 0 24 24">
            <path d="M9 5l7 7-7 7" />
          </svg>
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.wrap {
  --ax: var(--c-axis, var(--c-light, #f8fafc));

  display: flex;
  flex-direction: column;
  gap: .3rem;
  height: 100%;
  justify-content: center;
}

.cards-container {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: .7rem;
  align-items: stretch;
}

.card {
  min-width: 0;
  background: var(--c-light);
  color: var(--c-dark);
  border-radius: .6rem;
  padding: .65rem .75rem;
  display: flex;
  flex-direction: column;
  box-shadow: 0 2px 8px rgba(0, 0, 0, .06);
  border: 1px solid rgba(0, 0, 0, .06);
}

.card-header {
  display: flex;
  flex-direction: column;
  gap: .2rem;
  margin-bottom: .45rem;
}

.badge {
  align-self: flex-start;
  font-size: .55rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: .06em;
  padding: .12rem .4rem;
  border-radius: 999px;
}

.kernel-badge {
  background: color-mix(in srgb, var(--c-green, #22c55e) 22%, transparent);
  color: #15803d;
}

.explicit-badge {
  background: color-mix(in srgb, var(--c-orange, #fb923c) 22%, transparent);
  color: #c2410c;
}

.card-header h3 {
  margin: 0;
  font-size: .88rem;
  font-weight: 700;
  line-height: 1.3;
}

.card-body {
  display: flex;
  flex-direction: column;
  flex: 1;
  gap: .4rem;
}

/* Explicit mapping */

.phi-vectors {
  display: flex;
  flex-direction: column;
  gap: .25rem;
}

.phi-box {
  background: rgba(0, 0, 0, .045);
  padding: .3rem .4rem;
  border-radius: .3rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: .25rem;
  min-width: 0;
}

.phi-title {
  font-weight: 600;
  font-size: .64rem;
  white-space: nowrap;
}

.phi-cost {
  flex-shrink: 0;
  font-size: .61rem;
  font-weight: 700;
  padding: .08rem .32rem;
  border-radius: .22rem;
  background: color-mix(in srgb, var(--c-green, #22c55e) 17%, transparent);
  color: #15803d;
  white-space: nowrap;
}

/* Dot product */

.dot-breakdown {
  background: color-mix(in srgb, var(--c-orange, #fb923c) 12%, rgba(0, 0, 0, .025));
  padding: .4rem .45rem;
  border-radius: .35rem;
  display: flex;
  flex-direction: column;
  gap: .2rem;
}

.dot-heading,
.k-step {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: .3rem;
}

.section-label,
.tag {
  font-size: .56rem;
  text-transform: uppercase;
  letter-spacing: .05em;
  font-weight: 700;
  opacity: .65;
}

.dot-eq {
  font-size: .67rem;
  line-height: 1.5;
  font-variant-numeric: tabular-nums;
}

.dot-terms {
  display: inline;
  white-space: nowrap;
}

.dot-result {
  font-size: .67rem;
  font-weight: 500;
  text-align: left;
  border-top: 1px solid rgba(0, 0, 0, .1);
  padding-top: .2rem;
}

.dot-result strong {
  font-size: inherit;
}

.dot-breakdown .phi-cost {
  background: color-mix(in srgb, var(--c-green, #22c55e) 17%, transparent);
}

/* Kernel calculation */

.k-rows {
  display: flex;
  flex-direction: column;
  gap: .3rem;
}

.k-row {
  display: flex;
  flex-direction: column;
  gap: .3rem;
  padding: .45rem .5rem;
  border-radius: .35rem;
  background: rgba(0, 0, 0, .045);
  font-size: .72rem;
  font-variant-numeric: tabular-nums;
}

.k-row.result {
  background: color-mix(in srgb, var(--c-green, #22c55e) 15%, transparent);
}

.k-equation {
  line-height: 1.4;
}

.k-equation strong {
  font-size: inherit;
}

.kernel-explanation {
  font-size: .66rem;
  line-height: 1.4;
  opacity: .7;
  padding: .35rem .45rem;
  border-left: 2px solid var(--c-orange, #fb923c);
  background: rgba(0, 0, 0, .025);
  border-radius: 0 .25rem .25rem 0;
}

/* Results */

.summary-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: .3rem;
  margin-top: auto;
  padding: .4rem .5rem;
  border-radius: .35rem;
  background: color-mix(in srgb, var(--c-green, #22c55e) 15%, transparent);
  font-size: .69rem;
  font-variant-numeric: tabular-nums;
}

.summary-row > span:first-child {
  font-weight: 500;
  white-space: nowrap;
}

.summary-row > span:first-child strong {
  font-size: inherit;
}

.kernel-summary {
  background: color-mix(in srgb, var(--c-orange, #fb923c) 15%, transparent);
}

.cost-tag {
  display: flex;
  align-items: baseline;
  justify-content: flex-end;
  flex-wrap: wrap;
  gap: .2rem;
  font-size: .66rem;
  font-weight: 600;
  text-align: right;
}

.cost-tag strong {
  font-size: inherit;
}

.cost-tag small {
  opacity: .65;
  font-size: .59rem;
  font-weight: 400;
}

.x-col {
  color: var(--c-blue, #0284c7);
  font-weight: 600;
}

.z-col {
  color: var(--c-orange, #ea580c);
  font-weight: 600;
}

/* Navigation */

.nav-bar {
  display: flex;
  align-items: center;
}

.spacer {
  flex: 1;
}

.slide-nav {
  display: flex;
  opacity: .45;
  transition: opacity .2s;
}

.slide-nav:hover {
  opacity: 1;
}

.slide-nav button {
  background: none;
  border: 0;
  padding: 0;
  color: var(--ax);
  cursor: pointer;
  display: grid;
  place-items: center;
  width: 1.4rem;
  height: 1.4rem;
}

.slide-nav button:focus-visible {
  outline: 2px solid var(--ax);
  outline-offset: 2px;
  border-radius: .2rem;
}

.slide-nav svg {
  width: .95rem;
  height: .95rem;
  fill: none;
  stroke: currentColor;
  stroke-width: 2.2;
  stroke-linecap: round;
  stroke-linejoin: round;
}
</style>
