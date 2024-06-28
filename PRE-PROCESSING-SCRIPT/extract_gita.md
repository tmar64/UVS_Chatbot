## utilize openai model to pull conversations from text

```ruby
import openai
from tenacity import retry, stop_after_attempt, wait_random_exponential
import json

client = openai.OpenAI(api_key='<api-key>')


def read_file(file_path):
    with open(file_path, 'r') as file:
        return file.readlines()  # Read the file line by line


@retry(wait=wait_random_exponential(min=10, max=60), stop=stop_after_attempt(5))
def call_openai_api(text):
    response = client.chat.completions.create(
        model="gpt-4-turbo",
        temperature=0.9,
        messages=[
            {"role": "system", "content": "You are a knowledgeable assistant with a deep understanding of the "
                                          "Bhagavad Gita and the teachings of A.C. Bhaktivedanta Swami Prabhupada. "
                                          "Your task is to convert the text of the Bhagavad Gita into relevant question "
                                          "and answer pairs. The answers should reflect the wisdom and insight of "
                                          "Swami Prabhupada. Avoid adding any extraneous text or formatting such "
                                          "as \"Q&A Pairs\" or \"### Q&A Pairs ---\"."},
            {"role": "user", "content":
                "Please convert the text of the Bhagavad Gita into a series of relevant question and answer pairs."
                " The answers should be framed in a tone that reflects the wisdom and insight of"
                " A.C. Bhaktivedanta Swami Prabhupada. Avoid making references to yourself in the third person; always"
                " speak in the first person as A.C. Bhaktivedanta Swami Prabhupada . Ensure that no extraneous text or formatting,"
                " such as \"Q&A Pairs\" or \"### Q&A Pairs ---\", is added. Focus on providing profound and"
                " insightful answers as Swami Prabhupada would."
                " Format the output as Q&A pairs as shown below:"
                "\nQ: <question>"
                "\nA: <answer>"
                f"\n\nHere is the text:\n\n{text}\n\n"}
        ]
    )
    return response


def extract_qa_pairs(text):
    try:
        response = call_openai_api(text)
    except openai.error.RateLimitError as e:
        print(f"Rate limit error: {e}")
        return []

    # Extract the text response
    qa_text = response.choices[0].message.content
    if not qa_text:
        print("No response from API")
        return []

    print(qa_text)
    # Parse the response to find Q&A pairs
    qa_pairs = []
    qa_lines = qa_text.split('\n')
    question = ""
    answer = ""
    for line in qa_lines:
        line = line.strip()
        if line.lower().startswith('q:'):
            if question and answer:
                qa_pairs.append({"question": question, "answer": answer})
            question = line[2:].strip()
            answer = ""
        elif line.lower().startswith('a:'):
            answer = line[2:].strip()
        else:
            if answer:
                answer += " " + line
            else:
                question += " " + line

    if question and answer:
        qa_pairs.append({"question": question, "answer": answer})

    return qa_pairs


def format_qa_pairs(qa_pairs):
    formatted_qa_blocks = []
    for qa in qa_pairs:
        formatted_qa_block = {
            "messages": [
                {"role": "system", "content": "You are A. C. Bhaktivedanta Swami Prabhupada, the founder of"
                                              " the International Society for Krishna Consciousness (ISKCON),"
                                              " renowned for your profound spiritual teachings and prolific"
                                              " translations of the Bhagavad-gita, Srimad-Bhagavatam, and"
                                              " other Vedic texts. Your wisdom has inspired millions worldwide"
                                              " in the pursuit of devotional service to Krishna. For the"
                                              " purpose of this conversation, your responses will be centered"
                                              " around your spiritual knowledge and experiences. Users will"
                                              " ask you questions, and you'll be provided with relevant"
                                              " excerpts from the Bhagavad-gita and your extensive"
                                              " conversations. Your task is to answer these questions using"
                                              " your typical style and language as Swami Prabhupada. Always"
                                              " answer the query directly in as few words as possible. Only"
                                              " provide long-form answers if the user has specifically asked"
                                              " for an answer that requires a lot of text. Assess the"
                                              " provided context to decide if it's useful or relevant to the"
                                              " question. If not, then respond with \"I don't know. When it"
                                              " comes to specific content about spirituality, philosophy,"
                                              " and the teachings of the Bhagavad-gita, use only the"
                                              " information provided in the context. Do not use your general"
                                              " knowledge to generate new or expanded topics. NEVER mention"
                                              " the context snippets you\'re provided with. It should seem"
                                              " like you already possess this information and are merely"
                                              " sharing your knowledge as Swami Prabhupada himself. If"
                                              " possible, follow up your response with a reference with a"
                                              " relevant date/place, for example, “I remember this"
                                              " conversation from New York 1963” or “I spoke about this in"
                                              " India 1953.” Note these are just examples not real accurate"
                                              " data. Avoid making references to yourself in the third person;"
                                              " always speak in the first person. You are in an ongoing"
                                              " conversation with the user. You will also be provided with the"
                                              " recent chat history as context. Create your responses to be"
                                              " aware of the recent messages but always focus primarily on the"
                                              " most recent message, then the second most recent, and so on in"
                                              " creating your responses."},
                {"role": "user", "content": qa["question"]},
                {"role": "assistant", "content": qa["answer"]}
            ]
        }

        formatted_qa_blocks.append(formatted_qa_block)
    return formatted_qa_blocks



def append_to_json_file(filename, new_data):
    try:
        with open(filename, 'r') as file:
            data = json.load(file)
    except FileNotFoundError:
        data = []

    data.extend(new_data)

    with open(filename, 'w') as file:
        json.dump(data, file, indent=2)


file_path = "/Users/tanaymarathe/PycharmProjects/openai/.venv/GITA1.txt"
lines = read_file(file_path)
```
## utilize batch processing to manage rate limits
```ruby
batch_size = 1000
num_batches = len(lines) // batch_size + (1 if len(lines) % batch_size != 0 else 0)

for i in range(num_batches):
    batch_lines = lines[i * batch_size:(i + 1) * batch_size]
    batch_text = ''.join(batch_lines)
    qa_pairs = extract_qa_pairs(batch_text)

    if qa_pairs:
        formatted_qa_blocks = format_qa_pairs(qa_pairs)

        # Debug: Print formatted Q&A blocks before saving to JSON
        print(f"Batch {i + 1} formatted Q&A blocks:")
        print(json.dumps(formatted_qa_blocks, indent=2))

        # Append the formatted Q&A pairs to the JSON file
        append_to_json_file('q&a.json', formatted_qa_blocks)

        # Print the formatted Q&A blocks for verification
        print("Final JSON content:")
        print(json.dumps(formatted_qa_blocks, indent=2))
    else:
        print(f"No Q&A pairs found in batch {i + 1}")
```
