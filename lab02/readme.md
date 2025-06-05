Below is a breakdown of the provided code, section by section, showing how each piece fits into Temporal’s programming model (workflows, activities, task queues, client/worker interactions, etc.).
---

## 1. Import and Setup
```python
import asyncio
from dataclasses import dataclass
from datetime import timedelta

from temporalio import activity, workflow
from temporalio.client import Client
from temporalio.worker import Worker
```
- `asyncio`: Required because both the activity and the workflow run asynchronously under the hood. You’ll start the whole process with asyncio.run(main()).

- `dataclass`: Used to define a simple, serializable input structure for the activity. Temporal recommends packing activity parameters into a single (version‐stable) dataclass rather than using multiple bare parameters.

- `timedelta`: Used to set timeouts on the activity invocation.

- `temporalio.activity and temporalio.workflow`: These modules provide the decorators and primitives you need to define activities and workflows in Python.

- `Client`: The Temporal client connects to a Temporal server (here, at localhost:7233).

- `Worker`: This is the process that actually polls for workflow and activity tasks on a given task queue and executes them.

## 2. Defining a Dataclass for Activity Input
```python
@dataclass
class ComposeGreetingInput:
    greeting: str
    name: str
```
- Why a dataclass?

    Temporal strongly recommends wrapping all activity arguments into a single dataclass (or POJO) for backwards compatibility. If later you need to add fields (e.g. a locale: str), it won’t break old workflow executions that expect only (greeting, name).

- Fields

    - `greeting`: str (e.g. `"Hello"`)

    - `name`: str (e.g. `"World"`)

At runtime, when you call execute_activity(...), this object will be serialized, sent over to the worker, and deserialized into the activity function.

## 3. Defining the Activity

```python
@activity.defn
async def compose_greeting(input: ComposeGreetingInput) -> str:
    activity.logger.info("Running activity with parameter %s" % input)
    return f"{input.greeting}, {input.name}!"
```
- `@activity.defn` decorator

    Marks this function as an “activity” in the Temporal model. An activity is a unit of work that can run on any worker process (possibly on a different machine), is retried automatically on failure, and can have timeouts and heartbeating configured.

- Signature

    `Input`: One argument of type ComposeGreetingInput. Temporal will serialize this object automatically.

    `Return`: A plain str. When the activity completes successfully, that str becomes the activity result and is returned to whoever invoked it (in this case, the workflow).

- Logging

    It logs the received input via activity.logger.info(…). If you enable logging (see Section 6), you’ll see it in your worker’s stdout or wherever your logging is configured.

- Work Performed

    Very simple: concatenate greeting and name into a single string (e.g. "Hello, World!") and return it.

## 4. Defining the Workflow
```python
@workflow.defn
class GreetingWorkflow:
    @workflow.run
    async def run(self, name: str) -> str:
        workflow.logger.info("Running workflow with parameter %s" % name)
        return await workflow.execute_activity(
            compose_greeting,
            ComposeGreetingInput("Hello", name),
            start_to_close_timeout=timedelta(seconds=10),
        )
```
- `@workflow.defn` decorator

    Declares that GreetingWorkflow is a workflow type. Temporal’s Python SDK uses this decorator to register and serialize the class definition for the server.

- `@workflow.run` method

    Marks the method that will be invoked when a new workflow execution starts.

    - Signature: `(self, name: str) -> str`

        - The workflow takes a single string parameter (name).

        - Returns a str (which in our example will be the greeting returned by the activity).

- Inside `run(...)`

    1. `workflow.logger.info(…)`
        If/when you enable workflow‐side logging, you’ll see a message like “Running workflow with parameter Alice” whenever someone starts this workflow with name="Alice".

    2. `await workflow.execute_activity(...)`
        This is a special Temporal SDK call. It schedules the compose_greeting activity on the same task queue (unless otherwise overridden), waits for it to complete, and then returns its result to the workflow code.

        - First argument: the Python function object compose_greeting (the activity we defined above).

        - Second argument: an instance of ComposeGreetingInput("Hello", name). The workflow serializes this object and sends it over to the worker.

        - `start_to_close_timeout=timedelta(seconds=10):`

            Tells Temporal that once the activity is started, it must finish within 10 seconds (or it will be considered failed/timeout).
    3. Return value

        Whatever the activity returns (e.g. "Hello, World!") is returned from run(...), which becomes the overall workflow result.

## 5. `main()` Function: Wiring Up Client, Worker, and Workflow Execution

```python
async def main():
    # Uncomment the lines below to see logging output
    # import logging
    # logging.basicConfig(level=logging.INFO)

    # Start client
    client = await Client.connect("localhost:7233")

    # Run a worker for the workflow
    async with Worker(
        client,
        task_queue="hello-activity-task-queue",
        workflows=[GreetingWorkflow],
        activities=[compose_greeting],
    ):

        # While the worker is running, use the client to run the workflow and
        # print out its result. Note, in many production setups, the client
        # would be in a completely separate process from the worker.
        result = await client.execute_workflow(
            GreetingWorkflow.run,
            "World",
            id="hello-activity-workflow-id",
            task_queue="hello-activity-task-queue",
        )
        print(f"Result: {result}")
```
## 5.1. Enabling Logging (Optional)
```python
    # import logging
    # logging.basicConfig(level=logging.INFO)
```
- If you uncomment these two lines, both your workflow logs (via `workflow.logger.info`) and your activity logs (via activity.logger.info) will show up in the console. This is often essential for debugging when your worker doesn’t seem to be polling.

## 5.2. Creating the Temporal Client
```python
client = await Client.connect("localhost:7233")
```
- What this does:

    - Connects to a local Temporal server endpoint at `localhost:7233`.

    - By default, it uses the default namespace on that server. If you have created a custom namespace (e.g. via tctl namespace register), you could pass a namespace="your-namespace" argument here.

    - Once you have a Client object, you can use it to start/describe workflows or query them, etc.

## 5.3. Starting a Worker
```python
async with Worker(
    client,
    task_queue="hello-activity-task-queue",
    workflows=[GreetingWorkflow],
    activities=[compose_greeting],
):
    …

```
- `async with Worker(...)`
    Spawns a worker process in the same Python runtime. Under the hood, it does a few things:

    1. Polls the Temporal server for new workflow tasks on task_queue="hello-activity-task-queue".

    2. Registers that it knows how to execute the GreetingWorkflow and the compose_greeting activity.

    3. Keeps polling as long as the async with context is open. When you exit that context (e.g. your code finishes or you cancel), the worker will gracefully shut down.

- Arguments

    - `client`: The Client instance that’s connected to Temporal.

    - `task_queue="hello-activity-task-queue"`:
    Both the workflow stub (below) and the worker share the same “task queue” name. Workflows schedule activities on this queue, and the worker listens on that queue for both workflow tasks and activity tasks.

    - `workflows=[GreetingWorkflow]`:
    A list of workflow classes that this worker can run.

    - `activities=[compose_greeting]`:
    A list of activity function objects that this worker can execute.

## 5.4. Executing the Workflow (Client Side)
```python
result = await client.execute_workflow(
    GreetingWorkflow.run,
    "World",
    id="hello-activity-workflow-id",
    task_queue="hello-activity-task-queue",
)
print(f"Result: {result}")

```

- `client.execute_workflow(...)`

    This call does three things behind the scenes:

    1. Starts a new workflow execution of type GreetingWorkflow (specifically calling its run(...) entrypoint) with the single argument "World".

    2. Gives it a workflow ID: "hello-activity-workflow-id". This is a unique identifier for that workflow execution; if you try to start another workflow with the same ID while one is already running, you’ll get an error (unless you allow re‐runs).

    3. Specifies task_queue="hello-activity-task-queue", so the Temporal server knows which workers to dispatch tasks to.

    4. Returns a Python coroutine that resolves to the final result of the workflow once it completes. In this example, the workflow simply calls the activity, gets back "Hello, World!", and completes immediately.

- `await`
    Since execute_workflow is asynchronous, you await it. Once it finishes, you get back the string that the activity returned, and you print it.

## 6. Putting It All Together
```python
python3 hello_activity.py
```

1. Client.connect("localhost:7233") connects to your local Temporal server.

2. async with Worker(...) starts a worker thread that:

    - Polls on "hello-activity-task-queue" for workflow tasks and activity tasks.

    - Knows how to instantiate and run GreetingWorkflow.

    - Knows how to invoke compose_greeting when the workflow requests it.

3. Inside the same async with block, you call:
```python
result = await client.execute_workflow(
    GreetingWorkflow.run,
    "World",
    id="hello-activity-workflow-id",
    task_queue="hello-activity-task-queue",
)
```
  - Temporal server enqueues a new workflow task for GreetingWorkflow with argument "World".

  - The worker picks up that workflow task (because it’s listening on "hello-activity-task-queue").

  - The worker replays or begins running GreetingWorkflow.run("World"), sees the call to execute_activity(compose_greeting, ComposeGreetingInput("Hello", "World"), …).

  - The worker converts that into an activity task and sends it back to the server, which immediately routes it to the same (or another) worker listening on the same queue.

  - The worker picks up the activity task, invokes compose_greeting(ComposeGreetingInput("Hello","World")).

  - compose_greeting logs and returns "Hello, World!" to the worker, which then completes the activity task.

  - The workflow run resumes, returning that string as its final result.

  - client.execute_workflow(...) returns "Hello, World!" on the client side, and you print("Result: Hello, World!").
