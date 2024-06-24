## uploads file for fine-tuning

```ruby
import openai


# Reads the input file then returns it as a string
def open_file(filepath):
    with(open(filepath, 'r', encoding='utf-8')) as infile:
        return infile.read()


# Function to save content to a file
def save_file(filepath, content):
    with(open(filepath, 'w', encoding='utf-8')) as outfile:
        outfile.write(content)


# Sets the API key
client = openai.OpenAI(api_key='<api-key>')

# Uploads the JSONL file
with open('<file-path>', "rb") as file:
    response = client.files.create(
        file=file,
        purpose='fine-tune'
    )

file_id = response.id

print(f"File successfully uploaded: {file_id} ")
```
