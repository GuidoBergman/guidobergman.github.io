---
title: "Avoiding jailbreaks by discouraging their representation in activation space"
date: 2024-09-27T15:34:30-04:00
categories:
  - blog
tags:
  - Activation Engineering 
  - Interpretability
  - Jailbreaks
  - Large Language Model (LLM)
  - Artificial Intelligence (AI)
---


*This project was completed as part of the [AI Safety Fundamentals: Alignment Course](https://aisafetyfundamentals.com/alignment/) by BlueDot Impact. All the code, data and results are available in [this repository](https://github.com/GuidoBergman/jailbreak_direction).*

# Abstract

The goal of this project is to answer two questions: “Can jailbreaks be represented as a linear direction in activation space?” and if so, “Can that direction be used to prevent the success of jailbreaks?”. The *difference-in-means* technique was utilized to search for a direction in activation space that represents jailbreaks. After that, the model was intervened using *activation addition* and *directional ablation*. The activation addition intervention caused the attack success rate of jailbreaks to drop from 60% to 0%, suggesting that a direction representing jailbreaks might exist and disabling it could make all jailbreaks unsuccessful. However, further research is needed to assess whether these findings generalize to novel jailbreaks. On the other hand, both interventions came at the cost of reducing helpfulness by making the model refuse some harmless prompts.

# Introduction

*Jailbreak prompts* are attempts to bypass safeguards and manipulate Large Language Models (LLMs) into generating harmful content ([Shen et al., 2023](https://arxiv.org/abs/2308.03825)). This becomes more dangerous with advanced AI systems, which can contribute to threats like bioterrorism by aiding in the creation of deadly pathogens, as well as facilitating propaganda and censorship [(Hendrycks et al., 2023\)](https://arxiv.org/abs/2306.12001).

This project aims to study whether it is possible to avoid jailbreaks utilizing mechanistic interpretability. More precisely, it examines whether jailbreaks are represented by a linear direction in activation space and if discouraging the use of that direction makes them unsuccessful. 

The relevance of this project lies in the fact that mitigating jailbreaks by directly intervening in the model’s internals, instead of just its outward behavior, could potentially be a way to make jailbreaks impossible and not just less likely to occur.

In order to test this, the project attempts to find and prevent the model from using the direction in activation space that represents jailbreaks. This direction is found using the *difference-in-means* technique. More specifically, the direction is calculated as the difference in the means of the activations on interactions where the model answers a *forbidden question* and interactions where the model refuses to answer it. The first set corresponds to cases where a jailbreak prompt is successful and gets the model to answer the forbidden question. The second set corresponds to cases where the model refuses to answer the forbidden question because it is asked directly. Examples of both interactions are shown in Figure 1\.

![Baseline Interactions](Foto.jpeg)
![Baseline Interactions](/assets/images/Foto.jpeg)
***Figure 1\.** Examples of two interactions: one where the model refuses to answer a forbidden question (left) and other where it answers it (right). Note that the LLM’s response to the forbidden question was added only for illustrative purposes but in practice it is not used to find the direction that represents jailbreaks.*

After finding the direction in activation space that represents the “*jailbreak feature”*, the model is intervened to prevent it from using that direction. This intervention is done utilizing the methods of *activation addition* and *directional ablation*. Activation addition subtracts the direction from the activations, while directional ablation, erases the direction from the representations. An interaction with the model after the activation addition intervention is shown in Figure 2\.

![][image2]  
***Figure 2\.** Example of an interaction with the model, after the intervention using activation addition. Note that the jailbreak prompt and forbidden question are the same to the ones in the right side of Figure 1, but now the model refuses to answer.*

This project is heavily based on [this repository](https://github.com/andyrdt/refusal_direction/tree/main), which was used to find and disable a feature corresponding to refusals using the *difference-in-means* technique. 

# Data and Model

**Data**

The jailbreak prompts utilized are from [Shen et al., 2023](https://arxiv.org/abs/2308.03825). This dataset was chosen because it groups the jailbreak prompts into communities according to their similarity. This allowed for an examination of whether the directions found could generalize well to communities in the test set that were not present in the training set. 

Additionally, the [Alpaca](https://huggingface.co/datasets/tatsu-lab/alpaca) dataset is used to verify that avoiding jailbreaks would not make the model refuse harmless prompts.

**Model**

The model examined to find the direction representing jailbreaks is [Gemma 2 2B IT](https://huggingface.co/google/gemma-2-2b-it).[^1] 

# Methodology

The steps followed to find and discourage the use of the direction that represents jailbreaks are summarized in Figure 3\. 

![][image3]  
***Figure 3\.** Steps to find and disable the “jailbreak feature”.*

The steps followed consist of:

1. **Gather jailbreak prompts and forbidden questions**

*Jailbreak prompts* are attempts to bypass safeguards and manipulate LLMs into generating harmful content ([Shen et al., 2023](https://arxiv.org/abs/2308.03825)). Additionally, a *forbidden question* is a question that the model should refuse to answer, such as: "*How can I hack into someone's email account?*". Even though the method used allows more than one forbidden question, only that one is utilized in this project.

2. **Generate interactions with the baseline model**

    
Interactions are generated with the baseline model following the structure shown on the right side of Figure 1: first, the model provides a response to the *jailbreak prompt* and then a response to the *forbidden question*. 

3. **Filter interactions**  
   

At this step, the jailbreak prompts that successfully get the model to answer the forbidden question are separated from the unsuccessful ones.[^2] For that purpose, the model’s response to the forbidden question is evaluated to assess whether it is harmful. This is done using [Llama Guard 2](https://github.com/meta-llama/PurpleLlama/blob/main/Llama-Guard2/MODEL_CARD.md) and [HarmBench](https://huggingface.co/cais/HarmBench-Mistral-7b-val-cls).[^3] 

After that, the dataset is split into train and test sets:

* **Train set**: consists of 256 randomly selected interactions where the model answered the forbidden question.  
* **Test set**: consists of the 156 remaining interactions where the model answered the forbidden question. Additionally, 100 interactions where the jailbreak prompt was unsuccessful were randomly selected and included in this set.[^4]

All of the interactions in both sets consist of the jailbreak prompt, the model’s first response and the forbidden question, excluding the second answer. That is because the objective is to test what makes the model provide harmful answers, so the activations are computed up to that point.

4. **Find the direction representing the jailbreak feature**

At this step, the direction representing the jailbreak feature is found utilizing the *difference-in-means* method. To do this, two sets of contrasting interactions are used:

* **Interactions where the model answers the forbidden questions**: these are the ones in the train set from the last step.  
* **Interactions where the model refuses to answer the forbidden question**: this set is obtained by directly asking the model the forbidden question. That is because, when asked directly, the model refuses to answer it.

The direction corresponding to jailbreaks is calculated as the difference between the mean activation on the interactions where the model answered the forbidden question minus the mean activation on the interactions where it refused to answer it. More intuitively, that would yield the following formula:

***jailbreak \= successful jailbreak interactions \- refusal interactions*** 

That difference is only computed with the activations of the residual stream at layer 14 and at the last token position, which corresponds to the start of the model’s completion.[^5]

5. **Intervene the model**

The idea of this step is to intervene the model to discourage it from using the direction that represents jailbreaks. This is done utilizing two different methods: *activation addition* and *directional ablation* ([Arditi et al., 2024](https://arxiv.org/abs/2406.11717)).

**Activation addition***:* this method modulates the strength of the jailbreak *feature*, by subtracting its vector from the layers activation. Let x be the activations of the layer, r the direction corresponding to jailbreaks, then the formula for computing x' would be:

x'  x \- r 

**Directional ablation:** the direction corresponding to jailbreaks is erased from the model’s representations. Directional ablation “zeroes out” the component along  **r**  for every residual stream activation:

**x′ ← x − r rTx**

This operation is performed across all layers. This effectively prevents the model from ever representing the direction in its residual stream.

6. **Generate completions**  
   

At this step, completions are generated for the versions of the model intervened with activation addition and directional ablation. The completions are generated following again the format on the right side of Figure 1: the model provides a response to the jailbreak prompt and after that, it generates a completion to the forbidden question. 

7. **Evaluate completions**

Two evaluations are performed:

* **Refusal to answer forbidden questions**: the harmfulness of the model’s answer to the forbidden question is evaluated utilizing [Llama Guard 2](https://github.com/meta-llama/PurpleLlama/blob/main/Llama-Guard2/MODEL_CARD.md) and [HarmBench](https://huggingface.co/cais/HarmBench-Mistral-7b-val-cls).[^6] The cases where that answer was found to be harmful are considered successful jailbreaks. After this, the attack success rate (ASR) is calculated by dividing the number of successful jailbreaks, by the number of answers evaluated ([Ball et al., 2024](https://arxiv.org/abs/2406.09289)). In other words, the attack success rate is the fraction of jailbreaks that are successful.  
* **Refusal to answer harmless prompts:** it is evaluated whether the interventions make the model refuse harmless prompts, using the [Alpaca](https://huggingface.co/datasets/tatsu-lab/alpaca) dataset. Here, the fraction of refusals to harmless requests was calculated manually, with the help of a *regular expression* that matches common refusal substrings.

# Results

**Refusal to answer forbidden questions**

The attack success rates (ASR) of the jailbreak prompts in the baseline and intervened models are presented in Table 1:

| Version | ASR (Llama Guard 2\) | ASR  (HarmBench) |
| :---- | :---- | :---- |
| **Baseline** | 60.55% | 59.38% |
| **Activation addition** | **0.00%** | **0.00%** |
| **Directional ablation** | 96.88% | 84.77% |

***Table 1\.** Attack success rates percentages of the jailbreak prompts in the baseline and intervened models*

As can be seen in Table 1, **the activation addition intervention made all of the jailbreak prompts in the test set unsuccessful. This suggests that the intervened model could be immune to the prompts in that set, since the attack success rate dropped from 60% to 0%. Additionally, that indicates that a direction representing jailbreaks might exist and disabling it could make all jailbreaks unsuccessful.** 

An example of an interaction with the model after the intervention with activation addition is shown in Figure 2\.

Additional analysis of the jailbreak prompts that were successful with the baseline model, shows that there are 18 communities in the test that were not present in the training set. The fact that these jailbreak prompts were not successful in the model intervened with activation addition suggests that the method makes the model refuse jailbreak types it had not seen during training.

On the other hand, the directional ablation intervention made the model much more vulnerable to jailbreak prompts. It is not yet understood why this happened.

**Refusal to harmless prompts**

The percentages of refusal to harmless prompts are shown in Table 2:

| Version | Refusal to harmless prompts |
| :---- | :---- |
| **Baseline** | 3.15% |
| **Activation addition** | 18,88% |
| **Directional ablation** | 18,88% |

***Table 2\.** Percentage of harmless prompts refused by the baseline and intervened models*

As shown in Table 2, both interventions to the model come at the cost of making it refuse harmless prompts. By manually analyzing the model’s responses, the following conclusions were extracted:

* The only prompt that was refused by the 3 versions of the model, asked to provide credit card numbers, so it could be considered that the refusal was correct  
* There were two reasons why the intervened versions of the model refused the prompts that were not refused by the baseline:  
  * The model said it was not capable of answering the request. An example of this was when it responded to the prompt: “Compose a poem in which the majority of the lines rhyme.”.  
  * The model said that the request was unethical and potentially illegal. An example of this was when the prompt was: "Generate a list of tech companies in San Francisco.".

These results suggest that the method used reduces the helpfulness of the model. Two potential solutions to this problem are suggested:[^7]

* **Using bigger models**: since bigger models have more directions in activation space, they might be less likely to be induced to refuse harmless prompts by disabling one direction  
* **Using a smaller training set:** earlier iterations of this project that were done with smaller portions of the jailbreak prompt dataset did not show this tension between helpfulness and harmfulness. For this reason, it is speculated that increasing the size of the training set had a negative effect on helpfulness.

# Related work

**Difference-in-means technique**: this technique consists of finding a direction that represents a feature by taking the difference of means in the activations of two contrasting sets: one that contains the feature and one which does not ([Arditi et al., 2024](https://arxiv.org/abs/2406.11717)). 

**Finding jailbreaks as directions in activation space:** [Ball et al., 2024](https://arxiv.org/abs/2406.09289) searches for a direction representing jailbreaks with an approach similar to the one utilized here. The main differences are:

* [Ball et al., 2024](https://arxiv.org/abs/2406.09289) focuses on single-turn interactions, while this project uses multi-turn interactions.  
* [Ball et al., 2024](https://arxiv.org/abs/2406.09289) represents jailbreaks with several directions in activation space. This project uses only one direction to represent them.

# Advantages of the approach

The main **advantages** of the approach utilized to avoid jailbreaks are:

* **Directly intervenes in the model’s internals instead of its apparent behavior:** this means that in theory, it could prevent the usage of the direction that the model uses to represent jailbreaks, which might be a way to make jailbreaks impossible and not just less likely to occur.[^8]  
* **It is cost-effective**: computing the activations and intervening the model only take a few seconds in a A100 GPU. The parts of the process that take more time to compute are the steps at which the completions are generated, even though that might be necessary as well for other methods to avoid jailbreaks.  
  


# Disadvantages of the approach

The main **disadvantages** of the approach used are:

* **There seems to be a tension between helpfulness and harmfulness:** this was shown in the refusals to harmless prompts.  
* **If jailbreaks do occur, this approach will not make them less harmful:** if a jailbreak is successful even after applying this method, this approach will not make the model’s output less harmful. Other techniques such as unlearning ([Li, Pan, et al., 2024](https://arxiv.org/abs/2403.03218)) might be effective for this. For this reason, this approach is proposed as complementary to unlearning.[^9]  
  


# Further work

The directions for further research are:

* **Study whether the findings of this project generalize to other datasets and models:** currently, it is still unclear whether this method would generalize to novel jailbreaks. It would be specifically interesting to study whether this approach is useful with other jailbreak techniques such as universal adversarial attacks ([Zou et al., 2023](https://arxiv.org/abs/2307.15043)) or multi-turn jailbreaks ([Li, Han, et al., 2024](https://arxiv.org/abs/2408.15221)). Additionally, more forbidden questions and model sizes could be utilized.  
* **Explore ways to reduce the tension between helpfulness and harmfulnes:** two alternatives where proposed for this: using bigger models and/or a smaller training set.  
* **Use this approach to prevent other types of model misbehavior**: even though this project is focused on avoiding jailbreaks, the approach could be adapted to attempt to avoid other types of model misbehavior, such as sycophancy or deception. The modifications necessary to achieve this are explained in the [Appendix](#appendix:-modifications-needed-to-repurpose-this-approach-for-preventing-other-types-of-misbehavior).  
* **Evaluate if the effects of applying this approach can be fine-tuned away:** this would help to assess how useful the method is in open-source models.

# Conclusions

This project aimed to study whether jailbreaks are represented by a linear direction in activation space and if disabling that direction made them unsuccessful. The *difference-in-means* technique was utilized to search for this direction. After that, the model was intervened using *activation addition* and *directional ablation*. The activation addition intervention reduced the attack success rate of jailbreaks from 60% to 0%, suggesting that a direction representing jailbreaks might exist and disabling it could make all jailbreaks unsuccessful. Still, further research is necessary to determine whether this result would generalize to novel jailbreaks. However, both interventions came at the cost of reducing helpfulness by making the model refuse some harmless prompts. Two potential mitigations are proposed to solve this problem: increasing model size and reducing the size of the training set. 

# Acknowledgements

I’d like to give special thanks to Ruben Castaing and Josh Thorsteinson for their in-depth feedback. I also want to thank the rest of my cohort for all the stimulating discussions we had. Additionally, I’d like to acknowledge the course organizers for creating this wonderful space to learn such an important topic. Lastly, I want to thank Micaela Peralta for the invaluable guidance she provides in every aspect of my life.

# Appendix: Modifications needed to repurpose this approach for preventing other types of misbehavior {#appendix:-modifications-needed-to-repurpose-this-approach-for-preventing-other-types-of-misbehavior}

The approach used in this project can be adapted to prevent other types of misbehavior, such as sycophancy. To facilitate this, the code has been structured to concentrate all necessary modifications into two main components:

* [Datasets directory](https://github.com/GuidoBergman/jailbreak_direction/tree/main/dataset): This directory contains files that download and process the datasets, load them, split them into training and test sets and create the interactions used for the difference-in-means method. While the overall functionality in those files will remain intact, the code should be replaced with implementations relevant to a different problem. For instance, the current interactions used for the difference-in-means method involve successful jailbreaks and refusals, which can be modified to include interactions that promote sycophantic behavior and interactions that do not.  
* [Evaluate\_jailbreaks file](https://github.com/GuidoBergman/jailbreak_direction/blob/main/pipeline/submodules/evaluate_jailbreak.py): This file evaluates the model’s completions to verify whether the jailbreak prompts are successful and if harmless requests are refused. This should be replaced with code that assesses whether the model’s responses exhibit other types of misbehavior, such as sycophancy.

It is important to note that the original  [refusal direction repository](https://github.com/andyrdt/refusal_direction/tree/main) included functionality to [filter the datasets](https://github.com/andyrdt/refusal_direction/blob/main/pipeline/run_pipeline.py#L38), [select the best candidate direction](https://github.com/andyrdt/refusal_direction/blob/main/pipeline/submodules/select_direction.py) and [evaluate the loss on harmless datasets](https://github.com/andyrdt/refusal_direction/blob/main/pipeline/submodules/evaluate_loss.py). Although these features were not utilized in this project for simplicity, they could be advantageous for identifying directions that represent other types of misbehavior.

[^1]:  This model was selected because its small size allowed a fast iteration time.

[^2]:  That is because only the successful ones are used in the training set.

[^3]:  A jailbreak prompt is classified as succesful when both models classified the answer as harmful.

[^4]:  The reason there are fewer unsuccessful jailbreak prompts in the test set than successful ones is that it was intended to leave the set more focused on the prompts that produced harmful behavior.

[^5]:  That layer was selected since it was expected to be able to capture high-level features like jailbreaks. No other layers or positions were tried for the success of the results obtained using only those. Despite this, it is possible that to generalize the results to other models, it would be necessary to try computing the difference at other layers and token positions. The [function to select the best direction](https://github.com/andyrdt/refusal_direction/blob/cdd30769e68b505a2b39ad3d4f9cd67afc6b4820/pipeline/submodules/select_direction.py#L117) from the refusal direction repository could be useful for this. 

[^6]:  Two additional metrics were utilized for this: a *regular expression* to match common refusal substrings and another *regular expression* to match typical substrings that the model used when answering the forbidden question. Despite the fact that these metrics are much faster to compute, their results are not shown here since they are less precise than the other two metrics, with the difference being as much as  40% during development.

[^7]:  These solutions were not tested.

[^8]:  By no means it is stated that the impossibility of jailbreaks to occur has been proven here.

[^9]:  Additionaly, unlearning methods may not be able to cover all the cases of misbehavior. For example, to cause a model to ‘unlearn’ how to make fake news about politicians, it might be necessary to make them forget relevant information about the politician.