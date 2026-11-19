# Support Vector Machines: Slide Text Draft

> Figure placeholders are marked **[FIGURE: ...]**.

---

## Slide 0: Roadmap

A support vector machine (SVM) is a powerful and versatile machine learning model, capable of performing linear or nonlinear classification, regression and outlier detection.

**Goal:** classify points into two classes with the best possible dividing boundary.

We build SVMs in four steps:

1. A line that separates the classes, and determining the best such line (the margin)
2. What to do when the data can't be separated by a straight line
3. How to do it without too many computations (the kernel trick and kernels)
4. How SVMs compare with other methods

---

## Slide 1: The Separator

**One line to split two classes.**

With two features, a classifier draws a line. With three it draws a plane, and with more it draws a **hyperplane** (the same idea in higher dimensions).

The line is described by

$$w_1 x_1 + w_2 x_2 + \dots + b = 0 \quad\text{or}\quad w^\top x + b = 0$$

- $x$ is a data point with its full list of feature values
- $w$ is a list of weights that sets the line's direction
- $b$ is an offset that shifts the line

A point is classified by which side it falls on: $w^\top x + b > 0$ means class $+1$, and $< 0$ means class $-1$.

**[FIGURE: two clean clusters]**

---

## Slide 2: Outliers and the Soft Margin

But now what happens when there's an outlier from class A that's very close to class B? To keep every point on its correct side, the line gets pushed toward the wrong place. It fits the training data, but it's not a good fit in general.

**[FIGURE: two plots. Training data with the outlier, where the hard-edged line is not so useful; then test data overlaid, where we get a lot of errors]**

To account for this, we define a **margin** around the line and allow some points to violate it (sit inside it, or on the wrong side of the line) at a cost.

The algorithm tries to maximize the margin while penalizing violations.

$$\text{margin} = \frac{2}{\|w\|}$$


**[FIGURE: two plots. Line trained with the soft margin, and the same test data, now mostly correct]**

---

## Slide 3: Support Vectors

**Support vectors are the training points that touch the margin, sit inside it, or are misclassified. They are the only ones that influence the line as it adjusts.**

Correctly classified points outside the margin do not actually influence the position of the line.

**[FIGURE: two plots with support vectors in the exact same position and all the other points scattered completely randomly, and two lines that are the same, showing how only the support vectors, only the points closest to the ones of the opposite class ever influence the line]**

---

## Slide 4: When a Line Isn't Enough

If the classes can't be separated by a simple line, we can map the datapoints onto a higher-dimensional space where they become linearly separable, for instance from 2D to 3D.

We use a function $\phi(x)$ (a "feature map") that computes new coordinates from the datapoint's old ones.

(this is where we'll use Kernellift3d, some of the text below will go in the individual cards for each animation, keep it straighforward and simple)

### Example 1: Circle

Class A sits inside a circle and class B around it. No line separates them in $(x_1, x_2)$.

Add a third coordinate, the squared distance from the center:

$$\phi(x_1, x_2) = (x_1,\ x_2,\ x_1^2 + x_2^2)$$

| Point | $x_1^2+x_2^2$ | Class |
|---|---|---|
| inner ring (radius 1) | 1 | A |
| outer ring (radius 3) | 9 | B |

A flat plane at height $5$ now separates them.

**[FIGURE: 2D circle data, then 3D lifted version with the plane]**

### Example 2: XOR

| $x_1$ | $x_2$ | Class |
|---|---|---|
| $+1$ | $+1$ | A |
| $-1$ | $-1$ | A |
| $+1$ | $-1$ | B |
| $-1$ | $+1$ | B |

No line separates A from B.

Add the **interaction** feature $x_1 x_2$ (the product of the two features): it is $+1$ for A and $-1$ for B, so now they're separable.

**[FIGURE: XOR in 2D, then the same points plotted against $x_1x_2$]**

> Both examples worked because we could *see* the structure. With real data in many dimensions we can't, and computing the map for every point is expensive.

---

## Slide 5: The Kernel Trick

To make data linearly separable we lift it onto a higher dimension, but computing where every point goes ($\phi(x)$) is computationally expensive.

Instead of lifting the points onto a higher plane and then computing the dot product (similarity), we can use a **Kernel function** which directly computes similarity scores based on the original data in the original dimension.

Let's see how many computations we save with this example, using a degree-2 polynomial kernel and two points:

- Map: $\phi(x) = \big(x_1^2,\ x_2^2,\ x_3^2,\ \sqrt2\,x_1x_2,\ \sqrt2\,x_1x_3,\ \sqrt2\,x_2x_3\big)$
- Points: $x=(1,1,1)$ and $z=(0,1,1)$
- Kernel: [need its expression]



| Route | Computation | Result |
|---|---|---|
| **Map, then dot product** | $$\phi(x)=(1,1,1,\sqrt2,\sqrt2,\sqrt2)$$ $$\phi(z)=(0,1,1,0,0,\sqrt2)$$ $$\phi(x)\cdot\phi(z)=0+1+1+0+0+2$$ | **4** |
| **Kernel shortcut** | $(x \cdot z)^2 = (0+1+1)^2$ | **4** |

With 3 features we can already see how much simpler it is when using the Kernel, with 1000 features and a polynomial kernel of degree 5, the number of computations goes down from $8\times10^{12}$ to about 1000 multiplications.

---

## Slide 6: Kernels

A kernel function takes two points and returns a similarity score between them. 

Points that are more similar to each other should belong to the same class, and the line becomes a kind of 'similarity threshold' between points of different classes.

Three kernels are commonly used:

- **Linear**, $k(x,z) = x^\top z$: computes similarity using the dot product. Works well when the data has many features and few samples.
- **RBF**, $k(x,z) = \exp(-\gamma \|x - z\|^2)$: computes similarity using the Euclidean distance ($1$ when close, $0$ when far apart). It assumes that nearby points tend to share the same class, a relatively safe assumption, which makes it a good default.
- **Polynomial**, $k(x,z) = (x^\top z + c)^d$: uses interactions between features (products like $x_1 x_2$), so points are similar if they agree on those combinations.

One can also define **custom kernels** for a specific domain by deciding what similarity means, for instance in chemistry where similarity could be which substructures two molecules share.

---

## Slide 7: Practical Kernel Example

### The data

Five points, one feature $x$, two classes.

| Point | P1 | P2 | P3 | P4 | P5 |
|---|---|---|---|---|---|
| $x$ | $-2$ | $-1$ | $0$ | $1$ | $2$ |
| Class | B | A | A | A | B |

In 1D the separator is a single cutoff point, right now the two classes cannot be separated by a single point.

**[FIGURE: number line with the five points colored by class]**

### Similarity matrix (RBF, $\gamma = 0.5$)

Using the common RBF kernel with a gamma of 0.5,  gamma controls how strict or lax the similarity measure is, lower gammas means the similarity score decays fast as points are further away.

$k(x,z) = e^{-0.5\,(x-z)^2}$

(the cells should be colored in subtle shades of red and green according to the score)

| | P1 | P2 | P3 | P4 | P5 |
|---|---|---|---|---|---|
| **P1** | 1.0000 | 0.6065 | 0.1353 | 0.0111 | 0.0003 |
| **P2** | 0.6065 | 1.0000 | 0.6065 | 0.1353 | 0.0111 |
| **P3** | 0.1353 | 0.6065 | 1.0000 | 0.6065 | 0.1353 |
| **P4** | 0.0111 | 0.1353 | 0.6065 | 1.0000 | 0.6065 |
| **P5** | 0.0003 | 0.0111 | 0.1353 | 0.6065 | 1.0000 |

Example entry: P1 and P3 are 2 apart, so $k = e^{-0.5 \cdot 4} = 0.1353$. Neighbors are the most similar, and the score drops quickly with distance.

### How the matrix is used

The classification rule is

$$f(x) = \sum_i \alpha_i\, y_i\, k(x_i, x) + b$$

Training uses the whole table to find a weight $\alpha_i$ for every point ($0$ if it has no influence, positive for support vectors). 

$y_i$ is the label of training point $i$ ($+1$ for A, $-1$ for B).

Prediction is a **weighted vote of the support vectors, where each vote is scaled by similarity to the new point**.

Training the SVM (hard margin) on this table gives

$$\alpha \approx 2.22 \text{ for P1, P2, P4, P5},\qquad \alpha = 0 \text{ for P3},\qquad b \approx -0.15$$

P3 sits in the middle of class A and doesn't get a vote, the other four are the support vectors.

If we check using the predictor on one of the points like P2: 

$f(P2_x) = f(-1)  = 2.22\,(1 + 0.1353 - 0.6065 - 0.0111) - 0.15 \approx 1.00$, meaning P2 belongs to the class A, which is indeed correct.

### Classify a new point

New point $Q = 1.8$, which is closest to P5.

| Support vector | Squared distance to $Q$ | Similarity $k$ | Class |
|---|---|---|---|
| P1 | 14.44 | 0.0007 | B |
| P2 | 7.84 | 0.0198 | A |
| P4 | 0.64 | 0.7261 | A |
| P5 | 0.04 | 0.9802 | B |

$$f(Q) = 2.22\,\big(0.0198 + 0.7261 - 0.0007 - 0.9802\big) - 0.15 \approx -0.67$$

Negative means **class B**.
**[FIGURE: plot of $f(x)$ along the number line: positive in the middle, negative at the ends, crossing zero at about $x = \pm 1.5$, with the five points and $Q$ marked]**

---

## Slide 8: Support Vector Machines

**An SVM is a classifier that:**

1. finds the separating hyperplane with the **widest margin**,
2. tolerates some mistakes through a **soft margin** (controlled by $C$),
3. handles non-linear data via a **kernel** (a similarity function, computed with the **kernel trick**, never mapping the data explicitly),
4. makes predictions as a **similarity-weighted vote of its support vectors**:

$$f(x) = \sum_{i\,\in\,\text{SV}} \alpha_i\, y_i\, k(x_i, x) + b$$

**What you tune:** the kernel, the margin width $C$ (was C ever defined anyway, I'd rather have 2/w), and the kernel's own parameters (such as $\gamma$ for RBF), usually by cross-validation.

**Practical notes:** scale your features (RBF relies on distances); training gets slow on very large datasets.

---

## Slide 9: SVM vs KNN

### KNN in brief

To classify a new point, find its $k$ closest training points and take a **majority vote** of their labels. There is no training step; the model is the stored data.

**[FIGURE: new point with its k=3 neighbors circled]**

### Comparison

| | KNN | Kernel SVM |
|---|---|---|
| Who votes | the $k$ nearest points | the support vectors, weighted by similarity |
| Vote weights | equal | **learned** by maximizing the margin |
| What the model stores | all training data | only the support vectors |
| Main parameters | $k$ | $C$, kernel, $\gamma$ |

**Key difference:** KNN trusts every close neighbor equally, while the SVM learns which points matter and how much.

---

## Slide 10: Practical Comparison

(will work on this later I just need a placeholder)

**[FIGURE: bar chart of cross-validated accuracy, with error bars, for each model]**
