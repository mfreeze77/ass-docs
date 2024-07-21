# ass-docs
OpenAI Assistant API Docs
GPT Assistants API - README
Introduction
The GPT Assistants API is designed to help developers build powerful AI assistants capable of performing a variety of tasks. This API is currently in beta, and we are actively working on adding more functionality. Your feedback is crucial, so please share it in our Developer Forum!

Key Features
Assistants
Custom Instructions: Assistants can call OpenAI’s models with specific instructions to tune their personality and capabilities.
Parallel Tool Access: Assistants can access multiple tools in parallel, including both OpenAI-hosted tools and custom tools via function calling.
Persistent Threads: Threads store message history and handle truncation to fit within the model’s context length. Assistants append messages to threads as users reply.
File Management: Assistants can access files in several formats and create files (e.g., images, spreadsheets). They can also cite files in their messages.
Objects
Assistant
A purpose-built AI that uses OpenAI’s models and calls tools.

Thread
A conversation session between an Assistant and a user, storing messages and handling truncation.

Message
A message created by an Assistant or a user, stored as a list on the Thread.

Run
An invocation of an Assistant on a Thread, performing tasks by calling models and tools.

Run Step
A detailed list of steps taken by the Assistant during a Run.

Creating Assistants
To get started, create an Assistant by specifying the model to use. You can further customize the Assistant's behavior:

Instructions: Guide the personality of the Assistant and define its goals.
Tools: Provide access to up to 128 tools, including OpenAI-hosted tools like code_interpreter and file_search, or third-party tools via function calling.
Tool Resources: Provide files as resources to tools like code_interpreter and file_search.
Example:

python
Copy code
file = client.files.create(
  file=open("revenue-forecast.csv", "rb"),
  purpose='assistants'
)

assistant = client.beta.assistants.create(
  name="Data visualizer",
  description="You are great at creating beautiful data visualizations. You analyze data present in .csv files, understand trends, and come up with data visualizations relevant to those trends. You also share a brief text summary of the trends observed.",
  model="gpt-4o",
  tools=[{"type": "code_interpreter"}],
  tool_resources={
    "code_interpreter": {
      "file_ids": [file.id]
    }
  }
)
Managing Threads and Messages
Threads and Messages represent a conversation session between an Assistant and a user. There is no limit to the number of Messages you can store in a Thread. Threads smartly truncate messages once they exceed the context window of the model.

Creating a Thread:

python
Copy code
thread = client.beta.threads.create(
  messages=[
    {
      "role": "user",
      "content": "Create 3 data visualizations based on the trends in this file.",
      "attachments": [
        {
          "file_id": file.id,
          "tools": [{"type": "code_interpreter"}]
        }
      ]
    }
  ]
)
Context Window Management
The Assistants API automatically manages truncation to ensure it stays within the model's maximum context length. Customize this behavior by specifying maximum tokens for a run and/or the maximum number of recent messages to include in a run.

Max Completion and Max Prompt Tokens
Control token usage in a single Run by setting max_prompt_tokens and max_completion_tokens.

Truncation Strategy
Specify a truncation strategy to control how your thread should be rendered into the model's context window.

Run Lifecycle
Run objects can have multiple statuses:

queued: When Runs are first created or when you complete the required_action.
in_progress: While the Assistant uses the model and tools to perform steps.
completed: The Run successfully completed.
requires_action: When the model determines the functions to be called, requiring user action.
expired: When function calling outputs were not submitted before expires_at.
cancelling: Attempting to cancel an in_progress run.
cancelled: Successfully cancelled.
failed: Viewing the reason for the failure in the last_error object.
incomplete: Ended due to max_prompt_tokens or max_completion_tokens reached.
Polling for Updates
Periodically retrieve the Run object to keep the status of your run up to date.

Thread Locks
When a Run is in_progress and not in a terminal state, the Thread is locked.

Data Access Guidance
Ensure end-user authorization before performing reads or writes on Assistants, Threads, Messages, and Vector Stores. Implement strict API key access controls and consider creating separate accounts for different applications.

Next Steps
Explore Assistant Tools, which cover topics like Function calling, File Search, and Code Interpreter.

File Search Beta
The file_search tool augments the Assistant with knowledge from outside its model. OpenAI parses and chunks your documents, creates and stores embeddings, and uses both vector and keyword search to retrieve relevant content.

Quickstart
Create an Assistant with File Search enabled:

python
Copy code
assistant = client.beta.assistants.create(
  name="Financial Analyst Assistant",
  instructions="You are an expert financial analyst. Use your knowledge base to answer questions about audited financial statements.",
  model="gpt-4o",
  tools=[{"type": "file_search"}],
)
Upload files and add them to a Vector Store:

python
Copy code
vector_store = client.beta.vector_stores.create(name="Financial Statements")

file_paths = ["edgar/goog-10k.pdf", "edgar/brka-10k.txt"]
file_streams = [open(path, "rb") for path in file_paths]

file_batch = client.beta.vector_stores.file_batches.upload_and_poll(
  vector_store_id=vector_store.id, files=file_streams
)

assistant = client.beta.assistants.update(
  assistant_id=assistant.id,
  tool_resources={"file_search": {"vector_store_ids": [vector_store.id]}},
)
Create a thread and attach files as Message attachments:

python
Copy code
message_file = client.files.create(
  file=open("edgar/aapl-10k.pdf", "rb"), purpose="assistants"
)

thread = client.beta.threads.create(
  messages=[
    {
      "role": "user",
      "content": "How many shares of AAPL were outstanding at the end of of October 2023?",
      "attachments": [
        { "file_id": message_file.id, "tools": [{"type": "file_search"}] }
      ],
    }
  ]
)
Create a run and check the output:

python
Copy code
with client.beta.threads.runs.stream(
    thread_id=thread.id,
    assistant_id=assistant.id,
    instructions="Please address the user as Jane Doe. The user has a premium account.",
    event_handler=EventHandler(),
) as stream:
    stream.until_done()
Customizing File Search Settings
Customize how the file_search tool chunks your data and how many chunks it returns to the model context.

Code Interpreter Beta
The Code Interpreter allows Assistants to write and run Python code in a sandboxed execution environment. It can process files with diverse data and formatting and generate files with data and images of graphs.

Enabling Code Interpreter
Enable Code Interpreter by passing code_interpreter in the tools parameter of the Assistant object:

python
Copy code
assistant = client.beta.assistants.create(
  instructions="You are a personal math tutor. When asked a math question, write and run code to answer the question.",
  model="gpt-4o",
  tools=[{"type": "code_interpreter"}]
)
Passing Files to Code Interpreter
Files can be passed at both the Assistant and Thread levels. Upload a file and create an Assistant:

python
Copy code
file = client.files.create(
  file=open("mydata.csv", "rb"),
  purpose='assistants'
)

assistant = client.beta.assistants.create(
  instructions="You are a personal math tutor. When asked a math question, write and run code to answer the question.",
  model="gpt-4o",
  tools=[{"type": "code_interpreter"}],
  tool_resources={
    "code_interpreter": {
      "file_ids": [file.id]
    }
  }
)
Files can also be passed at the Thread level:

python
Copy code
thread = client.beta.threads.create(
  messages=[
    {
      "role": "user",
      "content": "I need to solve the equation `3x + 11 = 14`. Can you help me?",
      "attachments": [
        {
          "file_id": file.id,
          "tools": [{"type": "code_interpreter"}]
        }
      ]
    }
  ]
)
Reading Images and Files Generated by Code Interpreter
Download generated files by passing the file ID to the Files API:

python
Copy code
image_data = client.files.content("file-abc123")
image_data_bytes = image_data.read()

with open("./my-image.png", "wb") as file:
    file.write(image_data_bytes)
Input and Output Logs of Code Interpreter
Inspect the code input and outputs logs of Code Interpreter by listing the steps of a Run that called Code Interpreter:

python
Copy code
run_steps = client.beta.threads.runs.steps.list(
  thread_id=thread.id,
  run_id=run.id
)
Function Calling Beta
Define functions under the tools param of the assistant to enable function calling:

python
Copy code
assistant = client.beta.assistants.create(
  instructions="You are a weather bot. Use the provided functions to answer questions.",
  model="gpt-4o",
  tools=[
    {
      "type": "function",
      "function": {
        "name": "get_current_temperature",
        "description": "Get the current temperature for a specific location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "The city and state, e.g., San Francisco, CA"
            },
            "unit": {
              "type": "string",
              "enum": ["Celsius", "Fahrenheit"],
              "description": "The temperature unit to use. Infer this from the user's location."
            }
          },
          "required": ["location", "unit"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "get_rain_probability",
        "description": "Get the probability of rain for a specific location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "The city and state, e.g., San Francisco, CA"
            }
          },
          "required": ["location"]
        }
      }
    }
  ]
)
Create a Thread and add Messages:

python
Copy code
thread = client.beta.threads.create()
message = client.beta.threads.messages.create(
  thread_id=thread.id,
  role="user",
  content="What's the weather in San Francisco today and the likelihood it'll rain?",
)
Initiate a Run and submit tool outputs:

python
Copy code
with client.beta.threads.runs.stream(
  thread_id=thread.id,
  assistant_id=assistant.id,
  event_handler=EventHandler()
) as stream:
  stream.until_done()
What's New in v2? Beta
Improved Retrieval Tool: file_search can now ingest up to 10,000 files per assistant.
Vector Store Objects: Simplified file management and billing.
Token Usage Control: Manage token usage costs.
New Streaming and Polling Helpers: Available in our Node and Python SDKs.
Fine-Tuned Models: Support for fine-tuned versions of gpt-3.5-turbo-0125.
Assistant API Streaming Support: Now available for better real-time interaction.
Supported File Formats
FILE FORMAT	MIME TYPE
.c	text/x-c
.cs	text/x-csharp
.cpp	text/x-c++
.doc	application/msword
.docx	application/vnd.openxmlformats-officedocument.wordprocessingml.document
.html	text/html
.java	text/x-java
.json	application/json
.md	text/markdown
.pdf	application/pdf
.php	text/x-php
.pptx	application/vnd.openxmlformats-officedocument.presentationml.presentation
.py	text/x-python
.py	text/x-script.python
.rb	text/x-ruby
.tex	text/x-tex
.txt	text/plain
.css	text/css
.js	text/javascript
.sh	application/x-sh
.ts	application/typescript
.csv	application/csv
.jpeg	image/jpeg
.jpg	image/jpeg
.gif	image/gif
.png	image/png
.tar	application/x-tar
.xlsx	application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
.xml	application/xml or text/xml
.zip	application/zip
Conclusion
The GPT Assistants API offers a powerful set of tools and capabilities to build AI assistants tailored to your needs. Whether it's creating data visualizations, answering financial queries, or interpreting code, the API provides the flexibility and power to get the job done. Explore the API, provide feedback, and join us in shaping the future of AI assistants.
