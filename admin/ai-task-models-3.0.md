# AI models per task (3.0 preview)

How an installation operator lets a dedicated model do one AI task, such as a small local model that
reviews reactions, while everything else keeps using the organization's model. This page describes
a preview; details can change until 3.0 is released.

## Which tasks

Every AI call uses the organization's default model (**Manage → AI Assistant → Models & budget →
Default model**) unless something else is set. Two tasks can run on a model of their own:

| Setting | Shown as | Used for |
| --- | --- | --- |
| `ai.model.reaction_review` | Model for reaction reviews | The AI review of every resolved reaction |
| `ai.model.assessment` | Model for AI assessments | AI steps that have neither a model nor tools of their own, such as an investigation's assessment of its photos |

The value is the id of a model in the model catalog. Empty (the default) means the organization's
default model, as before.

- **Installation-wide.** A system admin sets the value for every organization in **Manage →
  Configuration** (scope `GLOBAL`).
- **Per organization.** An organization may set its own value under **Manage → Follow & Relief
  Settings**, in the group **AI models**, which overrides the installation's choice. It cannot be
  set per site.

For AI assessments, a model chosen on the step itself always wins, and an AI step with tools keeps
the organization's default model.

## When the task model cannot answer

The organization's default model takes over the task when the task model:

- is not allowed for the organization (**Models & budget → Models**),
- cannot be reached,
- refuses the request, or
- answers in a form the task cannot use.

In a reaction review the task model gets at most half of the review's time, so the default model
always has the other half to answer. A task model is never used as the fallback of another model.

Task models go through the same checks as every other model: the organization's allowed models,
data residency, budgets, usage records and the audit log.

## Free review models and the monthly review budget

Reaction reviews have their own budget, **Monthly review budget**
(`reaction_review.monthly_budget_eur`), next to the **Cost cap per review**. Once the month's
review budget is used up, reviews on a paid model are skipped.

A review on a **free** model keeps running after the budget is used up. A model counts as free when
its catalog entry has the cost **Free** (price 0), or when it is an OpenAI-compatible model that runs
in your network without prices. If a free review model cannot answer and the organization's default
model is a paid one, that paid call is refused once the budget is used up, and the review is
skipped with the budget as its reason.

## Adding a local model for a task

1. **Serve the model** on a server with an OpenAI-compatible API in your network, for example vLLM.
2. **Add it to the catalog.** As system admin, open **Manage → AI Assistant → Model catalog → Add
   model**:
   - **Provider**: *OpenAI-compatible (vLLM, Ollama)*,
   - **Provider's model name**: the name the server serves the model under,
   - **Server address**: the server's API base, e.g. `http://vllm.example.internal:8000/v1`,
   - **Context window (tokens)**: what the server runs the model with,
   - **Cost**: *Free*,
   - **Runs in the customer's network**: on (the platform then sends no API key),
   - under **Advanced**, the **Id (used in policies)**: the id you will use in the setting.
3. **Allow it** for the organizations that should use it (**Models & budget → Models**; none ticked
   means all models are allowed).
4. **Set the task's setting** (`ai.model.reaction_review` or `ai.model.assessment`) to the model's id,
   installation-wide or for one organization.

### How the agent service calls a task model

Some task models are trained for a shorter prompt or answer best with a fixed temperature. The
agent service's environment variable `AGENT_MODEL_PROFILES` describes such models, as one JSON
object of catalog model id → profile:

```bash
AGENT_MODEL_PROFILES='{"my-review-model": {"prompt": "compact", "output": "native", "temperature": 0}}'
```

| Profile field | Values | When not set |
| --- | --- | --- |
| `prompt` | `full`, or `compact`: the shorter review prompt | `full` |
| `output` | `tool`, or `native`: the server keeps the answer to the expected JSON format | `tool` |
| `temperature` | 0 to 2 | the provider's |
| `thinking` | `off`, `low`, `medium`, `high`, `xhigh`, `max` | what the task asks for; reviews ask for none |
| `timeout_seconds` | one call's timeout | the catalog's **Timeout per call** |
| `max_output_tokens` | one call's longest answer | the catalog's **Longest answer** |

With the Helm chart, set the value `agentService.modelProfiles` instead of the variable; the
chart writes it as JSON into `AGENT_MODEL_PROFILES`:

```yaml
agentService:
  modelProfiles:
    zqnt-reaction-review: {prompt: compact, output: native, temperature: 0}
    my-review-model: {prompt: compact, output: native, temperature: 0}
```

Left at `null` (the chart's default), the variable is not set and the agent service's default
below applies; `{}` sets no profiles. A JSON string is passed through as is. A value you set
replaces the default, so keep `zqnt-reaction-review` in it as long as that model does the reaction
review.

A model without a profile is called like any other model, so most models need no entry. When the
variable is not set, only the model id `zqnt-reaction-review` has a profile (`compact`, `native`,
temperature 0); `{}` means no profiles. An invalid entry stops the agent service at startup with
the reason. When the organization's default model takes over the task, it is called with its own
settings and the full prompt.

A reaction review always asks its model for a direct answer, without thinking, and each call ends
after the catalog's **Timeout per call**, but never later than the review's own time.
