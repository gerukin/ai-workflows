# Main specification

The workflows spec aims to handle everything from simple one off fully deterministic tool calls to repeated open-ended dynamic interactions.

## General recommendations

Workflows should have few steps and be fairly simple/constrained if possible. Composition should be preferred to long and convoluted workflows. As such, control flow options are limited to encourage this.

## Main properties

- `code`: A unique identifier for the workflow. No other workflow can have the same code.
- `name`: A name for this workflow which can serve as a tool name.
- `description`: A brief description of the workflow, which should be human and LLM friendly.
- `steps`: An array of steps to execute in the workflow.
- `onNewTriggerEvent` (optional): A list of options for an LLM to decide what to do upon receiving a new trigger event for the same topic.
- `parameters`: See the `parameters` for tools. This is what inputs this workflow accepts.
- `synchronous` (optional): Whether the workflow is synchronous or not (default is asynchronous).
- `output` (optional): The expected output of the workflow.

```yaml
code: example_workflow
name: exampleWorkflow
description: An example workflow to demonstrate the structure.
synchronous: true # in this case we want to run all steps before returning the output
steps:
  - code: step1
    # ...
  - code: step2
    # ...
output: `{{$.workflow.steps.step2.output.data}}`
```

> [!IMPORTANT]
> The output, if set, is returned as part of the workflow result. If the workflow is synchronous, all steps will complete before the output is returned (it is therefore possible to reference step outputs). If the workflow is asynchronous, there are 2 cases:
>
> - The workflow is executed as part of a parent workflow: output is similar to the synchronous case (although it runs fully asynchronously).
> - The workflow is called from a non workflow trigger (ex: backend call): the output is returned as soon as the workflow is accepted for processing. It is therefore not possible to reference step outputs in the output (an empty result will be returned).

### On new trigger event

Only one workflow per topic may run at a time. When a new event is received for the same topic, while the workflow is already running.

Options are:

- `abort_current_run_new`: Abort the current workflow and retry the workflow with the new event instead.
- `abort_all`: Abort the current workflow and do nothing else (no need to run a new workflow on the new event).
- `queue_new`: Queue the new event for processing after the current workflow completes.
- `add_new_continue` (disabled for now): Add the new event to the current workflow and continue processing.

No options are enabled by default. When nothing is enabled, the `queue_new` option is used without asking the LLM anything.

```yaml
onNewTriggerEvent:
	- abort_current_run_new
	- abort_all
	- queue_new
	# - add_new_continue
```

> [!WARNING]
> Adding the new event is actually tricky and may not make sense in most cases. It is not clear yet where and how to inject the new event into the current workflow, and this will be considered and defined later.

## Steps & control flow

Steps are the main building blocks of a workflow and can be ordered sequentially or run in parallel.

Steps within the same workflow all share the same overall context (if you need fresh context isolation, you need to have the LLM call a workflow).

Common properties:

- `code`: A unique identifier for the step.
- `description`: A brief description of the step, which should be human and LLM friendly.
- `type`: The type of the step (see the separate section on step types).
- `fallbackStep` (optional): If a constraint is violated, this step will be executed instead of moving forward with more actions within the current step or the remaining sub steps.

### Unique code

To allow for easy referencing, each step must be assigned a unique code:

```yaml
# another `code` with `step1` is not allowed anywhere in this workflow
- code: step1
  # ...
```

> [!IMPORTANT]
> The code must be unique within the entire workflow.

### Description

A description is also essential for understanding the purpose and function of the step. It may be shown to LLMs, included in logs, or used for debugging.

```yaml
- code: step1
  description: Get the current weather for Paris.
  # ...
```

### Step type

Steps always have a type:

- `parallel_sub_steps`: A step that runs multiple substeps in parallel.
- `sequential_sub_steps`: A step that runs multiple substeps in sequence.
- `workflow_call`: A step that calls another workflow.
- `tool_call`: A step that calls a tool.
- `llm_text`: A step that performs inference using the specified LLM to get some textual output.
- `llm_structured`: A step that performs inference using the specified LLM to get structured data output.
- `llm_tool`: A step that uses an LLM to interact with one or more tools.
- `pick_sub_step`: A step that uses an LLM to select one or more of several options (sub steps) to execute.

Each type has its own required properties and output.

#### `parallel_sub_steps`

Specific properties:

- `substeps`: An array of steps to run in parallel.

```yaml
- code: step1
	type: parallel_sub_steps
	# All substeps run in parallel
	substeps:
		- code: step1a
			# ...
		- code: step1b
			# ...
```

Output is always an object with the result of all sub steps:

```jsonc
{
	"step1a": {
		// if the step completed successfully and returned a result
		"output": "..."
	},
	"step1b": {
		// if an error occurred
		"error": "..."
	}
}
```

#### `sequential_sub_steps`

Specific properties:

- `substeps`: An array of steps to run in sequence.

```yaml
- code: step1
  type: sequential_sub_steps
  substeps:
		# This runs first
    - code: step1a
			# ...
		# This runs second
		- code: step1b
			# ...
```

Output is always an object with the result of all sub steps:

```jsonc
{
	"step1a": {
		"output": "..."
	},
	"step1b": {
		"output": "..."
	}
}
```

#### `workflow_call`

Specific properties:

- `workflowId`: The code of the workflow to call.
- `parameters`: The parameters to pass to the workflow.

```yaml
- code: step1
  type: workflow_call
  workflowcode: some_workflow
  parameters:
		param1: something
```

Output is always an object with the result of the workflow call:

```jsonc
{
	"data": "..."
}
```

#### `tool_call`

Specific properties:

- `toolId`: The code of the tool to call.
- `parameters`: The parameters to pass to the tool.

```yaml
- code: step1
	type: tool_call
	toolcode: weather_in
	parameters:
		city: Paris, France
		type: current
```

Output is always an object with the result of the tool call:

```jsonc
{
	"data": "..."
}
```

#### `llm_text`

The LLM can optionally be provided access to tools and workflows but the primary goal is to get a textual output back.

Specific properties:

- `modelId`: The code of the model to use for inference.
- `messageContextOptions`: The context options to provide messages to the model (see the [context management](#context-management) section).
- `prompt`: The prompt to pass to the model.
- `llmOptions`: Options to customize the LLM behavior.
- `toolOptions` (optional): Tool options available to the LLM (see the tools section).

```yaml
- code: step2
  type: llm_text
  modelcode: think_low_smart_high
  prompt: |
    Describe the weather in `{{$.workflow.steps.step1.input.city}}` using the following data:
    `{{$.workflow.steps.step1.output.data}}`.
```

Output is always an object with the result of the LLM inference:

```jsonc
{
	"data": "..."
}
```

#### `llm_structured`

The LLM can optionally be provided access to tools and workflows but the primary goal is to get a structured output back.

```yaml
- code: step1
	type: llm_structured
	# same as `llm_text`
	outputSchemacode: some_schema_id
```

You can see an example of a schema [here](./examples/structured-schemas/invoice.yaml).

#### `llm_tool`

The LLM can optionally be provided access to tools and workflows but we want to end on the last tool call.

```yaml
- code: step1
	type: llm_tool
	# same as `llm_text`
```

> [!NOTE]
> A `done` tool is automatically added for the LLM to call in order to finish early if no more tools need to be called (this tool is only added if ending early is possible based on the tool options and calls so far).

#### `pick_sub_step`

Specific properties:

- `modelId`: The code of the model to use for inference.
- `prompt`: The prompt to pass to the model.
- `llmOptions`: Options to customize the LLM behavior.
- `minSteps`: The minimum number of sub steps to execute.
- `maxSteps`: The maximum number of sub steps to execute.
- `substeps`: An array of substeps to choose from.

```yaml
- code: step1
  type: pick_sub_step
  modelcode: think_low_smart_high
  prompt: Choose the appropriate next step.
	substeps:
		- code: step1a
			# ...
		- code: step1b
			# ...
```

> [!NOTE]
> The `substeps` are passed as regular tools for the LLM to call, with no input parameters. The substep description is important here as it provides context for the LLM to understand it.

Output is always the result of the selected substep:

```jsonc
{
	"step1a": {
		"output": "..."
	}
}
```

### Fallback step

If a constraint would be violated by continuing, this step will be executed instead to produce a final outcome.

```yaml
- code: step1
	# ...
	fallbackStep:
		# no `code` as it inherits the `code` of the parent
		type: llm_text
		modelcode: think_low_smart_high
		prompt: |
			You must use the information you have gathered so far and return a short answer as best you can. You are no longer allowed to ask for additional information or call any tools, you may ONLY reply, now.
```

> [!IMPORTANT]
> If a step (or one of its sub steps) violates a constraint and returns early, the fallback step will be executed. If the step violates a constraint but did not return early (because the constraint could not be enforced until after the output was produced - and there is no further work to do), the fallback step will not be executed.

The output is based on the `type` and it becomes the output of the parent.

### Output

Each step produces a specific output (based on its type), which can be referenced and used in subsequent steps using JSONPath notation:

```yaml
- code: step2
  type: llm_inference
  modelcode: think_low_smart_high
  prompt: |
    Describe the weather in `{{$.workflow.steps.step1.input.city}}` using the following data:
    `{{$.workflow.steps.step1.output.data}}`.
```

> [!WARNING]
> You must ensure that a referenced step's output is available before it is used.

> [!NOTE]
> It is possible for a step to return no data (a function call with no return value).

> [!NOTE]
> A `metadata` property will likely be added in the future besides `output` (ex: `step1.metadata` vs `step1.output`).

### Interpolation

All values within a step can be interpolated using JSONPath notation for more dynamic execution. This applies to ALL fields except the step code and the type, which cannot be dynamic. Interpolation takes the form of a backtick-enclosed expression:

```
`{{$.workflow.name}}`
```

Interpolations used in strings follow these rules:

```ts
console.log(`Obj ${JSON.stringify({ key: 'value' })}`) // Obj {"key":"value"}
console.log(`Arr ${JSON.stringify([1, 2, 3])}`) // Arr [1,2,3]
console.log(`Num ${123}`) // Num 123
console.log(`Str ${'Hello, world!'}`) // Str Hello, world!
console.log(`Nil ${null}`) // Nil null
console.log(`Undef ${undefined}`) // Undef undefined
console.log(`BigInt ${123456789123456789123456789n}`) // BigInt 123456789123456789123456789
console.log(`Bool ${true}`) // Bool true
```

```yaml
- code: step2
	type: tool_call
	toolcode: some_tool
	input:
		# The output is stringified and interpolated into the string
		- param1: Some object... `{{$.workflow.steps.step1.output.data.obj}}`
```

```yaml
- code: step2
	type: tool_call
	toolcode: some_tool
	# The output object is included as an object
	input: `{{$.workflow.steps.step1.output.data.obj}}`
```

## Topics

A workflow can be long lived or short lived. When a workflow is invoked, a workflow topic can be optionally specified. A long lived workflow may receive many events sent to the same workflow topic. Or the topic could be different, indicating a new context.

It is possible to have multiple workflows listening to the same topic. Workflows called from within other workflows inherit the same topic by default, but it can be overridden.

When a trigger runs, it may specify a topic, but if not, this workflow is considered as having a random topic independent of all others.

Some `topicOptions` can be added to the workflow to delay execution of the workflow and bundle several events together. Ex:

- `addNewEventsReceivedWithin`: Specifies a time window for grouping events together. When receiving an event, the workflow will not start until the `addNewEventsReceivedWithin` has passed. If more events arrive within this window, they will be bundled together (and restart the `addNewEventsReceivedWithin` timer).
- `groupMaxSize`: If specified, it will ignore the `addNewEventsReceivedWithin` once `groupMaxSize` events have been received.

> [!NOTE]
> Only events for the same topic can be bundled together.

- [ ] TODO: add example YAML.

## Tools

Tools can be called by the LLM and 3 types exist:

- Tool
- Workflow
- Third party agent (future)

They all exist as tools, exposed to the LLM as such (when using `toolOptions`) or hard coded in `tool_call` or `workflow_call`.

> [!NOTE]
> Tools have the properties defined in the example file, but also have an optional `synchronous` property which mirrors that of a workflow.

- TODO: add example YAML.

You can see an example of a tool definition [here](./examples/tool-definitions/city-weather.yaml).

## Models

Models are simply LLM configurations identifying:

- A specific provider
- A specific model version valid for this provider
- A temperature
- Reasoning effort settings for this model and provider if applicable
- Max total tokens across all token types (full context)

## Constraints

Workflows involving LLMs can be unpredictable, and constraints are necessary to ensure reliable operation. Constraints cascade down to other workflows.

Constraints are set at the step level.

Constraints are of 3 types:

- budget
- time
- parameter

Budget and time constraints have 3 values:

- `min` (optional): used to indicate to upstream callers that this workflow/step will need at least this much (this is not enforced in the workflow/step itself, but it disables this workflow/step in the upstream caller, indicating why it cannot be called)
- `max`: used to indicate to upstream callers that this workflow/step will not exceed this much (this is enforced in the workflow itself, and the step will be forced to end early)
- wrapping up is handled by the fallback step

Parameter constraints have a single `schema` constraint, which replaces the original parameter schema with a more specific one.

> [!NOTE]
> Ideally a constraint should be evaluated BEFORE starting an action, but this is not always possible, so in practice, constraints will be exceeded, and their role is to avoid or guide further processing after that. A constraint can either be violated during execution (returns early + executes the fallback step) or after execution (was able to run fully, but with too much resources). The [fallback step](#fallback-step) is executed when a constraint is violated during execution.

> [!IMPORTANT]
> A workflow inherits ALL `max` and `value` constraints from its parent workflow, and they override any default `max` constraints set in the workflow itself. `min` constraints are unaffected.

- [ ] TODO: add example YAML.

### Budget constraints

Each step has a maximum budget, set in the base currency. The default top level budget for the workflow MUST be specified.

- [ ] TODO: add example YAML.

#### Inheritance rules

If a higher level budget would be exceeded by applying the lower level budget to the current step, the lower level budget will be ignored and replaced with the remaining budget from the higher level budget.

### Time constraints

The execution time can be limited. The default top level time limit for the workflow MUST be specified.

- [ ] TODO: add example YAML.

#### Inheritance rules

If a higher level time limit would be exceeded by applying the lower level time limit to the current step, the lower level time limit will be ignored and replaced with the remaining time from the higher level time limit.

### Parameter constraints

Each callable tool and workflow can have its input parameters constrained to a specific options. For example, a tool which takes in any string input in `param1` can be constrained to a specific string, or to an enum with specific values.

The LLM will see the constraint simply as a slightly updated JSON schema which will force it to pick from the specified options (or the specific value) for this parameter, instead of the original schema with more freedom.

- [ ] TODO: add example YAML.

#### Inheritance rules

Parameter constraints are inherited by all workflows and steps, and can be defined or overridden at any level. If a given step needs to have more freedom than its parent, it is possible to disable that constraint.

## Triggers

Triggers are anything outside the workflow which trigger the workflow. This is out of scope for this spec, but this would typically be manual actions taken by users, or event based or scheduled triggers.

In practice, the trigger just starts the workflow with the right workflow inputs and optional topic.

## Context management

For LLM related steps, each workflow step has its own messages context.

- `scope`: Controls the kinds of messages included in the context.
- `pruning` (disabled): In the future, this may control the message pruning behavior.

### Scope

The default is to include no context at all except what is in the prompt. The following values are available:

- `none`: Includes no messages at all (default). Context, if any is necessary, must be manually injected into the prompt.
- `current`: Includes only the messages in the current step.
- `hierarchical`: Includes all messages in the current hierarchy (current step, earlier siblings, parent, and up to and including the trigger).
- `full`: Includes all steps executed in the workflow so far.

Steps which are not LLM related will still be injected into the context transparently, in a user message. The framework will ensure that user and assistant messages alternate no matter what.

```yaml
messageContextOptions:
	scope: hierarchical
```

## State management

State is maintained in a central store (File, DB, or cache).

Reading/writing to state is mediated through normalized function interfaces which act as adapters.
