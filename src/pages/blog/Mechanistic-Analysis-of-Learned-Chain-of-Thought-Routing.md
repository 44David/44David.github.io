---
layout: ../../layouts/Markdown.astro
title: Mechanistic Analysis of Learned Chain-of-Thought Routing
date: 2026-08-04
---

# Mechanistic Analysis of Learned Chain-of-Thought Routing
##### 2026-08-04 

---

In a previous [article](https://davids.onl/blog/inducing-reasoning-in-small-language-models/), I created qwen-0.5b-reasoning, a lightweight math reasoning model. Through the use of SCoTD, the model was able to perform logical reasoning tasks to a certain extent of accuracy. Through the explicit use of learned ```<think>``` tags, the model learned to use these tags to split up its reasoning and its user response. 

After testing and using the model, what really got me interested was the fact that the model learned to use ```<think>``` tags in certain scenarios, and not all the time. Prompts such as *"What is the capital of France?"* did not induce think tags instead opting for a simple response of "Paris" or similar. 

This was a really interesting phenomenon, and I was really curious what was going on inside. I was specifically curious of how it "knew" the difference, when to use think tags or not. 

There may be some simple explanations, such as:

1\. The model has learned a response mode routing mechanism
	Given input prompt $x$, the representation of $x$ may differ, and we could say a sort of "mechanism" routes to a direct answer or CoT based on the representation.

2\. Model has learned that certain prompt features increase the next token probability of think tags
	Perhaps, given the ending of the prompt, or the overall vocabulary structure, the next token to predict would be the think tag. 


Before I started any causal experiments, I decided to analyze the model behavior on data alone. 

# Model Behavior Analysis 

I first constructed a small dataset of 200 samples, halved with 100 samples from both of the training sources of the model. As I mentioned in my previous article, version 2 was trained on both my custom CoT dataset, and alpaca, which is an assistant based dataset. 

What I was specifically looking for here is the stability of the ```<think>``` tag, how often does it appear? I wanted to be sure that inducing reasoning was in fact reproducible. 

The results show that thinking occurs for every single reasoning type question, and rarely for an assistant based question.

| Source | Samples | Think Rate | Mean Think Probability | Median Think Probability |
|:------|--------:|-----------:|-----------------------:|-------------------------:|
| Alpaca | 100 | 0.01 | 0.010113 | 2.097275e-10 |
| GSM8K | 100 | 1.00 | 1.000000 | 1.000000e+00 |

Though there were a few alpaca samples that did induce reasoning, hence the 0.01 think rate:
```
id: alpaca_4163 

prompt: Provide a solution to this problem.\n\nA cyclist is biking from Point A to Point B. Along the way, he passes point C, which is one-third of the distance from A to B. At what point is the cyclist halfway to his destination?                   

think_token_probability: 9.998459e-01        

used_think: True
```

```
id: alpaca_38311            

prompt: Construct a system of equations to solve the following problem.\n\nA store sells sandwiches and drinks. There are 6 sandwiches and 5 drinks. Each sandwich costs $3 and each drink costs $2. The total cost is $21.                   

think_token_probability: 1.143912e-02       

used_think: False
```

What I find interesting is that I would expect the second sample, ```alpace_38311``` to have a similar think rate, but it's in fact much lower, there is not much to deduce or interpret at this point in time, but from these tests alone, it seems that the language itself does not affect think tag usage as much as I originally thought.


The gsm8k samples had a very small amount of questions ~0.99 think probability, though I would account this to stochasticity. 

Although these results are already interesting, the samples used were of course from the training data, and so it makes complete sense that the questions it was trained on, will have the result that is expected of it. 


### Modified dataset samples


Because the samples used in the first test were from the training data, I decided to rewrite 20 samples by hand, split evenly between alpaca and the gsm8k dataset. 

What I modified here was the language, I took all of the gsm8k questions and rewrote them into a more assistant based, alpaca style language, and did the reverse for the alpaca questions:

```
Sample base gsm8k:
"prompt": "Theresa has 5 more than thrice as many video games as Julia. Julia has a third as many video games as Tory. If Tory has 6 video games, how many video games does Theresa have?", 


Sample gsm8k rewritten:
"prompt":"Given T = 6, J = T / 3, and H = 3J + 5, calculate H.", "reference_answer":"11"
```

After running inference on these 20 rewritten samples, the results stay nearly identical:
| Source | Samples | Think Rate | Mean Think Probability | Median Think Probability |
|:-------|--------:|-----------:|-----------------------:|-------------------------:|
| Alpaca rewrite | 10 | 0.0 | 0.000006 | 1.165493e-09 |
| GSM8K rewrite | 10 | 1.0 | 0.994620 | 9.999943e-01 |

Although this is a 20 sample dataset, the results are quite strong, this shows that the model has perhaps learned something deeper than surface level prompt structure or vocabulary. There may be something like a semantic math task detector. It could be a possibility, all of the math questions, even when stripped of the original word problem format, induced thing tags. The internal split seems tied more to mathematical computation than a prompt style. 


### Factorial Math Dataset

To further explore the idea of a semantic math detector, I constructed a factorial dataset consisting of variations of math, in total, there were 20 samples, with 4 samples in each category, below are 5 sample questions used from each category:

```
{"id":"trivial_math_01","source":"trivial_math","prompt":"What is 2 + 2?"}

{"id":"conceptual_math_03","source":"conceptual_math","prompt":"Why is division by zero undefined?"}

{"id":"numbers_no_math_02","source":"numbers_no_math","prompt":"Write a short story containing the numbers 12, 47, and 93."}

{"id":"logic_no_numbers_04","source":"logic_no_numbers","prompt":"A person must choose either the red door or the blue door, but not both. They did not choose the red door. Which door did they choose?"}

{"id":"multihop_fact_04","source":"multihop_fact","prompt":"Water freezes at zero degrees Celsius. A container of water is cooled below that temperature. What state will the water most likely enter?"}
```

These questions were split up to include logical reasoning, math questions, and questions with numbers, but no logic/problem. This last one was important, I thought that perhaps the model learned that the inclusion of various numbers would increase the likelihood of think tags.

| Source | Samples | Think Rate | Mean Think Probability | Median Think Probability |
|:-------|--------:|-----------:|-----------------------:|-------------------------:|
| Conceptual math | 4 | 0.0 | 5.126003e-10 | 3.588663e-11 |
| Logic, no numbers | 4 | 0.5 | 3.733550e-01 | 2.467131e-01 |
| Multihop fact | 4 | 0.0 | 7.406468e-06 | 2.739947e-06 |
| Numbers, no math | 4 | 0.0 | 1.357409e-06 | 8.654757e-07 |
| Trivial math | 4 | 0.0 | 2.757217e-07 | 2.262306e-09 |

This is quite interesting, barely any problems triggered think tags, with only 2 out of the 4 logical problems inducing them.
This points to the fact that the router, or mechanism isn't simply detecting math or numbers in general, but is more sensitive to certain kind of symbolic constraint reasoning. 


# Model Activations

Throughout all of the three evaluations, I saved the model activations, this category utilizes the activations for a variety of experiments.

## Final Prompt Token PCA

To examine how the model's response policy emerges across network depth, PCA was applied to the final prompt token hidden state at layer 0, 12, and 24. I believe these 3 layer choices can show a summarized diverse set of layer evolution. Each point represents one prompt. The same coordinates are shown twice, the figure on the left shows the points colored by prompt category, right side shows whether the model actually generated think tags.

<!--![[layer_evolution.gif|1000]]-->

<img
  src="/images/layer_evolution.gif"
  alt="Layer evolution"
  style="display: block; width: 600px; max-width: 100%; margin: 0 auto;"
/>

*Nice visualization to see the evolution of the clusters, you can see how the categories become more opinionated wrt layer depth*

| ![Layer 12 by category](/images/layer_12_by_category.png) | ![Layer 12 by think usage](/images/layer_12_by_used_think.png) |
| :------------------------------------------------------: | :-----------------------------------------------------------: |
| ![Layer 24 by category](/images/layer_24_by_category.png) | ![Layer 24 by think usage](/images/layer_24_by_used_think.png) |

The representations were effectively indistinguishable at layer 0, which is why the plots have been excluded. But the results in short showed that all prompts shared the same final input token from the chat template. So the embedding layer activation at that position contains almost no specific prompt information. Which makes sense, this is layer 0.

But by layer 12, we can already see that a strong response mode geometry has emerged. Prompts that later generate think tags heave a mean PC1 coordinate of 5.16, which is compared to -4.59 for direct answer prompts. And PC1 alone separates the two modes with an AUROC of ~0.99. gsm8k prompts form a very distinct cluster, with alpaca prompts mostly occupying the negative PC1 space in a broader region.

The category structure figure also shows that this separation is not limited to the original training wording. Using the custom rewritten gsm8k prompts still remain on the positive space of PC1. The two relational logic question we talked about earlier, which induced think tags are also on the positive side. With the rest of the categories remaining on the direct answer side. This shows us that the intermediate layer representations align more closely with the model's eventual response mode.


And layer 24 shows an even more amplified version of the prior distinction. The mean PC1 is for think prompts is 95.57, and -85.04 for direct answer prompts. gsm8k prompts again form a tight cluster at the extreme positive space. 

The rewritten and diagnostic prompts follow the behavior than the source wording, gsm8k rewrites still sit on the positive side, with alpaca rewrites still on the negative side. This is quite interesting, it shows that by layer 12, the results have nearly become solidified, deeper layers only amplify the regions. 

Some notes to keep in mind: Yes, the model was trained in majority on my custom gsm8k dataset, so the tight clusters will mirror this, but what really stands out is the alignment of the rewritten gsm8k prompts, it shows that the geometry is not reducible to exact prompt memorization or the surface form of the prompt alone. 

## Linear Probe

In order to quantify the accessibility of the routing signal beyond the PCA projection. To do this, I trained a linear probe, the task for the linear probe was for each layer, to predict whether the model will emit think tags. Performance was near 0.50 accuracy for layer 0, which makes sense, the model has just started inference, and so layer 0 would be unpredictable. But the accuracy increased to ~0.90 after the first layer, and remained high throughout the network. AUROC was near 1.0 across all layers as well. These results show that the two response modes were almost perfectly ranked bu the probe score. This tells us that the routing distinction is linearly decodable from the final-token representation shortly after contextual processing begins and then becomes fully separable in deeper layers. 

## Routing Direction/Steering Vector Analysis

To identify a candidate activation direction associated with the models response policy, we take the mean-difference vector computed independently at each layer:
$$
	v_{l} = \mu^{think}_{l} - \mu^{direct}_{l}
$$
Each sample was projected onto $v_{l}$, and AUROC was used to measure how well the resulting scalar projection separated the two response modes. The strongest separation occurred at layer 23, where the projection achieved 1.0 AUROC. With the direction obtaining a norm of ~52.28.

This layer 23 result means that, within the tested dataset, all ```<think>``` prompts received higher routing direction scores than all direct prompts. Unlike PCA, which shows high-variance direction without reference to the labels, this mean difference direction is more explicitly constructed from the contrast between the two modes. This therefore provides a more targeted estimate of the activation space axis which is associated with CoT reasoning. 

The fact that layer 23 produced the strongest score is consistent with layerwise PCA ad the probe results. The routing distinction is already detectable in earlier contextual layers, but becomes maximally separated immediately before the final layer and output projection. This suggests that the later network transforms the earlier distinction into a highly explicit output representation.

## Causal Intervention along the Steering Vector 

To test whether the layer 23 direction was correlated just with the response mode or was truly causally influenced, the direction was added to prompts which normally answered directly, and subtracted from prompts which generated think tags.

These interventions nearly reversed the model's routing behavior in both directions. The effect was particularly consistent when supressing think tags. When we subtracted the direction, the probability of think tags reduced near zero across all evaluated prompts. Adding the direction was also very effective, though there was still a small subset of direct prompts which resisted the intervention. 

From this alone, we can confidently say that layer 23 contains a compact and linearly accessible activation direction which causally controls the model's choice between direct and CoT response formats.

This activation direction should be interpreted as a CoT routing feature, not a general reasoning feature though. Its activation indicates that the model is preparing to enter the response format learned from the SCoTD data, whether this state also changes downstream reasoning quality, and not just the generation of think tags will need further analysis, a full generation intervention can help us decide concretely the importance of this layer. 



# Full Generation Causal Validation 

The purpose of this experiment is built on the result found in the previous section. We found that layer 23 contains an activation direction which causally controls the model's choice. 

With the full generation validation, we test 4 different tests, to see if this activation direction truly holds up, we want to see what this intervention actually changes. And so we test 4 scenarios:  

1\. Normal generation  
2\. Adding the layer 23 direction   
3\. Removing the layer 23 direction   
4\. A random direction added with the same control  


The question I want to answer with these 4 scenarios is to see if the direction activates or suppresses the complete learned CoT response mode, or see if it just manipulates the probability of the ```<th``` token. 


## First Iteration 

The first version of this experiment failed because of the way I applied the activation vector at every generated token, instead of only at the final prompt token where the routing direction was originally measured. 

Essentially, I believed that perhaps adding the same vector during every autoregressive decoding step would still produce coherent reasoning. 
The problem though, is that once generation begins, each token position represents a different computational step. And by reinjecting a large vector with said norm at every step can drive the model quite far off distribution:

```

{
	"id": "gsm8k_6630", 
	"used_think_label": true, 
	"condition": "add_direction", 
	"prompt": "In soccer, players receive yellow cards when they are cautioned and red cards when they are sent off. Coach Tim has a team of 11 players, 5 of them didn't receive cautions, the rest received one       yellow card each. How many red cards would the whole team collect, knowing that each red card corresponds to 2 yellow cards?", 
	"generated_text": "<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<think<th<think<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th<th", "opens_think": false, "closes_think": false, "reasoning_length_chars": 0, 
	"reasoning_length_tokens": 0, 
	"response_token_count": 256}
```


We can see a total collapse in the model, the pattern visible here is a good example of *oversteering*. 
So this shows that the persistent application of our routing direction throughout the decoding process destabilized generation, this tell us that the feature is position and state dependent.

## Iteration \#2 

In this version, I changed how the intervention is applied, and so now the layer 23 intervention is only applied during the initial prompt forward pass and at the final prompt token. The rest of the generated tokens will have no intervention applied. 

This new iteration produced clearer results and a better effect.

What I found is that adding the routing direction increased the overall ```<think>``` opening rate from 32.4% as a baseline, with new scores of:

35.3\% for $\alpha \in \{0.1, 0.25\}$  
55.9% at $\alpha = 0.75$  
97.1% at $\alpha = 1.0$  

where $\alpha$ denotes the strength of the sweep.

The random direction stayed the same at 35.3% across all strengths, showing that the effect is specific to the learned routing direction, and it is not merely affected by adding any random vector of the same norm.

The strongest evidence of these results comes from the prompts which normally answered directly:

Direct alpaca prompts went from 0% think rate to 90% for $\alpha=1.0$ 
Trivial arithmetic and factual prompts both went from 0% to 100% with the same 1.0 strength value. What strengthens this is that again, random directions did not trigger ```<think>``` in these groups. 


*example of a direct alpaca prompt with the added layer 23 direction*
```
{
	"id": "alpaca_14373", 
	"condition": "add_direction", 
	"prompt": "Name two world renowned artists.", 
	"generated_text": "<think> Two world renowned artists are Rembrandt and Vincent van Gogh.<|im_end|>\n<|endoftext|>", 
	"opens_think": true, "closes_think": false, 
	"alpha": 1.0
}
```

The main limitation of this strong direction addition is that as you can see, it often induces the opening tag without producing a well formed complete reasoning block.
For example, at $\alpha = 1.0$, 97.1% of responses opened a ```<think>``` block, but only 38.2% closed it. And for direct alpaca and factual prompts, almost all added direction outputs opened ```<think>```, with none of the responses closing it.


Subtraction of the layer 23 direction also works, but only at a sufficiently high strength value, lower strengths only change the ```<think>``` rate by about 3% from its 32.4% baseline, but $\alpha=1.0$ suppresses all uses of ```<think>```.


These results show that the layer 23 intervention causally controls entry into the learned ```<think>``` response mode in both a strength dependent and direction specific manner. Although strong activation primarily guarantee only the initiation of the CoT format, not a reliable end to end completion of a coherent thinking block.



# Activation space mechanisms & routing conclusion

All of the experiments so far have been quite useful to understanding the routing and the mechanisms controlling CoT and prompts modes.
If we look back at my original question, which in summary was:

"How does my model decide when to enter think mode, and can that decision be causally controlled?"

If we look at all the evidence so far, we answer this for each part of the experiments:

**Behaviourally,** the model does not trigger ```<think>``` just because a prompt looks like a gsm8k type prompt/question.
Computational structure matters more than surface style.

**Representationally,** think/direction behaviour becomes linearly separable very early and is strongly organized in the activation space by the intermediate and final layers. 

**Mechanistically,** we found that layer 23 contains a direction strongly associated with the routing decision

**Causally,** it was proven that adding this direction increases entry into ```<think>```, including on prompts that normally answer directly. Subtracting this direction can also suppress ```<think>```. To solidify the direction, we saw that adding a similar but random direction did not produce the same effect.

Combining this all together, there is a clear answer to the question:

*The model appears to form an explicit deeper layer activation space mechanism, which routes prompts between direct responses and the learned CoT format, this state can also be causally manipulated*


There is still one part that is potentially unresolved, given that we now have this direction:

"Does this direction activate *actual* reasoning, or does it only change the textual response format which is associated with CoT reasoning?"

The full generation causal results lean more towards the fact that it is more of a response mode routing, and not necessarily a general learned reasoning mechanism. At high strength, the direction reliably opens ```<think>```, but frequently does not close the reasoning block.


# Further Analyses 

Although the main question has been answered, we now know the existence of a learned activation space mechanism which is responsible for routing between the two formatted modes.

There is still more I want to find out about this model, and so I begin a few more experiments to further understand the architecture and model progression.

## Checkpoint differencing through weights

qwen-0.5b-reasoning was a model trained from the base model, qwen-2.5-0.5b, and then fine tuned to version 1, then finally version 2 of training. This experiment specifically investigates what changes additional fine-tuning made to the model.

What I found was that fine-tuning changed the model throughout the whole network, it was not only a single isolated layer, such as our layer 23 direction.
The MLP weight of the model changed much more than the attention weights on average. 

Though later attention projections in layers 21-23 were among the most strongly modified individual matrices. This suggests that the late layer routing representation may emerge from distributed changes, rather than this being installed exclusively at layer 23

<img src="/images/relative_change_heatmaps.png" alt="relative change heatmaps" />



The parameter differencing identified candidate layers and modules, but it did not determine which updates contributed specifically to the CoT routing. After inspecting these results, I decided a further analysis of the neuron-level could be interesting 


## MLP Activation Neuron Selectivity Analysis

The neuron metrics and analysis identified individual MLP neurons who final prompt token activations strongly distinguish ```<think>``` from direct response prompts.

The strongest candidate was layer 17 neuron 1036, this neuron had:
```
Standardized effect size: 6.10
AUROC: 0.998
correlation with routing direction projection: 0.959
relative checkpoint weight change: 0.104
```

This means its activation distributions are almost completely separated between the two behavioural classes, and its activation closely tracks the broader residual stream, routing score. 

Layer 23 neurons 2986 and 4469 also show similar extreme selectivity. For example, neuron 449 has very similar metric results, similar to L17 N1036, though its mean activation difference is larger at 10.47.

These results are quite interesting, the effects here are not subtle at all, these individual neurons are nearly sufficient to classify the observed behavior on the factorial dataset.


The analysis also showed the existence of opposing neuron populations as well. Through the experiment, I measured a metric, ```mean_diff```.

A positive mean_diff means that the neuron is more active when a prompt induces reasoning
And a negative mean_diff shows the opposite effect, active for direct response, or for suppressed ```<think>``` prompts.

Our earlier example of L17 N1036 can be seen as a strongly *think positive* neuron.

But examples like L17 N654 and L21 N1083 can be seen a *direct positive* neurons, 
with both neurons having similar metrics:
```
L17 N654:
effect size -5.06
AUROC: 0.0035
correlation: -0.952

L21 N1083:
effect size -5.01
AUROC: 0.0046
correlation: -0.931
```

The near zero AUROC scores indicate an almost perfect reversed ordering, once direction is accounted for, these neurons are about as selective as the think positive neurons.

This could suggest that the routing state is represented through a contrastive populations.
One population becomes more active in ```<think>``` states, and other populations become more active in direct states, and then the residual stream reflects their combined contribution.

<!--![[compact_top_neuron_heatmap.png]]-->
<img src="/images/compact_top_neuron_heatmap.png" alt="top neuron heatmap" />




Another interesting phenomenon is the richness of layer 17 in terms of selective neurons, across all 4,864 neurons in each tested layer, layer 17 had the strong overall neuron level selectivity:


| Layer | Mean Abs. Effect | 95th percentile | Max   |
| ----- | ---------------- | --------------- | ----- |
| 17    | 0.806            | 2.324           | 6.096 |
| 21    | 0.657            | 1.845           | 5.013 |
| 22    | 0.575            | 1.682           | 4.620 |
| 23    | 0.742            | 2.057           | 5.749 |

In addition, layer 17 also contributed most of the highest ranked candidate neurons.

A possible mechanistic hypothesis I have for this layer is that *layer 17 contains a rich population representation of the task or response mode. Deeper layers aggregate and transform the evidence from L17 into a more explicit output routed state*

This hypothesis would explain why the best single neurons appear earlier than what is the actual discovered intervention layer (L23)


Given all of these results, I further evaluated neurons through top-neuron ablation. 

To do the ablation, I removed the ten highly ranked neurons across layers 17, 21, and 23.

The aggregate results from this ablation were:
```
all prompts: mean probability change -0.0085
direct prompts: mean change ~0
think prompts: mean change -0.0170  
```

Though these results show neuron ablation is causal, the actual effect seems quite small

Although, what can be seen is that this selected top-neuron ablated population contributes positively to CoT routing, especially for prompts whose baseline routing decision is weak or unsure.

The modest average effect and resilience of the high confidence think prompts tell us that these neurons are neither individually, or jointly necessary for the whole behavior. 


An interesting result is the comparison between the effects of the neuron-level and the direction-level interventions.
In an earlier section, we saw that through direction subtraction at layer 23, we could suppress ```<think>``` nearly universally, whereas ablating the top neurons had a much smaller average effect.

The difference is that the result stream direction intervention did earlier edits the model's integrated routing state, on the other hand, the neuron ablation only removes several upstream contributors, **other neurons and modules can still reconstruct or preserve** the same residual feature.


## Mechanistic Account

Connecting all of these new analyses together gives us a coherent hierarchy of the system:  

1\. Fine tuning modifies parameters *throughout* the model  
2\. middle to deep MLP neuron populations encode think/direct evidence  
3\. Deeper layers integrate this evidence into a highly explicit residual stream routing direction  
4\. Editing the integrated direction has a much large causal effect than remove a small number of individual contributors   


