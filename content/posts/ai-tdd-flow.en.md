---
title: "Maintaining the discipline in AI-Assisted TDD"
date: 2025-11-14T08:00:00+01:00
draft: false
slug: "ai-assisted-tdd-flow"
translationKey: "ai-assisted-tdd-flow"
tags:
  - TDD
  - AI-Assisted-Development
  - Test-Driven-Development
  - LLM
  - Software-Development
  - Best-Practices
categories:
  - Development
  - Testing
  - AI
description: "How to practice Test-Driven Development with AI assistants while maintaining TDD discipline and leveraging LLM capabilities"
author: "Pascual Montesinos"
---

After several months experimenting with AI-assisted TDD, I've refined a workflow that allows me to combine the strict discipline of traditional TDD with the capabilities of AI assistants. It's not about accelerating the process at any cost, but about maintaining quality and control while the agent handles the more mechanical tasks.

## The challenge of staying on track

One of the main problems when working with AI assistants in TDD is maintaining process discipline. The temptation is real: letting the agent generate both tests and implementation in one go seems efficient. But by doing so, we lose the fundamental benefits of TDD. We lose the emergent design guided by tests, the minimal necessary implementation, and that confidence the process gives you when you need to refactor.

LLMs are extraordinarily capable of generating complete and well-structured code. But that very capability becomes a trap if we don't establish clear boundaries. Code that "works on the first try" often anticipates functionality we don't yet need, includes premature abstractions, or simply does more than the current test asks for.

That's why a workflow that maintains these characteristics of classic TDD while leveraging what assistants do well is crucial: generating bounded code, following established patterns, and executing the most repetitive tasks.

## Clear separation of responsibilities

The workflow is based on a simple premise: clearly defined roles between the developer and the AI agent. I, as the developer, maintain control over decisions. I decide which test to implement next, whether the proposed implementation is truly minimal or is anticipating functionality, whether a refactor is appropriate or not the right time.

The agent, on the other hand, is responsible for proposing test code following the specifications I give, implementing the minimal solution under my explicit direction, running the test runner and reporting results, identifying refactoring opportunities once we're green, and suggesting commit messages that follow semantic conventions.

This separation is not arbitrary. It replicates the natural flow of pair programming, where one writes code while the other reviews and guides. The difference is that the agent has a much greater code generation capacity than a human, but also a natural tendency to over-implement if not firmly guided.

## Starting with a test list

Before entering the TDD cycle proper, the agent proposes a test list that we refine and that serves as a map during the development session. This practice, which I detail in [my previous post about the test list](https://pasmontesinos.com/posts/ai-assisted-tdd-lista-de-tests/), is fundamental to maintaining context and development direction.

For example, when I start implementing a product repository with Django ORM, the initial list might look like this:

```
Test List - ProductRepository
==============================
1. ⌛ test_should_save_product
2. ⌛ test_should_find_existing_product
3. ⌛ test_should_return_none_when_product_not_found
```

This list constantly evolves. We discover new cases we hadn't considered, eliminate tests that turn out to be redundant, adjust priorities as we learn more about the domain. But the important thing is that we always know where we are and where we're going.

## Configuring the agent: the AGENTS.md file

The key to maintaining discipline is to explicitly document the workflow. In each project, I include an `AGENTS.md` file at the root with the precise instructions the agent must follow during the TDD cycle:

```markdown
## Strict TDD workflow (with agent)

**1. Test Selection**
- The agent shows the updated test list and waits for the human to choose which test to implement.

**2. Red Phase**
- The agent proposes the corresponding test code and the human accepts it or proposes changes.
- Once accepted, the agent modifies the code (without comments).
- The agent runs the tests with the command specified in the repository context and verifies that it **fails for the expected reason**.
- If the failure is not the expected one, the agent adjusts the test until achieving the correct failure.
- **No commit is made at this phase.**

**3. Green Phase**
- The agent proposes the **minimal implementation** necessary for the test to pass, and the human accepts it or proposes changes. It is VERY IMPORTANT to implement only what's necessary to pass the current test and not anticipate future tests.
- Once accepted, the agent modifies the code (without comments).
- The agent runs the tests to verify that **all tests pass correctly**.
- If any test fails, the agent proposes corrections until all pass.
- The agent proposes a commit with the message `<feat|chore|test|refactor|...>: <brief description in English in one line>` with the command specified in the repository context and asks: *"Should I make the commit now?"*

**4. Refactor Phase**
- The agent proposes **safe refactoring improvements** (without changing observable behavior).
- Once accepted by the human, the agent modifies the code (without comments).
- The agent runs the tests and confirms that all continue passing.
- If any test fails, the agent proposes corrections until all pass.
- The agent proposes a commit with the message `<feat|chore|test|refactor|...>: <brief description in English in one line>` with the command specified in the repository context and asks: *"Should I make the commit now?"*

The cycle repeats until the functionality is complete. It is essential to show the updated list in each iteration to maintain context of the progress.
```

## The cycle: Red, Green, Refactor

Now let's look at each phase in more depth with concrete examples of how it works in practice.

### Conscious test selection

The process always begins with a conscious selection of the next test to implement. The agent presents the updated list and I choose which one to tackle. This decision is not trivial: I consider incremental complexity, dependencies between tests, etc. It's not simply about doing "the next one" on the list, but choosing the right test at the right time.

When I tell the agent "let's implement test 1," we're establishing a contract: we're going to focus on this specific functionality, no more, no less.

### Red phase: the right failure

The agent proposes the test code. For example:

```python
def test_should_save_product(self) -> None:
    repository = DjangoORMProductRepository()
    product = Product(
        id="product-1",
        name="Laptop",
        price=Decimal("999.99"),
        stock=10
    )
    
    repository.save(product)
    
    product_model = ProductModel.objects.get(id=product.id)
    assert product_model.id == product.id
    assert product_model.name == product.name
    assert product_model.price == product.price
    assert product_model.stock == product.stock
```

I review the test. Is it well-structured? Are the assertions correct? If something doesn't convince me, we adjust it. Once approved, the agent runs the test.

Here comes something critical: the test must fail for the right reason. If it fails because the `save()` method doesn't exist, that's the expected failure. But if it fails due to a syntax error, a missing import, or a configuration problem, we're not really red. We're in an invalid intermediate state. The agent adjusts until we get the expected failure.

### Green phase: the temptation to do more

Once we have the test failing for the right reason, it's time for implementation. And this is where discipline is put to the test. I ask the agent to propose the minimal implementation to pass the test.

The agent proposes something like:

```python
class DjangoORMProductRepository:
    def save(self, product: Product) -> None:
        ProductModel.objects.create(
            id=product.id,
            name=product.name,
            price=product.price,
            stock=product.stock
        )
```

My job is to review that it's truly minimal. Is it only implementing the `save()` method? Is it not adding `find()` or `search()` methods anticipating future tests? Is it not adding validations we haven't tested yet? If the implementation is truly minimal, I give the green light.

The agent modifies the code and runs all tests. Not just the new one, all of them. This is another fundamental practice: every change must be verified against the entire test suite to ensure we haven't broken anything existing.

Once all tests pass, the agent proposes a commit: "feat: implement save method in ProductRepository". I review the message, confirm it follows conventional commits conventions, and we approve. At that moment, the agent validates types and format before committing. Now yes, the code is in the repository.

### Refactor phase: improving with confidence

With tests green, it's time to ask ourselves if there are opportunities for improvement. The agent identifies possible refactorings: names that could be clearer, duplication that could be extracted, structure that could be simplified, etc. I decide if it's the right time to apply these changes or if they should wait.

If it makes sense to do it now, the agent applies the changes and re-runs all tests. If something breaks, the tests tell us immediately. If everything stays green, we make another commit: "refactor: extract domain model conversion to private method".

This confidence to refactor is one of the main benefits of TDD. We're not guessing if our changes break something. Tests give us the answer immediately and objectively.

## The emerging rhythm

After several cycles, a natural rhythm emerges. The test list gets updated:

```
Test List - ProductRepository
==============================
1. ✅ test_should_save_product
2. ❌ test_should_find_existing_product
3. ⌛ test_should_return_none_when_product_not_found
```

Each symbol tells a story. Completed tests (✅) give us confidence. The red test (❌) shows us where we are. Pending ones (⌛) remind us where we're going. This visual feedback is surprisingly useful.

## What I've gained from this workflow

Process control remains completely in my hands. I decide what to implement and when. The agent executes, but I direct. This is fundamental because business context, product priorities, technical trade-offs, all of that requires human judgment that I don't want to leave in the agent's hands.

Implementation is truly incremental. By forcing minimal implementation in each cycle, design emerges without over-engineering. I'm not building for a hypothetical future, I'm building for real and present needs.

Commits are atomic and meaningful. Each cycle produces one or two small, focused commits. When I need to rollback, I know exactly what I'm reverting. When a colleague reviews the code, they can understand the logical progression of development. The Git history becomes documentation of the design process.

And there's an unexpected benefit: I learn continuously. By reviewing the agent's proposals, I discover new patterns, different ways to structure code, conventions I didn't know. The agent becomes a learning partner, not just a productivity tool.

## Integrating with the team

This workflow is not meant to work in isolation. In fact, it works especially well in collaborative contexts. In pair programming, one can handle the agent while the other reviews and suggests direction. In mob programming, the agent becomes one more tool of the mob, where the driver operates the agent under the group's guidance.

## Final reflections

I've come to understand that AI-assisted TDD is not about writing code faster. It's about maintaining quality while delegating the mechanical. The agent is exceptional at generating small code fragments, following established patterns, executing repetitive commands. But the criteria, design decisions, understanding of the business domain, all of that remains the developer's responsibility.

The key is not to give up control of the process. The agent is a powerful tool, probably the most powerful tool I've used in years of development. But it remains just that: a tool. The direction, the decisions, the "why" behind the code, that remains human territory.

And perhaps most importantly: this workflow allows me to maintain that sense of code ownership. When something goes to production, I understand every decision that was made. I can explain why it's structured this way, what alternatives were considered, what trade-offs were accepted. It's not code that an agent generated and I blindly approved. It's code that I designed, with an assistant that helped me materialize it.
