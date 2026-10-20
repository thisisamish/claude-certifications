# Streaming Responses

In a non-streamed request, the API hands you one complete message with every content block, fully formed. In a streamed request, the API instead sends a series of events that describe the message as it's being built. Your code listens to that series and reassembles the blocks. The message you end up with is identical to what a non-streamed call would have given you, but the difference is that you have to assemble the pieces, and you decide what to do if the events stop before the message is finished.

It helps to know what's not happening: the model isn't holding some live object open for you. Each event is its own small message describing a single change, a block started, some text or input got added to it, a block finished, the whole message finished. Your handler takes each event and applies it to the partial state it's been building up.

## The event sequence, and what your handler does with each

| Event               | What it signals                                                                                                       | What your handler does                                                                                                                                                      |
| ------------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| message_start       | A new message is beginning. Carries the message shell with empty content and initial usage.                           | Set up an empty content array to collect blocks in.                                                                                                                         |
| content_block_start | A new content block is opening, with its type (text, tool_use, or thinking) and index.                                | Make a slot at that index for the named block type. A tool_use block opens with its name and id, but no input yet.                                                          |
| content_block_delta | An incremental piece of one block: a text fragment, a fragment of JSON input for a tool call, or a thinking fragment. | Append the fragment to the block at that index. Tool-call inputs arrive as a partial JSON string spread across several deltas, you can't parse them until the block closes. |
| content_block_stop  | The block at this index is complete.                                                                                  | Finalize the block. For a tool_use block, this is the first moment the accumulated JSON input is complete enough to parse.                                                  |
| message_delta       | Top-level changes to the message: the stop_reason and final usage counts.                                             | Record the stop_reason. It tells you whether the model finished or stopped for some other reason.                                                                           |
| message_stop        | The stream is complete.                                                                                               | The assembled content array is now the finished message. From here, treat it exactly like a non-streamed response.                                                          |

## The rule that keeps your state from getting corrupted: don't act on a partial block

The tool_use block is the one to watch. Its input shows up as a partial JSON string spread across many content_block_delta events, and that string isn't valid JSON until content_block_stop closes the block. If your code tries to parse the input or run the tool before the block closes, it either chokes on malformed JSON or runs with half the arguments missing. So, the rule is simple: collect the deltas, and act only after content_block_stop for that block.

The same discipline applies when you add a streamed assistant turn to your conversation history. Add it only after message_stop, with every block fully assembled. A turn built from a stream that got cut off partway is incomplete, and the tool_use pairing rules will reject your next request if a half-built tool_use block ends up in the history.

## When the stream stops early

Streams sometimes fail in the middle. A dropped network connection, a timeout, or a client disconnect can end the event series before message_stop arrives. The failure that really bites is treating whatever you've collected so far as if it were complete. A partial text block shown to a user is just a cosmetic glitch and a partial tool_use block written into history is a structural problem that corrupts the next turn.

- **Track completion on purpose.** A turn is usable only once message_stop has arrived. Until then, treat what you've accumulated as provisional.
- **On an interrupted stream, throw away the partial assistant turn** instead of saving it to history, then retry the request. Committing a half-built turn is exactly what breaks the following request.
- **Check the stop_reason from message_delta** before you continue a loop. A stop_reason of tool_use means your assembled tool calls are ready to run; any other value means you're on a different path, not the tool path.
