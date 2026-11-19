# Support Vector Machines: Slide Text Draft

> Figure placeholders are marked **[FIGURE: ...]**. Speaker notes are in blockquotes.

---

## Slide 0: Roadmap or Table of content

A support vector machine (SVM) is a powerful and versatile machine learning model,
capable of performing linear or nonlinear classification, regression,

**Goal:** classify points into two classes with the best possible dividing boundary.

We build SVMs in four steps:

1. A line that separates the classes, and determining the best such line (the margin)
2. What to do when the data can't be separated by a straight line (kernels)
3. How to do it without too many computations (the kernel trick)
4. How SVMs compare with other methods

---

## Slide 1: The Separator

**One line to split two classes.**

With two features, a classifier draws a line. With three it draws a plane, and with more it draws a **hyperplane** (the same idea in higher dimensions).

The line is described by

w_1 * x_1 + w_2 * x_2 ... + b = 0

or

$$w^\top x + b = 0$$

- $x$ is a data point with its full list of feature values
- $w$ is a list of weights that sets the line's direction
- $b$ is an offset that shifts the line

A point is classified by which side it falls on: $w^\top x + b > 0$ means class $+1$, and $< 0$ means class $-1$.

**[FIGURE: two clean clusters]**

---

## Slide 2: Sensibility to outliers

But now what happens when there's an outlier from Class A that's very close to Class B? this will influence our line and push it very far, even though the line was trained on the training data, this is not a good fit

**[Two figures, one where we show how the training data with the outlier makes the line not so useful, and one where some overlayed test data shows how using a hard edge essentially fucked us over, we get a lot of errors]**

To account for this, we can define a margin around the line and allow for misclassifications within the margin, they still influence the line as it adjusts, but less so than points that touch it directly, which are called support vectors, or points that are misclassified and outside the margin, which influence it most.

The algorithm tries to maximize this margin while penalizing violations.

$$\text{margin} = \frac{2}{\|w\|}$$

**[Two figures, now the line trained with the soft margin is actually useful]**

---

## Slide 3: Outliers and the Soft Margin

?

Not sure how to organize this in regards to the previous slide, that one feels too heavy

---

## Slide 4: Support Vectors

**Support vectors are the training points that touch the margin, sit inside it, or are misclassified, they are the ones that influence the line as it adjusts.**

**[FIGURE: highlight support vectors; grey out the rest]**


---

## Slide 5: When a Line Isn't Enough

If the classes can't be separated by a simple line, we can try to map those datapoints onto a higher plane where they become linearly separable, for instance from 2D to 3D.

We use a function $\phi(x)$ (a "feature map") that computes new coordinates from the old ones based on the datapoint's coordinates.

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

Add the **interaction** feature $x_1 x_2$ (the product of the two features) to raise A and lower B, now they're separable:


**[FIGURE: XOR in 2D, then the same points plotted against $x_1x_2$]**

> Both examples worked because we could *see* the structure. With real data in many dimensions we can't, which motivates kernels.

---

## Slide 6: Kernels

The previous examples were for illustration purposes, with real data that has many features, we can't clearly visualize the mapping we should use .

Instead we use a kernel function, it takes two points and returns a sort of similarity score between them, points that are more similar to each other should belong to the same class, the line becomes a kind of 'similarity threshold' between points of different classes.

Three Kernels are commonly used:

- Linear, $k(x,x') = x^\top x'$: computes similarity using the dot product. Works well when the data has many features and few samples.
- RBF, $k(x,x') = \exp(-\gamma \|x - x'\|^2)$: computes similarity using the Euclidean distance (1 when close, 0 when far apart). It assumes that nearby points tend to share the same class which is a relatively safe assumption which makes it a good default to use.
- Polynomial, $k(x,x') = (x^\top x' + c)^d$: it uses interactions between features, e.g. x_1 * x_2 evolves differently based on how x_1 and x_2 interact with each other, data points with features that evolve similarly are more similar to each other
- Custom kernels: One can define a kernel for specific domains by deciding what similarity means, for instance in chemistry representing similarity by which substructures a molecule contains.

---

## Slide 7: Kernel Example

### The data

Four points, three binary features, two classes.

| Point | X | Y | Z | Class |
|---|---|---|---|---|
| P1 | 0 | 0 | 0 | A |
| P2 | 1 | 1 | 1 | A |
| P3 | 1 | 0 | 0 | B |
| P4 | 0 | 1 | 1 | B |

**[FIGURE: cube with the four points; A and B in different colors; draw the two diagonals P1-P2 and P3-P4]**

### No plane separates them

P1-P2 and P3-P4 are two diagonals of the cube. Both have the same midpoint, $(0.5, 0.5, 0.5)$, so they cross. Any plane that has P1 and P2 on one side also has their midpoint on that side, but that midpoint is also the midpoint of P3 and P4, so it can't have P3 and P4 on the other side.

(This does not sound right at all, I think I need a fifth point to really show that there's no hyperplane that can separate the data in 3D. The current computations seem to make sense though, with the pairs P1/P3 and P2/P4 being indeed the most similar to each other)

### Similarity matrix (RBF, $\gamma = 0.5$)

For binary points, $\|x-z\|^2$ is just the number of features where they differ.

| | P1 | P2 | P3 | P4 |
|---|---|---|---|---|
| **P1** | 1.0000 | 0.2231 | 0.6065 | 0.3679 |
| **P2** | 0.2231 | 1.0000 | 0.3679 | 0.6065 |
| **P3** | 0.6065 | 0.3679 | 1.0000 | 0.2231 |
| **P4** | 0.3679 | 0.6065 | 0.2231 | 1.0000 |

Example entry: P1 and P2 differ in 3 features, so $k = e^{-0.5 \cdot 3} = 0.2231$.

We can clearly see that 

### How the matrix is used

The classification rule is

$$f(x) = \sum_i \alpha_i\, y_i\, k(x_i, x) + b,$$

TO build the prediction hyperplane, we essentially iterate over every pair of points in the dataset (consider a point x, then iterate over every other point x_i) and compute similarity scores 'K(x_i, x)', the values are weighed using alpha_i based on the point's positions (support vector, inside the margin, outside) and $y_i = +1$ for class A and $-1$ for class B depending on x. 

Prediction is a **weighted vote of the training points, where each vote is scaled by similarity to the new point**.

Training the SVM on this table (hard margin) gives

$$\alpha_1=\alpha_2=\alpha_3=\alpha_4 \approx 4.02,\qquad b = 0.$$

All four points are support vectors. The weights are equal because the data is perfectly symmetric.

**Check:** for P1, $f = 4.02\,(1 + 0.2231 - 0.6065 - 0.3679) = 4.02 \times 0.2487 = 1.00$. So P1 lands exactly on the margin, and likewise every other point.

(Again with just 4 points this is utterly useless, this needs to change)

### Classify a new point

New point $Q = (0.9,\ 0.1,\ 0)$, which is closest to P3.

| Training point | Squared distance to $Q$ | Similarity $k$ | Class |
|---|---|---|---|
| P1 | 0.82 | 0.6637 | A |
| P2 | 1.82 | 0.4025 | A |
| P3 | 0.02 | 0.9900 | B |
| P4 | 2.62 | 0.2698 | B |

$$f(Q) = 4.02\,\big(0.6637 + 0.4025 - 0.9900 - 0.2698\big) = 4.02 \times (-0.1936) \approx -0.78$$

Negative means **class B**.

> The new point isn't on the original cube's corners, which is why we avoid ties. By symmetry, the cube corners not in the training set all score exactly $0$ here.

(whoever said anything about a cube, this is worse than useless it's confusing, this helps no one)

---

## Slide 8: The Kernel Trick

**We never actually moved the points anywhere.** Everything in the previous slide used only the similarity table.

As we saw before, to make data linearly separable, we can lift it onto a higher dimension, but computing where every point is going to go in the new dimension (phi(x)) before computing similarity scores is computationally expensive.

This is why we use the Kernel, computing similarity scores based on the original data in the original dimension is much more efficient.
A kernel computes those scores directly from the original features.

(Feels like this explanation or whole slide should come before the previous practical example, although the comparison between computation complexity is very useful)



Using the same dataset as the previous example and a degree-2 polynomial kernel, we can clearly see how the kernel trick saves us many computational steps:

$$\phi(x) = \big(x_1^2,\ x_2^2,\ x_3^2,\ \sqrt2\,x_1x_2,\ \sqrt2\,x_1x_3,\ \sqrt2\,x_2x_3\big)$$

Take P2 $=(1,1,1)$ and P4 $=(0,1,1)$:

| Route | Computation | Result |
|---|---|---|
| **Map, then dot product** | $\phi(P2)=(1,1,1,\sqrt2,\sqrt2,\sqrt2)$, $\phi(P4)=(0,1,1,0,0,\sqrt2)$; $\phi(P2) \cdot \phi(P4) $= 0+1+1+0+0+2 = 4$ | **4** |
| **Kernel shortcut** | $(P2\cdot P4)^2 = (0+1+1)^2$ | **4** |


---

## Slide 9: Support Vector Machines

**An SVM is a classifier that:**

1. finds the separating hyperplane with the **widest margin**,
2. tolerates some mistakes through a **soft margin** (controlled by $C$),
3. handles non-linear data via a **kernel** (a similarity function, computed with the **kernel trick**, never mapping the data explicitly),
4. makes predictions as a **similarity-weighted vote of its support vectors**:

$$f(x) = \sum_{i\,\in\,\text{SV}} \alpha_i\, y_i\, k(x_i, x) + b$$

**What you tune:** the kernel, $C$, and the kernel's own parameters (such as $\gamma$ for RBF), usually by cross-validation.

**Practical notes:** scale your features (RBF relies on distances); training gets slow on very large datasets.

---

## Slide 10: SVM vs KNN

### KNN in brief

To classify a new point, find its $k$ closest training points and take a **majority vote** of their labels. There is no training step; the model is the stored data.

**[FIGURE: new point with its k=3 neighbors circled]**

### Comparison

| | KNN | Kernel SVM |
|---|---|---|
| Who votes | the $k$ nearest points | all support vectors, weighted by similarity |
| Vote weights | equal | **learned** by maximizing the margin |
| What is stored | all training data | only the support vectors |
| Training | none | an optimization problem |
| Boundary | follows the local data, can be jagged | smooth; controlled by $C$ and $\gamma$ |
| Main knob | $k$ | $C$, kernel, $\gamma$ |

**Key difference:** KNN trusts every close neighbor equally, while the SVM learns which points matter and how much.

**Related:** an RBF-SVM with a very large $\gamma$ behaves almost like 1-nearest-neighbor.


---

## Slide 11: Practical Comparison

**Question:** how does SVM perform against other classifiers on the same data?

**[FIGURE: bar chart of cross-validated accuracy, with error bars, for each model]**

| Model | One-line description |
|---|---|
| **SVM (RBF)** | Widest-margin boundary on a similarity-based feature space |
| **Naive Bayes** | Uses class probabilities, assuming features are independent given the class |
| **Decision tree** | Repeated yes/no questions on individual features |
| **KNN** (optional) | Majority vote among nearest neighbors |

