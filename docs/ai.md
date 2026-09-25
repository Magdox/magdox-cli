# AI review with your own model

The MAGDOX engine decides what is a finding. With `--ai`, a model you choose
reads each finding and the code around it and adds an advisory verdict:
**likely real**, **likely false positive** or **needs review**, with a short
reason. With `--ai-fix` it also proposes a patch, and the engine re-scans a
patched copy of the file: a fix is marked **verified** only when the finding
is gone and nothing new appeared. Your files are never modified.

AI verdicts never delete a finding or change its severity. Code you scan can
contain text written to mislead a model, so treat every verdict as a second
opinion.

## Set up a provider

```sh
magdox ai add anthropic                 # key read from ANTHROPIC_API_KEY, or prompted
magdox ai add openai
magdox ai add groq
magdox ai add gemini
magdox ai add bedrock --region us-east-1  # Bedrock API key (AWS_BEARER_TOKEN_BEDROCK)
magdox ai add azure --base-url https://<resource>.openai.azure.com/openai/v1
magdox ai add ollama                    # local, no key
magdox ai add mine --provider openai --base-url https://llm.example.com/v1 --key-env MY_LLM_KEY

magdox ai models groq                   # list the models your key can use
magdox ai use groq --model <model-id> --effort medium
magdox ai test
```

Other presets: `openrouter`, `mistral`, `deepseek`, `together`, `meta`,
`custom`. Any OpenAI-compatible API works with `--provider openai --base-url`.

Effort is `none`, `low`, `medium` or `high`. It maps to extended thinking on
Claude and to `reasoning_effort` elsewhere; models without a reasoning control
ignore it.

Keys: a typed key is stored owner-only (0600) in the MAGDOX config directory,
beside your sign-in. `--key-env VAR` stores only the variable name, which is
what CI should use. `magdox ai status` never prints a key.

## Approve a project, then scan

```sh
magdox scan --ai --ai-show-prompt .     # see exactly what would be sent; sends nothing
magdox ai allow .                       # approve this project
magdox scan --ai .
magdox scan --ai-fix .                  # verdicts plus engine-verified fixes
```

Approval is stored in your config, never in the repository, so a cloned repo
cannot approve sending itself anywhere. In CI set `MAGDOX_AI_CONSENT=1`.

What is sent: the rule and its guidance, the engine's trace, and up to about
240 lines around the finding, with paths relative to the project. Anything
the secrets engine recognises, and credential-style assignments, are replaced
with `[REDACTED]` first.

## Options

| Flag | Meaning |
|---|---|
| `--ai` | Add a verdict to each finding |
| `--ai-fix` | Also propose fixes and verify them by re-scan |
| `--ai-show-prompt` | Print what would be sent, send nothing |
| `--ai-max <n>` | Findings reviewed per run, highest severity first (default 40) |
| `--ai-gate` | Likely false positives at 80%+ confidence do not trip `--fail-on`. Opt-in only: code can carry text that talks a model into "false positive", so keep it off for untrusted repositories |
| `--ai-no-cache` | Ignore cached answers |

Answers are cached owner-only by provider, model, effort and exact prompt, so
re-running an unchanged scan costs nothing.

Rate limits: when a provider answers 429, the CLI waits for the time the
provider asks for and retries up to five times. Free tiers with small
per-minute token limits (for example Groq's 8,000 tokens per minute) still
work, just more slowly. Reasoning models get extra output room for thinking;
if one returns no text, lower `--effort`.

`--offline` (or `MAGDOX_OFFLINE=1`) refuses remote providers; a local one such
as Ollama still works. `MAGDOX_AI_PROVIDER=<name>` picks a provider for one run
without changing the active one.

`MAGDOX_AI_DEBUG=1` prints per-finding timings (prompt, model, verification)
to stderr.

"Verified" means the engine no longer reports that finding on the patched
file and reports nothing new. It is not proof the patch is the best fix:
review it as you would any pull request.

## Uploads

With `--upload`, each finding carries the verdict, confidence, provider and
model. The written reason is included only with `--include-code`, and a
proposed patch is never uploaded. Servers that do not yet accept AI reviews
receive the findings without them.
