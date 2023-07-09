# Example: Planning and Executing a Query Plan

In this example, we will demonstrate how to use the OpenAI Function Call `ChatCompletion` model to plan and execute a query plan using a question-answering system. We will define the necessary structures using Pydantic and show how to execute the query plan step-by-step.

!!! note "Graph Generation"
    Notice that this example produces a flat list of items with dependencies that resemble a graph, while pydantic allows for recursive definitions, its much easier and less confusing for the model to generate flat schemas rather than recursive schemas. If y ou want to see a recursive example see [recursive schemas](recursive.md)

## Defining the Structures

Let's define the necessary Pydantic models to represent the query plan and the queries.

```python
import enum
from typing import List

from pydantic import Field
from openai_function_call import OpenAISchema


class QueryType(str, enum.Enum):
    """Enumeration representing the types of queries that can be asked to a question answer system."""

    SINGLE_QUESTION = "SINGLE"
    MERGE_MULTIPLE_RESPONSES = "MERGE_MULTIPLE_RESPONSES"


class Query(OpenAISchema):
    """Class representing a single question in a query plan."""

    id: int = Field(..., description="Unique id of the query")
    question: str = Field(
        ...,
        description="Question asked using a question answering system",
    )
    dependancies: List[int] = Field(
        default_factory=list,
        description="List of sub questions that need to be answered before asking this question",
    )
    node_type: QueryType = Field(
        default=QueryType.SINGLE_QUESTION,
        description="Type of question, either a single question or a multi-question merge",
    )


class QueryPlan(OpenAISchema):
    """Container class representing a tree of questions to ask a question answering system."""

    query_graph: List[Query] = Field(
        ..., description="The query graph representing the plan"
    )

    def _dependencies(self, ids: List[int]) -> List[Query]:
        """Returns the dependencies of a query given their ids."""
        return [q for q in self.query_graph if q.id in ids]
```

## Planning a Query Plan

Now, let's demonstrate how to plan and execute a query plan using the defined models and the OpenAI API.

```python
import asyncio

import openai

def query_planner(question: str) -> QueryPlan:
    PLANNING_MODEL = "gpt-4-0613"

    messages = [
        {
            "role": "system",
            "content": "You are a world class query planning algorithm capable ofbreaking apart questions into its dependency queries such that the answers can be used to inform the parent question. Do not answer the questions, simply provide a correct compute graph with good specific questions to ask and relevant dependencies. Before you call the function, think step-by-step to get a better understanding of the problem.",
        },
        {
            "role": "user",
            "content": f"Consider: {question}\nGenerate the correct query plan.",
        },
    ]

    completion = openai.ChatCompletion.create(
        model=PLANNING_MODEL,
        temperature=0,
        functions=[QueryPlan.openai_schema],
        function_call={"name": QueryPlan.openai_schema["name"]},
        messages=messages,
        max_tokens=1000,
    )
    root = QueryPlan.from_response(completion)
    return root
```


```
plan = query_planner(
    "What is the difference in populations of Canada and the Jason's home country?"
)
plan.dict()
```

```python
{'query_graph': [{'dependancies': [],
                    'id': 1,
                    'node_type': <QueryType.SINGLE_QUESTION: 'SINGLE'>,
                    'question': "Identify Jason's home country"},
                    {'dependancies': [],
                    'id': 2,
                    'node_type': <QueryType.SINGLE_QUESTION: 'SINGLE'>,
                    'question': 'Find the population of Canada'},
                    {'dependancies': [1],
                    'id': 3,
                    'node_type': <QueryType.SINGLE_QUESTION: 'SINGLE'>,
                    'question': "Find the population of Jason's home country"},
                    {'dependancies': [2, 3],
                    'id': 4,
                    'node_type': <QueryType.SINGLE_QUESTION: 'SINGLE'>,
                    'question': 'Calculate the difference in populations between '
                                "Canada and Jason's home country"}]} 
```

In the above code, we define a `query_planner` function that takes a question as input and generates a query plan using the OpenAI API.

## Conclusion

In this example, we demonstrated how to use the OpenAI Function Call `ChatCompletion` model to plan and execute a query plan using a question-answering system. We defined the necessary structures using Pydantic, created a query planner function. 

If you want to see multiple version of this style of code please visit 

1. [query planning example](https://github.com/jxnl/openai_function_call/blob/main/examples/query_planner_execution/query_planner_execution.py)
2. [task planning with topo sort](https://github.com/jxnl/openai_function_call/blob/main/examples/task_planner/task_planner_topological_sort.py)

Feel free to modify the code to fit your specific use case and explore other possibilities of using the OpenAI Function Call model to plan and execute complex workflows.