---
layout: ../../layouts/Markdown.astro
title: Pure & Neural Network Optimization Problems
date: 2026-07-27
---

# Pure & Neural Network Optimization Problems
##### 2026-07-27 

---

In a previous article post, I discussed neural network optimization spaces, and specifically gradient descent. I want to focus more on pure optimization and deep learning optimization.

# Pure Optimization Problems
In the most general sense, an optimization problem has a common form of:

$x$ Our unknown vector of variables, this can also be called the *parameters* of the problem 
$f$ Is the objective function of $x$. It is the function we want to maximize or minimize wrt $x$
$c_{i}$ Are the functions of constraints which are on $x$. These functions $c$ define certain inequalities or equations that our parameters, $x$ must obey or satisfy by

And so formally, an optimization problem is the task of minimizing or maximizing a function of parameters with constraints:

$$
min_{x} f(x) \quad  \text{subject to constraints} \quad c_{i}(x)   
$$


This general formula already looks quite similar to what neural networks use to learn. We have our parameters for our model, usually formalized using $\theta$ instead of $x$, since $x$ would represent the input data. 

And of course our objective function is quite similar, most of the time we may call it the loss function, but it serves the same purpose, we usually would want to *minimize* the loss function wrt our model parameters $\theta$.


# Neural Network Optimization Problems
Despite the similarities between pure optimization and model optimization, there is a large difference between the two, specifically the objective function we optimize. 

In most model optimization cases, our original loss function we truly want to optimize is intractable, there are many such cases of objective functions which are too difficult to optimize directly. A common which comes to mind is the RL expected return function. 

The RL return function is defined as the expectation over all model trajectories
$$
	J(\theta) = \mathbb{E_{\tau \sim \pi_{\theta}}}[R(\tau)]
$$
This objective function is impossible to compute exactly, with the number of all trajectories.
So instead, we commonly use a surrogate objective function which we optimize instead. 
This surrogate function is used to approximate the original function as closely as possible, and preserve any of its desirable properties.

Another smaller, yet still important distinction is convergence. Usually, pure optimization problems will halt at a local minimum, but model training consists of minimizing the surrogate loss, and instead halting due to specified convergence criterion. This criterion is chosen to avoid overfitting the dataset, and is based on the original loss function, such as the performance of the loss function on a validation dataset. 

## How do we find surrogate objectives?

It may sound quite easy to just say that certain loss functions are intractable, and we therefore have a convenient surrogate that we can optimize instead. But this process of first finding out that a function is intractable, and then devising a function that is not only tractable, but also closely relates to the original loss can be quite interesting.

### Intractability of Objective Functions 

Most of the time when researchers say that an objective function is intractable, it is commonly a strong suspicion. Rarely is it the case that we *know* an optimization problem is intractable, we usually have strong evidence that it is. 

A common approach is applying complexity theory. We can prove that optimizing an objective is intractable by showing that the problem is equivalent to attempting to solve a known hard problem. 

Essentially a version of pattern matching, we can assume certain aspects of the problem to be true, then show that the next steps would be near impossible. Example:

"Suppose there exists an efficient algorithm for solving our optimization problem. We can then show that this algorithm could be used to efficiently solve the *Travelling Salesman Problem* (TSP) as well. Since TSP is NP-Hard, then this implies that our algorithm can solve an NP-Hard problem, this is already not known to exist, therefore our optimization problem is also NP-Hard, and not possible"

Other common instances of proving intractability are methods such as simply inspecting the formula, if we see an impossibly large value to compute, then we know it would be quite difficult to optimize it therefore. 

Another common way is inspecting its convexity, if the function is non-convex (which is almost always the case in deep learning) then we know this has millions of parameters with thousands of local minima (Shown in a later section). At this point in time we do not know of an efficient algorithm which can find a global optimum in a non-convex plane with thousands of local minima. 



### Designing Surrogate Functions

When designing surrogate objectives, there is no clear algorithm or guide to follow, each function is different, and the constraints $c_{i}$ are different

There are common patterns that researchers may apply:

**1\. Relax the hard objective**  
In some cases, such as the 0-1 loss, the gradient of the function is unusable, essentially we say that $\nabla \mathcal{L}(\theta)$ provides no meaningful value, due to a number of reasons, the gradient may be too noisy, discontinuous, or just 0 nearly everywhere, providing nothing for us to learn off of. In the case of the 0-1 loss, though this function is quite easy to evaluate, its empirical objective is piecewise constant, has zero gradient nearly everywhere and is discontinuous at boundaries. All of this combined means that our ordinary gradient-based optimization would receive no useful local signal of how to improve the classifier. Essentially, **the iterates do not minimize in this landscape.**  


Then we can simply replace it with a smooth function which may penalize mistakes in a similar manner.
Options like cross-entropy are a good fit, since the function is differentiable while encouraging the same behaviour.

**2\. Local Approximation**  
This entails repeatedly building a surrogate that is only accurate near the current point

For example, if we use Newton's Method, defined as $f(x)$,  we could instead solve an easier approximation around the current iterate, say:
$$
	g(x \mid x_{k})
$$
where $x$ represents our iterates.

**3\. Empirical Loss Design**  
Not every surrogate loss is necessarily a unique task of proofs. Deep learning has examples of proposed losses which were created to help with optimization or performance. Functions such as the Focal Loss, which is the cross-entropy loss function but designed specifically for dealing with class imbalances. Focal loss was designed to give less weight, or score to easily classified examples, this forces the model to focus on more difficult samples. 




# Optimization Space

The optimization space is a common idea in both pure optimization and the subset of deep learning specific optimization. 
This space is known by a number names, most commonly in deep learning we can it is the parameter space or the loss landscape. And more formally it is the search space, or the feasible set.

For a generic optimization problem:
$$
	min_{x}f(x)
$$
$x$ is usually in the optimization space itself, this makes sense, our set of parameters must exist in the search space. 
We can formally then say $x \in \mathcal{X}$. Our optimizer then searches the space to find the point with the smallest value.


## Neural Network Parameter Space

Earlier, I mentioned that if the function is non-convex, then there exists multiple local minima. This proof is quite straightforward.  

If we assume a linear neural network:

$$
	f(x) = W_{2}W_{1}x
$$

then let:
$$
M = W_{2}W_{1}
$$

This means that for any invertible matrix $C$ of compatible dimensions:
$$
W_{2}W_{1}= (W_{2}C)(C^{-1}W_{1})
$$

And this is exactly true because:
$$
	(W_{2}C)(C^{-1}W_{1})=W_{2} (CC^{-1}) W_{1} = W_{2}W_{1}
$$

  
This then means that there is a infinite amount of unique and different parameter settings that all equal the same function:

Assuming $C$ can be any invertible matrix, there are infinitely many such matrices, so this gives us infinitely distinct parameter pairs of $(W_{2}W_{1})$ which all compute the same mapping.



And so it means that if we move along the manifold in the direction of one of these equal parameter settings: $(W_{2}W_{1})\to (W_{2}C)(C^{-1}W_{1})$ then the loss will effectively remain constant, implying zero first and second-order variation in tangent directions. Since the direction we're moving to is essentially the exact same as the one we're at right now.


The space:
$$
(W_{2}C, C^{-1}W_{1}) 
$$

forms a smooth manifold, which means that across this manifold, $f(x)$ does not change, and since the function does not change, the empirical loss cannot either

In practice, this point matters greatly, it means that:

1. This paramterization is now redundant: With infinitely many parameter vectors corresponding to the same function 
2. Non-convexity: Instead of a strict, isolated local minimizer, the optimization problem may possess a continuous manifold of equivalent minima  
3. This leads to optimization ambiguity, gradient descent may converge to any point on this manifold of equivalent minimizers
4. Because some directions have essentially no curvature, these flat directions can produce ill-conditioning, because the Hessian contains near-zero eigenvalues
5. Biased optimizers (Implicit Regularization): The choice of the optimizer may implicitly choose a singular direction/solution to traverse out of the infinite many, which could affect generalization. 


Another common effect of parameterization and high dimensionality is saddle points. The more dimensions we have, it becomes more likely that our search space will have directions that curve upward, while others curve downward. 
Symmetries in the search space also contribute to saddle points, with multiple parameter settings representing the same function, some directions will be completely flat (as we discussed earlier), while other directions may increase or decrease the loss. And because of these changes in curvature, the Hessian will ultimately capture this effect, being indefinite. 



## Pure Optimization Search Space 

Of course, the two spaces do not fundamentally differ, after all, a neural network optimization problem is a standard optimization problem. BUt the difference lies in what the search space represents and the geometry. 

In a pure optimization problem, the search space simply contains all of the candidate solutions, we are searching for one optimal point, which will be our solution. But in a neural network optimization problem, one point in our space represents one complete neural network. We are not mapping a point to a direct solution, but we map parameters to a function.
neural network 
And for the most part, this space is quite similar, but the fact that our parameter space is overparameterized leads to the core differences.


# Attention Mechanisms and Effects on the Landscape 

Attention mechanisms change the geometry of the objective, $\mathcal{L(\theta)}$. It does so because attention fundamentally adds more complex parameter interactions. 

Instead of our layers remaining relatively local:
$$
	x \to W_{1} \to W_{2} \to \dots
$$
Self attention adds complex relationships of weights:

$$
	Q = XW_{Q}, \quad K = XW_{K}, \quad V = XK_{V}
$$
$$
	\text{softmax}\left( \frac{QK^T}{\sqrt{ d }} \right) V
$$
Which means that the output now simultaneously depends on three weight matrices, $W_{Q}, W_{K}, W_{V}$. This in turn couples a lot of parameters together, which leads to a richer and more non linear optimization landscape. It leads to a non linear landscape because self attention is inherently non-linear:

Homogeneity defined as
$$
	f(\lambda x) = \lambda f(x)
$$

In the case of softmax:

$$
softmax(x) = \frac{e^{{x_{i}}}}{\sum_{j}e^{{x_{j}}}}
$$
$$
softmax(\lambda x) = \frac{e^{\lambda x_{i}}}{\sum_{j}e^{\lambda x_{j}}}
$$
$$
\lambda \cdot softmax(x) = \lambda \cdot \frac{e^{{x_{i}}}}{\sum_{j}e^{{x_{j}}}}
$$
And we can see
$$
	softmax(\lambda x) \neq \lambda \cdot softmax(x)
$$
Because linearity must satisfy both conditions of additivity and homogeneity, softmax is not linear. 
And because softmax is applied to the attention equation, then attention is also non linear. 


Though the addition of attention and non linearity may sound like additional complexity, it can actually help us deal with the non convexity of the parameter space. 
Non linearity is introduced to make the model expressive enough to learn complex functions. Consider that even if we had a purely linear network of $f(x) = W_{2}W_{1}x$, even though this computes a linear function, the optimization problem is already non-convex because of the parameters. 
Which is why we in fact add non-linear functions such as ReLU. 

Attention is therefore added for similar reasons, through its introduction, it increases the representational capacity of the network, because it allows dynamic input-dependent interactions between features. 