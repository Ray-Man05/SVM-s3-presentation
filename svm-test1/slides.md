---
theme: default
title: Support Vector Machines
colorSchema: light
transition: slide-left
comark: true
duration: 35min
drawings:
  persist: false
---

# Support Vector Machines

Your Name · Your Institution

---
layout: media-text
---

::title::

# Which line is the best line?

::media::

<!-- Replace with your own figure; rounded corners are applied globally -->
<img src="https://cover.sli.dev" alt="Separating lines" />

::default::

Many lines separate the two classes, but they are not equally safe.

- A line close to the data is **fragile**
- The SVM picks the line **farthest** from both classes
- Only the closest points, the *support vectors*, decide it

---
layout: text
---

::title::

# The maximum-margin problem

::default::

For labels $y_i \in \{-1, +1\}$ the separating hyperplane is $\mathbf{w}^\top \mathbf{x} + b = 0$ and the margin has width $2/\lVert\mathbf{w}\rVert$.

$$
\min_{\mathbf{w},\,b,\,\xi}\ \tfrac12\lVert\mathbf{w}\rVert^2 + C\sum_i \xi_i
\quad\text{s.t.}\quad y_i(\mathbf{w}^\top\mathbf{x}_i + b)\ge 1-\xi_i,\ \ \xi_i \ge 0
$$

```python
from sklearn.svm import SVC
clf = SVC(kernel="linear", C=1.0).fit(X, y)
print(clf.support_vectors_)
```

---
layout: full
---

::title::

# Finding the widest margin, step by step

::default::

<SvmSteps clicks>

<template #step0="{ errors, nSV }">

A random line. **{{ errors }}** points are on the wrong side, and {{ nSV }} points touch or break the margin.

</template>

<template #default="{ step, steps, nSV, margin, final }">

Step {{ step }} of {{ steps }}: refit using the worst offenders.

{{ nSV }} points touch or break the margin, and the margin width $2/\lVert\mathbf{w}\rVert$ is {{ margin }}.

<span v-if="final">**Converged:** only the true support vectors remain.</span>

</template>

</SvmSteps>

<script setup>
import SvmSteps from './components/SvmSteps.vue'
</script>

---
layout: text
---

::title::

# The kernel trick

::default::

Replace dot products with a kernel and the SVM works in a richer feature space without ever computing it.

$$
K(\mathbf{x},\mathbf{x}') = \varphi(\mathbf{x})^\top\varphi(\mathbf{x}'),
\qquad
K_{\text{RBF}}(\mathbf{x},\mathbf{x}') = \exp\!\big(-\gamma\lVert\mathbf{x}-\mathbf{x}'\rVert^2\big)
$$

---
layout: full
---

::title::

# Lift the data, then separate it

::default::

<KernelLift3D>

<template #default>

In the original plane, **no straight line** separates the two classes.

A kernel lets us work in a space where one does.

</template>

<template #ring>

Feature map (RBF, one coordinate shown):

$$ z = e^{-2\,(x^2+y^2)} $$

Inner points keep a high $z$, outer points a low $z$, so the plane $z=c$ separates them.

</template>

<template #xor>

Feature map (product kernel):

$$ z = x\,y $$

Same-sign points go up, opposite-sign points go down, so the plane $z=0$ separates them.

</template>

</KernelLift3D>
<script setup>
import KernelLift3D from './components/KernelLift3D.vue'
</script>

---
layout: media-text
---

::title::

# Template: media + text (video)

::media::

<video :src="'/media/demo.mp4'" autoplay loop muted playsinline controls></video>

::default::

Put a video here, or swap it for any Vue component, e.g. `<KernelLift3D />`.

---
layout: figure
---

::title::

# Template: concise text + large figure

::default::

One short sentence that frames the figure below.

::figure::

<img src="https://cover.sli.dev" alt="Large figure" />