<p align="center">
    <a href="#-features">🚀 Features</a> •
    <a href="#-dataset">🤗 DataSet & Models</a> •
    <a href="#-quick-start">🔥 Quick Start</a> •
    <a href="#-case-study">➕ Case Study</a> •
    <a href="#-citation">📜 Citation</a> •
    <a href="#-license">📝 License</a>
</p>


**CODE-DITING** is a novel code generation evaluation metric that directly assesses the functional alignment between problem descriptions and generated code. 
This approach overcomes the limitations of traditional reference solutions and test case methods. While maintaining high accuracy, it achieves efficient computational performance and provides excellent explainability.

## 🚀 Features

- *Accuracy*: CODE-DITING outperforms existing state-of-the-art methods across multiple datasets.
- *Efficiency*: Using smaller models (1.5B and 7B parameter scale), it achieves comparable or even superior results to large-scale models (such as GPT-4o and DeepSeek-V3 671B) with only about 1% of the parameter count.
- *Explainability*: By introducing reasoning paths, the evaluation process becomes more transparent, making it easier to understand the basis for the model's judgments.

## 🤗 DataSet & Models

We provide datasets for training and evaluation, as well as pre-trained models:

- **Datasets**:
  - [CodeJudge](https://huggingface.co/datasets/Code-DiTing/CodeJudge_17k) - Dataset for training
  - [HumanEval-Judge](https://huggingface.co/datasets/Code-DiTing/HumanEval-Judge) - Dataset for HumanEval-Judge evaluation
  - [MBPP-Judge](https://huggingface.co/datasets/Code-DiTing/MBPP-Judge) - Dataset for MBPP-Judge evaluation
  - [BigCodeBench-Judge](https://huggingface.co/datasets/Code-DiTing/BigCodeBench-Judge) - Dataset for BigCodeBench-Judge evaluation

- **Models**:
  - [CODE-DITING-1.5B](https://huggingface.co/datasets/Code-DiTing/BigCodeBench-Judge) - 1.5B parameter model
  - [CODE-DITING-7B](https://huggingface.co/Code-DiTing/Code-DiTing-7B) - 7B parameter model

## 🔥 Quick Start

- **How to train**

```python
from unsloth import FastLanguageModel
import torch
max_seq_length = 4096 * 2 
dtype = None 
load_in_4bit = False 

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "deepseek-ai/DeepSeek-R1-Distill-Qwen-7B", 
    max_seq_length = max_seq_length,
    dtype = dtype,
    load_in_4bit = load_in_4bit,
)
print("model init ...")
model = FastLanguageModel.get_peft_model(
    model,
    r = 32,
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                      "gate_proj", "up_proj", "down_proj",],
    lora_alpha = 16,
    lora_dropout = 0,
    bias = "none",    
    use_gradient_checkpointing = "unsloth", 
    random_state = 3407,
    init_lora_weights = "pissa", # NEED TO MODIFY THE CODE IN UNSLOTH
    use_rslora = False,  
    loftq_config = None,
)
print("lora init ...")
from datasets import load_dataset
dataset = load_dataset("csv", data_files="CodeJudge_17k.csv", split="train")
print("dataset init ...")

from trl import SFTTrainer
from transformers import TrainingArguments, SchedulerType
from unsloth import is_bfloat16_supported

trainer = SFTTrainer(
    model = model,
    tokenizer = tokenizer,
    train_dataset = dataset,
    dataset_text_field = "text",
    max_seq_length = max_seq_length,
    dataset_num_proc = 2,
    packing = True,
    args = TrainingArguments(
        per_device_train_batch_size = 4,
        gradient_accumulation_steps = 4,
        warmup_ratio = 0.1,
        num_train_epochs = 3,
        save_strategy="epoch",
        output_dir="./results",
        learning_rate = 1e-4,
        fp16 = not is_bfloat16_supported(),
        bf16 = is_bfloat16_supported(),
        logging_steps = 1,
        weight_decay = 0.01,
        lr_scheduler_type = "linear",
        seed = 3407,
        report_to = "none", 
    ),
)

gpu_stats = torch.cuda.get_device_properties(0)
start_gpu_memory = round(torch.cuda.max_memory_reserved() / 1024 / 1024 / 1024, 3)
max_memory = round(gpu_stats.total_memory / 1024 / 1024 / 1024, 3)
print(f"GPU = {gpu_stats.name}. Max memory = {max_memory} GB.")
print(f"{start_gpu_memory} GB of memory reserved.")

trainer_stats = trainer.train()
model.save_pretrained_merged("save_model_code_diting_7b", tokenizer, save_method = "merged_16bit",)
```


- **How to inference**

```python
from tqdm import tqdm
from transformers import AutoTokenizer
from vllm import LLM, SamplingParams
import torch

template = """Determine the correctness of the code snippet. 
Please reason step by step within <think></think>, and put your final answer within <answer></answer>.

Problem Statement:
{{PROBLEM}}

Code Snippet:
{{CODE}}

"""

def main(n):
    messages = []
    nl = '''Check if in given list of numbers, are any two numbers closer to each other than given threshold.'''
    code = '''from typing import List


def has_close_elements(numbers: List[float], threshold: float) -> bool:
  for i in range(len(numbers)):
    for j in range(i + 1, len(numbers)):
      if abs(numbers[i] - numbers[j]) < threshold:
    return True
  return False
    '''
    content = template.replace("{{PROBLEM}}", str(nl)).replace("{{CODE}}", str(code))
    message = [
        {
            "role": "user",
            "content": content
        }
    ]
    messages.append(message)
    model_path = 'Code-DiTing/Code-DiTing-7B'
    sampling_params = SamplingParams(temperature=0.6, max_tokens=4096*2, n=n)
    llm = LLM(model=model_path, trust_remote_code=True, max_model_len=4096*2, gpu_memory_utilization=0.85)
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    inputs = []
    for message in tqdm(messages, total=len(messages)):
        prompt = tokenizer.apply_chat_template(message, tokenize=False, add_generation_prompt=True)
        inputs.append(prompt)
    outputs = llm.generate(inputs, sampling_params)
    for i in range(len(outputs)):
        n = len(outputs[i].outputs)
        for j in range(n):
            out = outputs[i].outputs[j].text
            print(out)


if __name__ == '__main__':
    main(n=7)
```

## ➕ Case Study

We've conducted a comprehensive analysis of both successful and unsuccessful CODE-DITING predictions. Our findings are documented in two separate files:

1. **[Case_Study.md](./Case_Study.md)** - Contains examples of correct evaluations.

2. **[Failure_Case_Study.md](./Failure_Case_Study.md)** - Examines cases where CODE-DITING made incorrect judgments, categorizing failure patterns and providing insights for future improvements.

These case studies offer valuable insights into the model's reasoning capabilities, highlighting both its strengths in accurate code evaluation and opportunities for enhancement in handling ambiguous problem descriptions and preventing over-analysis.


## 📜 Citation

If you find this work useful, please consider citing:

wait updating...


## 📝 License

This project is licensed under the [MIT License](LICENSE).

<!-- ## 🙏 Acknowledgements
We thank all researchers and developers who contributed to this project. Special thanks to [Institution/Organization Name] for their support. -->
