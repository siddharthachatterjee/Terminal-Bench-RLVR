### 1. System Environment



1. **Prompt Definition:** The RL Agent receives the system prompt:


> 'You are an expert terminal agent. Given a task, reason step by step inside `<think>...</think>` tags, then output your final bash solution inside `<answer>...</answer>` tags. Output ONLY these two sections.'
> 
> 


The prompt will be a series of tasks, following the terminalbench2 format, with an instruction, reference script, test script, and difficulty categorization. The prompt is tokenized to a maximum of 512 tokens, with no other embeddings or environment state outside the token sequence.


2. **Action Space:** The action space is the autoregressive token generation over the vocabulary of the model. The response is capped at 512 tokens with the expected format of the system prompt. The agent applies corrections through updating its policy to favor higher-reward completions.


3. **Reward Functions:**

* **Accuracy:** This is the highest reward function, giving a reward of 1.0 if the generated bash script passes the test script and 0.0 otherwise. It runs the bash in a temporary directory using a subprocess (in actual production, I'd likely shift this to a Docker container).


* **Format:** Reward of 0.1 if the generation matches the format of the system prompt exactly, 0.0 otherwise.


* **Length Penalty:** Penalizes -0.2 if the completion exceeds a threshold token limit. This is to encourage efficient memory usage.





---

### 2. Model Selection and Training Plan



| Component | Description |
| --- | --- |
| **Base Model Selection** | **Qwen/Qwen2.5-7B-Instruct** <br>

<br> It's instruction-tuned so it has a good starting point for chat-based evaluation which I'll be performing in the task. It's also shown strong performance in code and terminal based tasks. Additionally, Qwen is a lightweight model that I can run on an A100 NVIDIA GPU using my education account access.

 |
| **RL Algorithm** | **GRPO** <br>

<br> GRPO is more memory efficient than PPO, and the GRPO library has the ability to customize reward functions with verifiable reward values. During each training step, $G=8$ completions are generated for each prompt. Advantages are computed relative to the group mean, normalized by the group standard deviation. The policy is updated to increase the probability of higher-reward completions relative to lower-reward ones within the group.

 |
| **Dataset Size / Configuration** | Used an LLM to generate list of tasks matching the format and difficulty spread of TerminalBench2. Tasks span five categories: file operations, data processing, networking, system administration, and scripting across three difficulty levels: easy (5 tasks), medium (6 tasks), and hard (6 tasks, including 5 novel synthetic tasks created as a stretch goal). Each task includes a natural language instruction, a reference bash solution, and a pytest-style test script that serves as the reward verifier. The dataset is split $80/20$ into train and eval. Currently, tested on a small dataset of 17 tasks so I could ensure quality of tasks and test easily, but in production would scale this to 500+ tasks using a script to generate tasks and run the bash to verify their validity.

 |
| **Hyperparameters** | * **Learning Rate:** 5e-6

<br>

<br>* **Batch Size:** 2 per device (using the A100 GPU)

<br>

<br>* **Optimizer:** AdamW

<br>

<br>* **Epochs:** 3

<br>

<br>* **Number of Generations:** 8

<br>

<br>* **Max Completion Length:** 512

<br>

<br>* **Temperature:** 0.9

<br>

<br>* **Warm Up Steps:** 20

<br>

<br>* **Max Steps:** 500

<br>

<br>* **Gradient Accumulation Steps:** 8

 |
| **Metrics** | * **Reward** for each completion and reward function

<br>

<br>* **Overall success rate** for each reward func

<br>

<br>* **Eval pass@1:** greedy pass@1 accuracy on the held-out eval set, measured every 100 steps. Ground truth training signal for whether the policy is actually improving.

 |
