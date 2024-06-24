# check fine-tuning job status

```ruby
import openai
client = openai.OpenAI(api_key='<api-key>')

job_status = client.fine_tuning.jobs.list(limit=20)
print(job_status)
```
