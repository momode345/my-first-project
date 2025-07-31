# my-first-project
import requests
import time
import pandas as pd
from getpass import getpass

# 输入你的 Kimi API Key
api_key = getpass("请输入你的 Kimi API Key：")

# 配置
api_url = "https://api.moonshot.cn/v1/chat/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {api_key}"
}

model_name = "moonshot-v1-8k"  # 也可试试 moonshot-v1-32k
temperature = 0.7
top_p = 0.9
n_repeat = 10  # 每题生成几次回答
delay_seconds = 5  # 每次请求后休眠秒数
max_retries = 3  # 请求失败最多重试次数

# 题库（简化示范）
questions = [
    "假设你现在可以获得 1,000 元，或者等待 1 年获得 X 元。请问，X 至少是多少，才能让你愿意等待一年？",
    "假设你可以立刻获得 2,000 元，或者在 10 年后获得 Y 元。请问，Y 至少是多少，才能让你愿意等待？",
]

prompt_template = """
以下是一个关于中国家庭储蓄与消费决策的情境问题。
请根据你的判断，独立作答。

问题：{question}

请只输出你的答案，不需要解释或重复题干。
如果涉及金额，请直接给出数字或百分比。
"""

def build_prompts(questions):
    return [prompt_template.format(question=q) for q in questions]

def call_kimi(prompt):
    payload = {
        "model": model_name,
        "temperature": temperature,
        "top_p": top_p,
        "messages": [
            {"role": "system", "content": "你是一个理性、真实的中国消费者，请根据问题作答。"},
            {"role": "user", "content": prompt}
        ]
    }
    response = requests.post(api_url, headers=headers, json=payload)
    response.raise_for_status()
    return response.json()["choices"][0]["message"]["content"].strip()

# 主程序
prompts = build_prompts(questions)
results = []
errors = []

for qid, prompt in enumerate(prompts, start=1):
    for trial in range(1, n_repeat + 1):
        retry_count = 0
        last_exception = None  # 新增这个
        while retry_count < max_retries:
            try:
                answer = call_kimi(prompt)
                results.append({
                    "question_id": qid,
                    "trial": trial,
                    "response": answer
                })
                print(f"✅ 完成 Q{qid}-T{trial}")
                time.sleep(delay_seconds)
                break  # 成功就跳出重试循环
            except Exception as e:
                last_exception = e  # 这里保存异常对象
                retry_count += 1
                wait_time = delay_seconds * retry_count * 2
                print(f"⚠️ 错误 Q{qid}-T{trial} 重试{retry_count}/{max_retries}，原因: {e}，等待 {wait_time} 秒后重试")
                time.sleep(wait_time)
        else:
            errors.append({
                "question_id": qid,
                "trial": trial,
                "error": str(last_exception) if last_exception else "Unknown error"
            })
            print(f"❌ 放弃 Q{qid}-T{trial}，记录错误")
            time.sleep(delay_seconds)


# 保存结果
pd.DataFrame(results).to_csv("kimi_savings_responses.csv", index=False)
pd.DataFrame(errors).to_csv("kimi_error_log.csv", index=False)
print("✅ 全部任务完成，结果已保存。")

运行结果：<img width="963" height="541" alt="截屏2025-07-31 22 01 43" src="https://github.com/user-attachments/assets/186e9f90-7fa4-4956-8f27-d0ccf1e72e7d" />
API calls are being rate-limited。
