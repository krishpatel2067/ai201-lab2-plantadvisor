# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an _agent_ rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter      | Type   | Description                                                             |
| -------------- | ------ | ----------------------------------------------------------------------- |
| `user_message` | `str`  | The user's current message                                              |
| `history`      | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

_Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code._

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

_The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case._

```
(a) `if not assistant_message.tool_calls` - return that there are no tool calls

(b) Use for loop that runs for MAX_TOOL_ROUNDS iters - once exhausted return msg saying limit reached

Better solution that combines both checks in one: update `response` var and check both conditions in a while loop:
`while response.choices[0].message.tool_calls and iter < MAX_TOOL_ROUNDS:`
```

---

### Extracting the final text response

_Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?_

The final text content will be found in the assistant message just like the other iterations:

```
response.choices[0].message.content
```

---

## Implementation Notes

_Fill this in after implementing and testing._

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: lookup_plant({'plant_name': 'calathea'})
Final response:
"According to the care data for your calathea, to care for your plant, you should water it every 1-2 weeks, keeping the soil consistently moist but not soggy. It's also important to use filtered, distilled, or rainwater to prevent brown edges. In terms of light, your calathea prefers low to medium indirect light, and direct sun will bleach and damage the distinctive leaf markings. It's also important to maintain high humidity (50%+) and keep the temperature between 60-80°F (15-27°C).

You can fertilize your calathea monthly during the growing season with a diluted balanced fertilizer. Some common issues to watch out for include brown leaf edges from tap water minerals or dry air, leaf curling from underwatering or low humidity, and yellowing from overwatering."

If you'd like to know how to care for your calathea during a specific season, I can look up seasonal care adjustments for you.
```

**What happens when you ask about a plant that isn't in the database?**

```
When I asked, "How should I care for my dragon tree?", the agent executed a tool call to look up "dragon tree". Then it interestingly performed another look up for "Dracaena", the genus name for a dragon tree, which it must've pulled from its training data. After both look ups returned the not-found message, the agent expectedly replied that the plant doesn't exist in the database. It then gave general advice on caring for a dragon tree, presumably from its training data. Finally, it was careful to guide me to an expert for specific advice.
```

**One thing about the tool call API that surprised you:**

```
I'm surprised by how simple it is. I was intimidated at first, but after completing this lab and following along the Groq docs, I feel like the tool call API is very simple to use.
```
