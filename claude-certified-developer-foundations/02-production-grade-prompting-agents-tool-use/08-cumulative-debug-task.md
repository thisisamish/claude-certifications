# Cumulative Debug Task

The agent implementation below has four planted bugs, one in each of four layers: the schema layer, the streaming layer where the response is assembled and committed, the context layer where the message structure is built, and the memory layer.

Work through the two stages below. This screen covers Stage 1: identify each bug. Stage 2, writing the corrected version, is on the next screen.

### Buggy implementation

```python
# --- TOOL DEFINITIONS ---
tools = [
  {
    "name": "get_customer_data",
    "description": "Gets data.",
    "input_schema": { "type": "object", "properties": { "id": {"type":"string"} }, "required": ["id"] }
  }
]

# --- AGENT LOOP ---
def run_agent(user_request, session_history):
  messages = session_history + [{"role":"user","content":user_request}]
  while True:
    blocks = {}
    stop_seen = False
    with client.messages.stream(
        model=model, max_tokens=4096, tools=tools, messages=messages,
        thinking={"type": "adaptive"}
    ) as stream:
      for event in stream:
        if event.type == "content_block_start":
          blocks[event.index] = init_block(event)
        elif event.type == "content_block_delta":
          apply_delta(blocks[event.index], event.delta)
        elif event.type == "message_stop":
          stop_seen = True
    assistant_content = [b for b in assemble(blocks) if b["type"] != "thinking"]
    messages.append({"role": "assistant", "content": assistant_content})
    response = finalize(blocks)
    if response.stop_reason == "end_turn":
      return response
    for block in response.content:
      if block.type == "tool_use":
        result = execute_tool(block.name, block.input)
        messages.append({"role":"user","content":[{"type":"tool_result",
                         "tool_use_id":block.id,"content":result}]})

# --- MEMORY ---
def build_session_history(prior_sessions):
  # Concatenating all prior session transcripts in-context
  full_history = []
  for session in prior_sessions:
    full_history.extend(session["messages"])
  return full_history
```

## Stage 1: Identify each bug

The implementation above has four bugs, one in each of four layers. For each bug: name the layer it belongs to and write one sentence describing what it causes at runtime.

### Answer on the Course Page

- **Bug 1 (vague description "Gets data."):** _Schema layer:_ Claude cannot distinguish this tool from any other retrieval tool and selects on surface-level matching rather than intent.

- **Bug 2 (turn committed before message_stop; thinking block stripped):** _Streaming layer:_ An interrupted stream writes a partial, possibly half-built tool_use, block into history; the stripped thinking block breaks the carry-back rule and the API rejects the next request because the signature no longer matches.

- **Bug 3 (only tool_result appended; no preceding assistant tool_use turn):** _Context layer:_ The API sees a tool_result referencing a tool_use block it never received as a complete assistant turn and rejects the request.

- **Bug 4 (all prior session transcripts concatenated in-context):** _Memory layer:_ The context window grows with every session and fills before the agent can process the current request by session four or five.

## Stage 2: write the corrected version of each bug identified on the previous screen. For each one, show the fixed code and name what it changes.

### Buggy implementation (for reference)

```python
# --- TOOL DEFINITIONS ---
tools = [
  {
    "name": "get_customer_data",
    "description": "Gets data.",
    "input_schema": { "type": "object", "properties": { "id": {"type":"string"} }, "required": ["id"] }
  }
]

# --- AGENT LOOP ---
def run_agent(user_request, session_history):
  messages = session_history + [{"role":"user","content":user_request}]
  while True:
    blocks = {}
    stop_seen = False
    with client.messages.stream(
        model=model, max_tokens=4096, tools=tools, messages=messages,
        thinking={"type": "adaptive"}
    ) as stream:
      for event in stream:
        if event.type == "content_block_start":
          blocks[event.index] = init_block(event)
        elif event.type == "content_block_delta":
          apply_delta(blocks[event.index], event.delta)
        elif event.type == "message_stop":
          stop_seen = True
    assistant_content = [b for b in assemble(blocks) if b["type"] != "thinking"]
    messages.append({"role": "assistant", "content": assistant_content})
    response = finalize(blocks)
    if response.stop_reason == "end_turn":
      return response
    for block in response.content:
      if block.type == "tool_use":
        result = execute_tool(block.name, block.input)
        messages.append({"role":"user","content":[{"type":"tool_result",
                         "tool_use_id":block.id,"content":result}]})

# --- MEMORY ---
def build_session_history(prior_sessions):
  # Concatenating all prior session transcripts in-context
  full_history = []
  for session in prior_sessions:
    full_history.extend(session["messages"])
  return full_history
```

### My Answer

```python
# --- TOOL DEFINITIONS ---
# Adding description to the tool and the required parameter
tools = [
  {
    "name": "get_customer_data",
    "description": "Gets customer data by Customer Full Name. Returns Customer ID, Age, Gender, Address, and a list of orders. Call this tool when you need the above details and you have customer's full name. Do not call this tool if you have the above mentioned details in the context for the particular customer.",
    "input_schema": { "type": "object", "properties": { "id": {"type":"string", "description": "full name of the customer"} }, "required": ["id"] }
  }
]

# --- AGENT LOOP ---
def run_agent(user_request, session_history):
  messages = session_history + [{"role":"user","content":user_request}]
  while True:
    blocks = {}
    stop_seen = False
    with client.messages.stream(
        model=model, max_tokens=4096, tools=tools, messages=messages,
        thinking={"type": "adaptive"}
    ) as stream:
      for event in stream:
        if event.type == "content_block_start":
          blocks[event.index] = init_block(event)
        elif event.type == "content_block_delta":
          apply_delta(blocks[event.index], event.delta)
        elif event.type == "message_stop":
          stop_seen = True
    # Adding check so new blocks are only added to the messages when we are sure that entire block has been received
    if stop_seen:
        # Remove the check that discards the "thinking" blocks
        assistant_content = [b for b in assemble(blocks)]
        messages.append({"role": "assistant", "content": assistant_content})
        response = finalize(blocks)
        if response.stop_reason == "end_turn":
            return response
        elif response.stop_reason == "max_tokens":
            return "Max tokens reached. Trim the context before proceeding further."
        tool_result_blocks = []
        for block in response.content:
            if block.type == "tool_use":
                result = execute_tool(block.name, block.input)
                # A single request can have multiple tool calls. They should go in the same user message
                tool_result_blocks.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })
        if tool_result_blocks:
            messages.append({
                "role":"user",
                "content": tool_result_blocks,
            })

# --- MEMORY ---
# A simplified lookup function that only adds to the context when needed
def build_session_history(prior_sessions, content_to_match):
  # Concatenating all prior session transcripts in-context
  add_to_context = []
  for session in prior_sessions:
    words = session["messages"]
    for index, word in enumerate(words):
        if word == content_to_match:
            start = max(0, index - 10)
            end = min(len(words), index + 11)
            add_to_context.extend(words[start:end])
  return add_to_context
```

### Answer on the Course Page

- **Bug 1 fix:** _Schema layer:_ Replace the description with one that states intent and an exclusion:
  "Use this to retrieve full account and contact details for a customer by customer ID. Do not use this for order history or transaction records."
- **Bug 2 fix:** _Streaming layer:_ Keep all blocks including the thinking block. Gate the commit on stop_seen and raise on interruption:
  ```python
  assistant_content = assemble(blocks)	# keep all blocks including thinking
  if stop_seen:
      messages.append({"role": "assistant", "content": assistant_content})
  else:
      raise StreamInterruptedError(
          "Discarding partial turn; retry from last complete turn."
      )
  ```
- **Bug 3 fix:** _Context layer:_ Bug 3 is resolved once Bug 2 is fixed. The full assistant turn, including the tool_use block, is now appended before the tool_result, satisfying the pairing rule.
- **Bug 4 fix:** _Memory layer:_ Use external storage and inject only a summary at session start rather than concatenating full transcripts:
  ```python
  def build_session_history(prior_sessions):
      if not prior_sessions:
          return []
      summary = load_session_summary(prior_sessions[-1]["id"]) # from external store
      return [{"role": "user", "content": f"Session context: {summary}"}]
  ```
