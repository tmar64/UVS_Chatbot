# creates fine-tuning job

```ruby
import openai
client = openai.OpenAI(api_key='<api-key>')

response = client.fine_tuning.jobs.create(
  training_file="<file-id>",
  model="<desired-model>"
)

job_id = response.id
print(f"Job successfully started: {job_id}")
```
