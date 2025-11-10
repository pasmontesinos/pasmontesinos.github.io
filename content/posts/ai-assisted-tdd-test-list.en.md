---
title: "The Test List as a Guide in AI-Assisted TDD"
slug: "ai-assisted-tdd-test-list"
translationKey: "ai-assisted-tdd-test-list"
date: 2025-01-10T12:00:00+01:00
draft: false
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
description: "How Kent Beck’s test list gains new relevance when practicing TDD with AI assistants, providing context and control throughout the development cycle."
author: "Pascual Montesinos"
---

# The Test List as a Guide in AI-Assisted TDD

## Introduction

The practice of **AI-assisted TDD** (Test-Driven Development supported by artificial intelligence) combines the core principles of TDD with the capabilities of large language models (LLMs).  
In this approach, the developer collaborates with an AI assistant during the development cycle, leveraging its ability to generate code and maintain the iterative rhythm characteristic of TDD.

One of the main challenges when working with AI assistants is preserving consistency and direction throughout the process. In this context, it becomes especially relevant to revisit a concept introduced by **Kent Beck** more than two decades ago: the **test list** as a guide for development.

## The Test List According to Kent Beck

In his book *Test Driven Development: By Example*, Kent Beck emphasizes the importance of maintaining a **to-do list** throughout the TDD process.  
This list helps maintain focus, record ideas that arise during development, and retain control over pace and direction.

Beck introduces this practice early in the book, using it to note pending tests, necessary refactorings, and ideas that emerge throughout the cycle.  
The rule is simple: maintain a list and use it to stay on track; if something interrupts your current flow, write it down and continue the task at hand.

In the context of AI-assisted development, this practice takes on renewed significance. The list not only helps maintain working context but also provides the assistant with a clear guide to the current development state and upcoming actions.

## A Prompt to Maintain Context

The following prompt structure helps keep an updated test list during each iteration of the TDD cycle:

```markdown
During the TDD cycle, the agent and the human will work with a **Test List** to maintain context and track progress.

It is ESSENTIAL to show the updated test list at every iteration so that the human can select the next test to implement.

**Test List:**
- Displayed as a numbered Markdown list.
- Each item follows the format:  
  `<number>. <status_icon> <TestClass>.test_should_<description> — <short goal>`
- Status icons:  
  - ✅ if the test is implemented and passing  
  - ❌ if the test is implemented but currently failing  
  - ⌛ if the test has not yet been implemented  
- Entries should be concise and incremental.  
- No code is shown, only the list.  
- The human will indicate the next test by its number.
```

## Example of Use

With this structure, a typical AI-assisted TDD session can be organized as follows:

```
Current test list:

1. ✅ TestDjangoORMProductRepository.test_should_save_product — persists a new product in the database
2. ❌ TestDjangoORMProductRepository.test_should_find_existing_product — retrieves a previously saved product
3. ⌛ TestDjangoORMProductRepository.test_should_return_none_when_product_not_found — returns None if it does not exist

Suggested commit G: implement Django product repository save. Should the commit be applied now?

Next natural step: 1) implement test 2 for retrieval; 2) then cover the absence case (test 3).
```

## Observed Benefits

### 1. **Process Control**
The visual list provides a clear overview of progress and pending work, enabling informed decisions on what to address next.

### 2. **Efficient Communication**
Instead of detailed explanations about the next step, simply stating “implement test 2” conveys full context and intent to the assistant.

### 3. **Context Recovery**
When a session resumes after an interruption, the list allows both developer and assistant to quickly reestablish the current state of development.

### 4. **Strategic Decisions**
The list makes it easier to prioritize based on complexity, available time, or dependencies between functionalities.

## Final Thoughts

AI-assisted TDD represents a natural evolution of traditional TDD, preserving its fundamental principles while leveraging the capabilities of modern LLMs.  
The test list serves as a bridge between the discipline of TDD and the generative potential of artificial intelligence.

This practice helps retain control over the development process while increasing productivity and consistency.  
As Kent Beck reminded us, the principle remains the same: **don’t forget your test list.**
