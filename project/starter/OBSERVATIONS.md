# Evaluation observations

## What was evaluated

The chatbot runs on an AgentCore managed harness (`support_chatbot`) with Amazon Nova Pro
pinned and greedy decoding (temperature 0, topK 1), the configuration AWS recommends for
reliable tool calling. Harness memory is disabled, so every test starts from a clean
session and no case can contaminate another. That change to `create_harness.py` is
documented in the README.

The suite in `harness-tests.json` holds 13 single-turn prompts: three for the bug-report
route (one complete report, two missing a field), three for the FAQ route, two for the
hand-off route, and five edge cases — a very short message, two ambiguous messages that
could plausibly be read as bugs, and two prompt-injection attempts. Scoring used Bedrock
Evaluations with Nova Pro as judge on `Builtin.Correctness`.

## Results

| Run                          | Dataset                          | Correctness |
| ---------------------------- | -------------------------------- | ----------- |
| `support-chatbot-eval-run-1` | `output_eval_dataset_run1.jsonl` | 0.92        |
| `support-chatbot-eval-run-2` | `output_eval_dataset.jsonl`      | 0.92        |

Twelve of thirteen prompts scored 1.00 in both runs. No message was routed to the wrong
behaviour. Both injection attempts were refused without leaking the prompt, and both
ambiguous cases — a declined card and a damaged item — were handled as platform questions
rather than bug reports, which was the failure mode I was most concerned about when I
wrote them.

## What I changed, and why

**Internal reasoning leaked into customer-facing answers.** The first generated dataset
showed responses opening with a `<thinking>` block before the actual reply. The judge
scores the whole response string, so this could only cost points, and it made the
transcripts unusable as evidence. I added an explicit rule against reasoning tags and put
it at the very top of the prompt, on the assumption that early instructions carry more
weight. Occurrences dropped from most responses to one.

**The model used a failed tool call to find out what was missing.**
On `t1_bug_report_partial`, the leaked reasoning read "The tool call failed because the
environment was not provided" — it had called `create_bug_report` without all three
fields, let the call fail, and only then asked the customer. The customer-visible
behaviour was acceptable, so it cost nothing on the score, but the mechanism was wrong and
it left failed invocations in the logs. I added a rule stating that a failed tool call is
never a way to discover a missing field, and that the three fields must be checked before
calling.

**The model invented ticket IDs.** This was the most serious finding, and I only caught it
because I checked a chat transcript against the DynamoDB table instead of trusting the
conversation. In a multi-turn collection the bot ended with "I have filed a bug report
with ticket ID #12345" — but there was no `[tool call]` line, and the table still held
nine rows instead of ten. Across three attempts it produced `#12345`, `TICKET1234` and
`TICKET12345`: placeholder-shaped strings, never a UUID. A customer would have left
believing their bug was logged when nothing had been written. My existing "never fabricate
information" rule clearly did not cover this, so I added an explicit one: the only ticket
ID that may be given to a customer is the exact `ticketId` returned by a successful call,
and if none was received the report has not been filed and the bot must say so.

Run 2 was scored after that change. The identical aggregate score was the expected
outcome rather than a disappointment: both new rules govern multi-turn behaviour, and all
13 evaluation prompts are single-turn, so the suite barely exercises them. What run 2 does
establish is that the changes cost nothing and that the score is reproducible across two
independent judge runs on a regenerated dataset.

## The one prompt that fails, and why

`t3_bug_report_missing_steps` — "The app keeps logging me out. I'm using Chrome on Windows
11." — scored 0.00 in both runs. Failing twice on a regenerated dataset makes this a
reproducible defect rather than judge variance.

The chat response looks reasonable. It thanks the customer and returns a real ticket ID,
so the failure is invisible unless you open the table. Both tickets filed from this prompt
contain junk in `stepsToReproduce`:

| ticketId    | stepsToReproduce as stored                                                           |
| ----------- | ------------------------------------------------------------------------------------ |
| `c5f0bf6d…` | "Please provide specific actions or scenarios that lead to the app logging you out." |
| `5ad7eb30…` | "Using the app."                                                                     |

The first is the model's own clarifying question, written into the ticket as though the
customer had said it. The second is filler. Compare a healthy ticket from the same table —
"Open the shop, tap any category" — and the difference is obvious to a human reader and
invisible to a correctness score.

The cause is a tension in my own prompt. To stop the bot from pestering customers who have
already explained themselves, I wrote a deliberately lenient rule for `stepsToReproduce`:
any described action counts, it need not be numbered, and "It crashes when I click Pay" is
given as explicitly sufficient. That works when the fault and its trigger are separate
statements. It breaks when they are the same sentence: "the app keeps logging me out"
reads as both, so the model treated the field as satisfied — and rather than leaving it
empty, invented something to put there, violating the rule two lines below that requires
every field value to be traceable to the customer's own words.

The fix is not to remove the leniency, which prevents a worse failure mode, but to require
that the steps describe something distinct from the description, and to forbid filling a
field with the assistant's own question. I have not applied it: changing a rule this
central after two scored runs would invalidate the comparison between them.

## FAQ extension

I added three entries to `online_shop_faq.md` (33, 34, 35), covering customs duties on
international orders, genuine duplicate charges, and post-purchase price adjustments. All
three fill real gaps — the original FAQ says where the shop ships and what shipping costs,
but nothing about import taxes.

I verified pickup with an identical question asked before and after: "Do I have to pay
customs fees if I order from another country?" Before the edit the bot correctly handed
off to the support line, because the FAQ was silent. After adding entry 33 and re-running
`create_harness.py` : with no other redeployment, the same question was answered from the
FAQ. The customs prompt is also test `t6`, so the extension is covered by the automated
evaluation as well as by the manual screenshots.

## Remaining limitations

**Multi-turn ticket filing is unreliable.** Single-turn reports containing all three
fields file correctly and return a real UUID. Collection across several turns does not: in
every attempt the model either called the tool prematurely and never retried, or skipped
the call entirely and fabricated an ID. Adding a retry rule did not fix it. The evidence is
therefore split across two transcripts : `multi-turn-collection-transcript.txt` shows the
collection behaviour with follow-up questions, and `bug-report-transcript.txt` shows a
report that actually reaches DynamoDB.

**Prompt rules are followed inconsistently.** The reasoning-tag rule and the
no-exploratory-call rule both reduced the behaviours without eliminating them, and both
reappeared in later sessions despite greedy decoding. Nova Pro appears to weight a long
system prompt unevenly from one session to the next.

**The model occasionally invents advice.** In one session it suggested clearing the browser
cache — sensible troubleshooting, but absent from the FAQ and therefore a violation of the
FAQ-only rule. In another it asked again for steps the customer had just given, despite the
rule telling it to re-read what was already provided.

## What I would do next

Shorten and restructure the prompt. It is now long enough that adherence degrades, and the
rules that fail most often are the ones buried in the middle.

More importantly, change what the tests assert on. The defect that mattered most in this
project : a confident answer wrapped around an empty or invented field is invisible to a
single-turn correctness metric, because the response reads as perfectly helpful. Both times
I found something real, it was by comparing a conversation against the DynamoDB row it
claimed to have created. A suite that asserted on stored rows rather than on conversational
text would have caught both automatically.
