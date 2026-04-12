# Relation-Grounded Latent Reasoning

This framework trains Qwen2.5-VL to answer visual questions through an intermediate latent reasoning phase. Instead of asking the model to write explicit textual reasoning, we let it use latent hidden states to represent question-relevant visual relations.

The model generates answers in the following form:

```text
<|lvr_start|> <|lvr|> ... <|lvr|> <|lvr_end|>
<answer> ... </answer>
```

Each internal `<|lvr|>` position is a latent reasoning step. The token is only a placeholder; the actual reasoning content is the hidden state at that position.

## Relation-Grounded Reasoning

For each training example, we select the scene-graph relations that are relevant to the question. If the question mentions or grounds certain objects, we supervise all relations connected to those objects.

For example, if the question refers to a suitcase, the selected relations may include:

```text
suitcase has_attribute black
suitcase left_of person
wall behind suitcase
```

If `K` relations are selected, the model receives `K` latent reasoning steps. The order is deterministic, so the `t`-th latent step is aligned with the `t`-th selected relation.

The relation vocabulary is built from canonicalized relation labels in the training data. Each selected relation provides a relation id and subject/object grounding targets.

## Model Architecture

The framework uses Qwen2.5-VL as the base vision-language model. The vision encoder provides visual token states, and the language model produces the latent reasoning tokens and final answer tokens.

On top of the latent reasoning states, we add a lightweight relation-grounding module. For a sample with `K` latent steps and `N` visual tokens, the module receives:

```text
latent states: h_1, h_2, ..., h_K
visual states: v_1, v_2, ..., v_N
```

Both latent and visual states are projected into a shared relation hidden space. A small cross-attention transformer then lets each latent state attend to all visual tokens, producing an updated reasoning state.

From each updated reasoning state, the module produces two queries:

```text
subject query
object query
```

These queries generate two attention distributions over visual tokens:

```text
subject attention
object attention
```

The attentions pool subject and object visual contexts. The relation head then predicts the relation label from:

```text
[updated reasoning state ; subject visual context ; object visual context]
```

This design makes each latent step both relation-aware and visually grounded.

## Training Objective

The training objective combines four losses. Each one supervises a different part of the framework.

### Boundary Loss: `L_boundary`

The model learns the boundary of the latent reasoning phase by predicting:

```text
<|lvr_start|>
<|lvr_end|>
```

The internal `<|lvr|>` placeholders are not trained with next-token cross entropy. This is important: we do not want latent states to be forced toward the embedding of a fixed placeholder token.

`L_boundary` therefore teaches the model when to enter and exit latent reasoning, but it does not supervise the content of the internal latent steps.

### Answer Loss: `L_answer`

The final answer is trained with standard supervised fine-tuning inside:

```text
<answer> ... </answer>
```

`L_answer` teaches the model to produce the final task answer after finishing the latent reasoning phase.

### Relation Loss: `L_relation`

Each latent step predicts the relation label of its aligned selected relation:

```text
L_relation = CE(predicted_relation_t, relation_label_t)
```

This teaches each latent hidden state what relation it should represent.

### Grounding Loss: `L_ground`

Each selected relation has subject and object grounding targets. Their bounding boxes are mapped to the visual-token grid and used to supervise the subject/object attention distributions.

This is the loss that explicitly encourages the model's attention to concentrate on the relevant objects. For each latent step, the model predicts:

```text
subject attention over visual tokens
object attention over visual tokens
```

The ground-truth subject/object boxes are converted into soft attention targets on the visual-token grid. `L_ground` penalizes the distance between the predicted attention distributions and these box-derived targets.

For attribute relations such as:

```text
suitcase has_attribute black
```

the subject and object grounding targets can refer to the same object region.

### Full Objective

The full training loss is a weighted sum:

```text
L = lambda_boundary * L_boundary
  + lambda_answer   * L_answer
  + lambda_relation * L_relation
  + lambda_ground   * L_ground
```

The first two losses train the language-level structure and final answer. The last two losses train the internal latent reasoning steps to be relation-aware and visually grounded.

## Inference

At inference time, the model generates the latent reasoning span and the final answer itself.

There are two useful modes:

### Free Reasoning

The model decides how many latent steps to generate and exits the latent phase when it emits:

```text
<|lvr_end|>
```

### Step-Budget Reasoning

The model still generates the latent reasoning phase by itself, but an external budget limits the maximum number of `<|lvr|>` steps. Once the budget is reached, the latent phase is forced to end.

This is useful for comparing different reasoning budgets, such as 4, 8, or 16 latent steps.

## Summary

The framework is designed to make latent reasoning structured, visually grounded, and less dependent on textual chain-of-thought.

The model does not write intermediate triplets as text. Instead, each latent step is supervised by:

- a relation label,
- subject/object visual grounding,
- and final answer correctness.

This keeps reasoning in continuous hidden states while still giving each step explicit visual and relational supervision.
