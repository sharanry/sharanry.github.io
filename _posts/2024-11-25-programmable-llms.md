---
layout: post
title: "Programmed Intelligent Systems"
categories: llms, agentic systems, structured llms, dsls
---

Some quick thoughts on how to make LLMs more programmable.

I have been thinking a lot about how to efficiently encode complex multi-step branched business logic in a way which is efficient, robust and easy to debug. 

Here are a few key challenges that make adopting AI for business logic tasks harder than necessary:
- Structured generation of Pydantic objects, json, etc. becomes important when dealing with data extraction tasks. - outlines and instructor are great for this.
- Business logic often involves complex multi-step decision making which requires efficient context management. LMQL seems to be tailor made for this however it lacks the level of structured generation that outlines and guidance seem to provide.
- LLMs are often used for classification tasks where having insights into the confidence of the classification is important. - no tool seems to be able to do this yet.

We are going beyond using llms as chatbots but expecting them to do a lot in a one-shot manner still rarely works in practice beyond demos. For deployable AI systems, we need to be able to guide the llm in a multi-step manner.

## Vision

Let me lay out a vision for a system which can guide the generation of llms in a multi-step manner.

### Task management assistant

Let us first consider a simple task management assistant which I set out to build which led me to these insights.

We start by initializing the llm with the appropriate context.
```python
llm = ChatOpenAI(model="gpt-4o-mini", system_prompt="You are a helpful task management assistant.")
```


We then classify the user's request into one of the three actions: create, update, delete.
```python
action = classify(llm, request, "User's task management request", ["create", "update", "delete"], beam_search=True)
```


We then parse the user's request into the appropriate Pydantic objects.
```python
class Task(BaseModel):
    title: str = Field(descrieption="The title of the task")
    due_date: datetime = Field(description="The due date of the task", default=datetime.now() + timedelta(days=1))
    status: str = Field(description="The status of the task", default="pending")

class UpdateTask(BaseModel):
    title: str
    due_date: date
    status: str

class DeleteTask(BaseModel):
    title: str
```

We then perform the appropriate action based on how the user's request was classified. 
```python
if action == "create":
    # we are able to monitor the confidence of the classification
    print("Creating a new task with confidence: ", action.confidence)
    
    # provide only the apppropriate additional context for the task creation
    task = parse(llm, request,"Create a new todo with the following attributes: title, description, due_date, status. {{ Task | schema}}", Task)
    
    # we are able to search for existing tasks in the database
    existing_task = db.semantic_search(task)
    if existing_task is not None:
        user_prompt("Similar task already exists. Are you sure you want to create a new task?", ["yes", "no"])
        return
    db.create_task(task)
```

Similarly, we can update or delete tasks.
```python
elif action == "update":
    print("Updating a task with confidence: ", action.confidence)
    task_update = parse(llm, request, "Update the todo with the following attributes: title, description, due_date, status. {{ UpdateTask | schema}}", UpdateTask)
    existing_task = db.semantic_search(task_update)
    if existing_task is None:
        print("Task not found")
        return
    db.update_task(existing_task, task_update)
elif action == "delete":
    if action.confidence < 0.9:
        # since deletion is a critical operation, we want to make sure the user really wants to delete the task
        print("Task deletion stalled due to low confidence: ", action.confidence)
        confirm = userprompt("Are you sure you want to delete this task?", ["yes", "no"])
        if confirm == "no":
            print("Task deletion cancelled.")
            return
    print("Deleting a task with confidence: ", action.confidence)
    task_delete = parse(llm, request, "Delete the todo with the following attributes: title, description, due_date, status. {{ DeleteTask | schema}}", DeleteTask)
    existing_task = db.semantic_search(task_delete)
    if existing_task is None:
        print("Task not found")
        return
    db.delete_task(existing_task)
```

This is just a high level overview of the system. The actual implementation is a lot more complex but the key idea is that we are able to guide the llm in a multi-step manner while still being able to monitor the confidence of the classification and the state of the llm.

<!-- ## Cookbook Examples from Outlines

Every cookbook example can be simplified to a procedural program which is easy to understand, debug and modify. We can see how this paradigm can be used easily extended each of the examples to tackle more complex tasks.

### Classification

```python
llm = ChatOpenAI(model="gpt-4o-mini", system_prompt="You are an experienced customer success manager.")

urgency = classify(llm, email, "User's request", ["urgent", "not urgent"])

# the demo stops here

if urgency == "urgent":
    db.add_tag(email.id, "urgent")
    
    email.reply("Thank you for contacting us. We have noted the urgency and will get back to you very soon.")
elif urgency == "not urgent":
    db.add_tag(email.id, "not urgent")

    related_discussions = db.semantic_search(discussions, email)

    email.reply("We will get back to you soon. Meanwhile, you might find the following discussions helpful: " + ", ".join([d.title for d in related_discussions]))

``` -->

## Inspirations
- Outlines 
    - robust structured generation of Pydantic objects, json, etc.
    - token management
    - What is lacking?
        - stopping generation midway and guiding the generation based on current state.
        - branching, loops, conditionals
- guidance
    - structured generation with option to guide the direction of the generation explicitly with branching
    - llm state management which allows for a specific llm state to be reused for multiple generations - useful in loops and conditionals
    - simplicity of the classification task using `select`.
    - What is lacking?
        - classification confidence using beam estimate of the classification 
        - complex object constraints
        - structured generation of Pydantic objects, json, etc.
- lmql
    - procedural programming of llms
    - allows for conditionals, loops, and other control flow constructs
    - What is lacking?
        - complex constraints
        - structured generation of Pydantic objects, json, etc.

- Langchain
    - complex prompt management
    - model family specific considerations
    - What is lacking?
        - a lot of structured things
