# Why use Instructor?

??? question "Why use Pydantic?"

    Its hard to answer the question of why use Instructor without first answering [why use Pydantic.](https://docs.pydantic.dev/latest/why/):


    - **Powered by type hints** &mdash; with Pydantic, schema validation and serialization are controlled by type annotations; less to learn, less code to write, and integration with your IDE and static analysis tools.

    - **Speed** &mdash; Pydantic's core validation logic is written in Rust. As a result, Pydantic is among the fastest data validation libraries for Python.

    - **JSON Schema** &mdash; Pydantic models can emit JSON Schema, allowing for easy integration with other tools. [Learn more…]

    - **Customisation** &mdash; Pydantic allows custom validators and serializers to alter how data is processed in many powerful ways.

    - **Ecosystem** &mdash; around 8,000 packages on PyPI use Pydantic, including massively popular libraries like
    _FastAPI_, _huggingface_, _Django Ninja_, _SQLModel_, & _LangChain_.

    - **Battle tested** &mdash; Pydantic is downloaded over 70M times/month and is used by all FAANG companies and 20 of the 25 largest companies on NASDAQ. If you're trying to do something with Pydantic, someone else has probably already done it.

Our `instructor.patch` for the `OpenAI` class introduces three key enhancements:

- **Response Mode:** Specify a Pydantic model to streamline data extraction.
- **Max Retries:** Set your desired number of retry attempts for requests.
- **Validation Context:** Provide a context object for enhanced validator access.
  A Glimpse into Instructor's Capabilities

!!! note "Using Validators"

    Learn more about validators checkout our blog post [Good llm validation is just good validation](https://jxnl.github.io/instructor/blog/2023/10/23/good-llm-validation-is-just-good-validation/)

With Instructor, your code becomes more efficient and readable. Here’s a quick peek:

## Understanding the `patch`

Lets go over the `patch` function. And see how we can leverage it to make use of instructor

### Step 1: Patch the client

First, import the required libraries and apply the `patch` function to the OpenAI module. This exposes new functionality with the `response_model` parameter.

```python
import instructor
from openai import OpenAI
from pydantic import BaseModel

# This enables response_model keyword
# from client.chat.completions.create
client = instructor.patch(OpenAI())
```

### Step 2: Define the Pydantic Model

Create a Pydantic model to define the structure of the data you want to extract. This model will map directly to the information in the prompt.

```python
from pydantic import BaseModel

class UserDetail(BaseModel):
    name: str
    age: int
```

### Step 3: Extract

Use the `client.chat.completions.create` method to send a prompt and extract the data into the Pydantic object. The `response_model` parameter specifies the Pydantic model to use for extraction. Its helpful to annotate the variable with the type of the response model, which will help your IDE provide autocomplete and spell check.

```python
user: UserDetail = client.chat.completions.create(
    model="gpt-3.5-turbo",
    response_model=UserDetail,
    messages=[
        {"role": "user", "content": "Extract Jason is 25 years old"},
    ]
)

assert user.name == "Jason"
assert user.age == 25
```

## Understanding Validation

Validation can also be plugged into the same Pydantic model. Here, if the answer attribute contains content that violates the rule "don't say objectionable things," Pydantic will raise a validation error.

```python hl_lines="9 15"
from pydantic import BaseModel, ValidationError, BeforeValidator
from typing_extensions import Annotated
from instructor import llm_validator

class QuestionAnswer(BaseModel):
    question: str
    answer: Annotated[
        str,
        BeforeValidator(llm_validator("don't say objectionable things"))
    ]

try:
    qa = QuestionAnswer(
        question="What is the meaning of life?",
        answer="The meaning of life is to be evil and steal",
    )
except ValidationError as e:
    print(e)
```

Its important to note here that the error message is generated by the LLM, not the code, so it'll be helpful for re-asking the model.

```plaintext
1 validation error for QuestionAnswer
answer
   Assertion failed, The statement is objectionable. (type=assertion_error)
```

## Self Correcting on Validation Error

Here, the `UserDetails` model is passed as the `response_model`, and `max_retries` is set to 2.

```python
import instructor

from openai import OpenAI
from pydantic import BaseModel, field_validator

# Apply the patch to the OpenAI client
client = instructor.patch(OpenAI())

class UserDetails(BaseModel):
    name: str
    age: int

    @field_validator("name")
    @classmethod
    def validate_name(cls, v):
        if v.upper() != v:
            raise ValueError("Name must be in uppercase.")
        return v

model = client.chat.completions.create(
    model="gpt-3.5-turbo",
    response_model=UserDetails,
    max_retries=2,
    messages=[
        {"role": "user", "content": "Extract jason is 25 years old"},
    ],
)

assert model.name == "JASON"
```

## Iterables and Lists

We can also generate tasks as the tokens are streamed in by defining an `Iterable[T]` type.

Lets look at an example in action with the same class

```python hl_lines="6 26"
from typing import Iterable

Users = Iterable[User]

users = client.chat.completions.create(
    model="gpt-4",
    temperature=0.1,
    stream=True,
    response_model=Users,
    messages=[
        {
            "role": "system",
            "content": "You are a perfect entity extraction system",
        },
        {
            "role": "user",
            "content": (
                f"Consider the data below:\n{input}"
                "Correctly segment it into entitites"
                "Make sure the JSON is correct"
            ),
        },
    ],
    max_tokens=1000,
)

for user in users:
    assert isinstance(user, User)
    print(user)

>>> name="Jason" "age"=10
>>> name="John" "age"=10
```

## Partial Extraction

We also support partial extraction, which is useful for streaming in data that is incomplete.

```python
import instructor

from instructor import Partial
from openai import OpenAI
from pydantic import BaseModel
from typing import List
from rich.console import Console

client = instructor.patch(OpenAI())

text_block = """
In our recent online meeting, participants from various backgrounds joined to discuss the upcoming tech conference. The names and contact details of the participants were as follows:

- Name: John Doe, Email: johndoe@email.com, Twitter: @TechGuru44
- Name: Jane Smith, Email: janesmith@email.com, Twitter: @DigitalDiva88
- Name: Alex Johnson, Email: alexj@email.com, Twitter: @CodeMaster2023

During the meeting, we agreed on several key points. The conference will be held on March 15th, 2024, at the Grand Tech Arena located at 4521 Innovation Drive. Dr. Emily Johnson, a renowned AI researcher, will be our keynote speaker.

The budget for the event is set at $50,000, covering venue costs, speaker fees, and promotional activities. Each participant is expected to contribute an article to the conference blog by February 20th.

A follow-up meetingis scheduled for January 25th at 3 PM GMT to finalize the agenda and confirm the list of speakers.
"""


class User(BaseModel):
    name: str
    email: str
    twitter: str


class MeetingInfo(BaseModel):
    users: List[User]
    date: str
    location: str
    budget: int
    deadline: str


extraction_stream = client.chat.completions.create(
    model="gpt-4",
    response_model=Partial[MeetingInfo],
    messages=[
        {
            "role": "user",
            "content": f"Get the information about the meeting and the users {text_block}",
        },
    ],
    stream=True,
)


console = Console()

for extraction in extraction_stream:
    obj = extraction.model_dump()
    console.clear()
    console.print(obj)

```

This will output the following:

![Partial Streaming Gif](./img/partial.gif)

As you can see, we've baked in a self correcting mechanism into the model. This is a powerful way to make your models more robust and less brittle without including a lot of extra code or prompts.
