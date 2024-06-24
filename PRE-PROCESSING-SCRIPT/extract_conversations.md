## utilize openai model to pull conversations from text<br/>
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
        model="gpt-4o-2024-05-13",
        temperature=0.9,
        messages=[
            {"role": "user", "content":
                "Step 1: The attached text contains conversations between Prabhupāda and various other people."
                " Each block of conversations has a location and date associated with it."
                " Step 2: For each block of conversation, extract all the statements/questions posed to Prabhupāda, along with Prabhupāda's responses."
                " Step 3: Append the date/location to the end of Prabhupāda's responses."
                " For example, append 'I mentioned this teaching in [Date, Location]' to the end of each of Prabhupāda's responses."
                " Step 4: Ensure the context is maintained by keeping relevant surrounding sentences when necessary."
                " Step 5: Format the output as Q&A pairs in the following format:"
                "\nQ: <statement/question>"
                "\nA: <Prabhupāda's response> (Date, Location)"
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
            "prompt": qa["question"], "completion": qa["answer"],
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


file_path = "<file-path>"
lines = read_file(file_path)
```
## batch processing script to manage rate limits
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
        append_to_json_file('conversations.json', formatted_qa_blocks)

        # Print the formatted Q&A blocks for verification
        print("Final JSON content:")
        print(json.dumps(formatted_qa_blocks, indent=2))
    else:
        print(f"No Q&A pairs found in batch {i + 1}")
```
