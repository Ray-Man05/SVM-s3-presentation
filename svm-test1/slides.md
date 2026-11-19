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
layout: text
---

::title::

# Roadmap

::default::

A support vector machine (SVM) is a powerful, versatile model for classification, regression, and outlier detection.

Our goal is to classify points into two classes using the best possible dividing boundary.

We build SVMs in a few steps:

1. A line that separates the classes, and determining the best such line (the margin)
2. What to do when the data can't be separated by a straight line (kernels)
3. How to do it without too many computations (the kernel trick)
4. How SVMs compare with other methods

---
layout: media-text
---

::title::

# The Separator

::default::

With two features, the separator is a line, in 3D it's a plane, and with more dimensions it's a **hyperplane**.

We determine the equation of the line or separator during training based on the datapoints.
It can be written as:

$$w_1 * x_1 + w_2 * x_2 ... + b = 0$$
Or: 
$$w^\top x + b = 0$$

Where:
- $x$ is a data point with its full list of feaure values
- $w$ is a list of weights that sets the line's direction
- $b$ is an offset that shifts the line

Points are classified based on which side of the line they fall.

::media::

<!-- **[FIGURE: two clean clusters, one simple line with no margins, its equation written on the graph next to it (a x_1 + b x_2 + c= 0)]** -->

<SimpleLine/>

---
layout: media-text
---

::title::

# Sensitivity to outliers

::default::



A margin is an area around the line that shouldn't touch any point, we try to <span class="text-green font-bold">maximize</span> it, the wider the margin the more distinct the separation between the two classes is:

$$
\textcolor{#16a34a}{\textbf{margin}} = \frac{2}{\|w\|}
$$

A **hard margin** does not allow for any misclassifications during training, this means that **outliers can skew the line** heavily to one side, which also collapses the margin.

When evaluated against test data, we see that a single outlier ruined our classifier's performance.

::media::

  <MarginDemo static margin="hard" caption="Hard margin" show-count />
  

<script setup>
import MarginDemo from './components/MarginDemo.vue'
</script>



---
layout: figure-text
---

::title::

# Soft margin

::default::

To fix this we can use a **soft margin**, it allows misclassifications during training, misclassified points inside the margin still influence the line during training, but less than misclassifications that are outside of it.



::figure::

<MarginDemo>

<template #step0="{ nTrain }">

{{ nTrain }} training points. 

An outlier (orange) lands inside the blue group.

</template>

<template #step1="{ mode, margin }">

<div v-if="mode === 'hard'">

A hard margin allows zero errors, which forces the line to tilt and collapses the margin to a very small value: <b>{{ margin }}</b>.

</div>
<div v-else>

A soft margin tolerates the outlier, it retains a robust boundary with a margin of <b>{{ margin }}</b>.

</div>

</template>

<template #step2="{ nTest }">

Overlaying {{ nTest }} unseen **test** points.

</template>

<template #default="{ mode, testErrors, nTest }">

<div v-if="mode === 'hard'">

**{{ testErrors }} of {{ nTest }}** test errors. 

One outlier ruined the rigid line.

</div>
<div v-else>

**{{ testErrors }} of {{ nTest }}** test errors. 

Tolerating one training mistake yielded a better model.

</div>

</template>

</MarginDemo>

<script setup>
import MarginDemo from './components/MarginDemo.vue'
</script>

---
layout: full
---

::title::

# Hard vs. soft margin

::default::

<div class="grid grid-cols-2 gap-4 h-full items-center">
  <MarginDemo static margin="hard" caption="Hard margin" show-count />
  <MarginDemo static margin="soft" caption="Soft margin" show-count />
</div>
<script setup>
import MarginDemo from './components/MarginDemo.vue'
</script>

---
layout: media-text
---

::title::

# Support vectors



::default::

**Support vectors** are the training points that touch the margin, sit inside it, or are misclassified, they are the datapoints that influence the line as it adjusts.

Correctly classified points outside the margin do **not** influence the boundary position.

::media::

<!-- **[FIGURE: highlight support vectors, point out a misclassification inside the margin, one outside the margin, and a correctly classified point that touches the margin; grey out the rest]** -->
<SupportVectors />

<script setup>
import SupportVectors from './components/SupportVectors.vue'
</script>



---
layout: figure
---

::title::

# Lifting data into higher dimensions

::default::

When the classes can't be separated by a simple line, we can map the datapoints onto a higher dimension where they become linearly separable.

We use a function called a feature map ($\phi(x)$) to compute new coordinates based on the original point's data.

::figure::

<KernelLift3D>

<template #default>

When data isn't linearly separable, the feature map $\phi(x)$ lifts points into a space where a flat plane works.

</template>

<template #ring>

**Circle:** Class A around ($r=3$) and class B around ($r=1$).

We use $\phi(x_1, x_2) = (x_1, x_2, x_1^2 + x_2^2)$ to separate them with a plane at $z = 5$.

</template>

<template #xor>

**XOR:** Same-sign vs opposite-sign quadrants. 

We use $\phi(x_1, x_2) = (x_1, x_2, x_1 * x_2)$ to separate the classes with a plane at $z = 0$.

</template>

</KernelLift3D>

<script setup>
import KernelLift3D from './components/KernelLift3D.vue'
</script>

---
layout: text
---

::title::

# Kernels

The previous examples were for illustration purposes, with real data that has many features, we can't clearly visualize the mapping we should use.

Instead we use a **kernel function**, it takes two data points and returns a sort of **similarity score** between them.

Points that are more similar to each other should logically belong to the same class, the hyperplane becomes a kind of *similarity threshold* between points of different classes.

Three Kernels are commonly used, each can be thought of as a different way of measuring **similarity**:

- <span class="text-blue font-bold">Linear</span>, $k(x,x') =$ <span class="text-blue font-bold">$x^\top x'$</span>: computes similarity using the <span class="text-blue font-bold">dot product</span>. Works well when the data has many features and few samples.
- <span class="text-orange font-bold">Radial Basis Function</span>, $k(x,x') =$ <span class="text-orange font-bold"> $e^{(-\gamma \|x - x'\|^2)}$ </span>: computes similarity using the <span class="text-orange font-bold">Euclidian distance</span> (1 when close, 0 when far apart). It assumes that nearby points tend to share the same class which is a relatively safe assumption, that makes it a good default to use.
- <span class="text-green font-bold">Polynomial</span>, $k(x,x') =$ <span class="text-green font-bold"> $(x^\top x' + c)^d$ </span>: it uses <span class="text-green font-bold">interaction features</span>, e.g. $x_1 * x_2$ evolves differently based on how $x_1$ and $x_2$ interact with each other, data points with features that evolve in a similar way are more similar to each other


---
layout: text
---

::title::

# The kernel trick

::default::

To compare between two given points $x$ and $z$, we can either use one of the Kernels we just saw, or map both to a higher dimension using $\phi()$ then compute their dot products.

Explicitly computing $\phi(x)$ is costly, whereas **kernel functions** compute dot products directly in the input space.

Example with polynomial kernel $K(x, z) = (x^\top z)^2$ for $x=(1,1,1)$ and $z=(0,1,1)$:

| Route | Computation | Result |
|---|---|---|
| **Explicit map + dot product** | $\phi(x)\cdot\phi(z) = 0 + 1 + 1 + 0 + 0 + 2$ | **4** |
| **Kernel shortcut** | $(x^\top z)^2 = (0 + 1 + 1)^2$ | **4** |

*With 1,000 features and degree 5, operations drop from $\sim 8\times 10^{12}$ to $\sim 1,000$.*

---
layout: figure
---

::title::

# The Kernel Trick

::default::

To compare between two given points $x$ and $z$, we can either use one of the Kernels we just saw, or map both to a higher dimension using and then compute their dot products.

Explicitly computing $\phi(x)$ is costly, whereas **kernel functions** compute dot products directly in the input space.
Here we show a comparison using the points $x=(1,1,1)$ and $z=(0,1,1)$ and:
- A polynomial kernel: $K(x, z) = (x^\top z)^2$
- A mapping $\phi(x) = \big(x_1^2,\ x_2^2,\ x_3^2,\ \sqrt2\,x_1x_2,\ \sqrt2\,x_1x_3,\ \sqrt2\,x_2x_3\big)$

::figure::

<KernelTrickTest>
</KernelTrickTest>

<script setup>
import KernelTrickTest from './components/KernelTrickTest.vue'
</script>

---
layout: text
---

::title::

# Kernel example: 1D dataset & similarity

::default::

Five points: P1($-2$), P2($-1$), P3($0$), P4($1$), P5($2$). Classes: B, A, A, A, B.

Using RBF kernel $k(x,z) = e^{-0.5(x-z)^2}$ ($\gamma = 0.5$):

| | P1 | P2 | P3 | P4 | P5 |
|---|---|---|---|---|---|
| **P1** | 1.00 | 0.61 | 0.14 | 0.01 | 0.00 |
| **P2** | 0.61 | 1.00 | 0.61 | 0.14 | 0.01 |
| **P3** | 0.14 | 0.61 | 1.00 | 0.61 | 0.14 |
| **P4** | 0.01 | 0.14 | 0.61 | 1.00 | 0.61 |
| **P5** | 0.00 | 0.01 | 0.14 | 0.61 | 1.00 |

*Neighbors have high similarity; distant points decay quickly.*

---
layout: text
---

::title::

# Kernel example: classification

::default::

Prediction rule: $f(x) = \sum_i \alpha_i y_i k(x_i, x) + b$

- Support vectors: P1, P2, P4, P5 ($\alpha \approx 2.22$)
- Inactive point: P3 ($\alpha = 0$)
- Offset: $b \approx -0.15$

**Classifying new point $Q = 1.8$:**
- Similarity to SVs gives weighted vote: $f(1.8) \approx -0.67$
- Negative sign $\implies$ **Class B**

---
layout: text
---

::title::

# Summary & SVM vs. KNN

::default::

**SVM:** Maximizes margin ($\frac{2}{\|w\|}$), uses soft margins ($C$), and applies kernel similarity. Only stores support vectors.

| Feature | KNN | Kernel SVM |
|---|---|---|
| **Voting** | $k$-nearest neighbors | Support vectors (weighted) |
| **Weights** | Equal | Learned via margin optimization |
| **Storage** | Entire dataset | Support vectors only |

*Remember to scale your features when using distance-based kernels like RBF.*