---
layout: post
title:  "LLM-Oriented Programming: Keeping Your Codebase Organized for Large Language Models"
date:   2023-04-04 00:00:00
categories: programming
---

# Introduction

I feel that the world of coding is changing. In a couple of years, I expect a lot of new developer tools based on LLMs to come up. They will likely change the way we write code. And maybe even change the definition of what we think is Software Developer. With the release of GPT-4, the ways you can use it in daily programming activities is impressive.

As with any tool, developers should learn how to make a better use of it. While working with LLMs you may notice that some codebases are suited better for them. As powerful as these LLM models are, they have their limitations. One of the primary constraints is their inability to maintain context over long input prompts. This can lead to confusion and misinterpretation when working with large codebases.

More and more people start to think that SDEs will be replaced. My take is that our job might be transformed, but not replaced. More and more developers will likely to be more like software architects and team leads, using LLMs to solve partial tasks and combining final results. LLMs are also pretty bad at correctness and validation, which will most likely remain an important part of our job.

This raises a question: how to write code to be better understood by language models? Let's dive into LLM-oriented programming.


<!--more-->


The idea is to keep your codebase organized in such a way that most parts of it should always fit into LLM message context or, at most, up to 3 messages. By doing so, you enable LLMs like GPT to maintain context more effectively and engage with your codebase more accurately.

In this blog post, we will explore the concept of LLM-oriented programming, discuss its benefits, and provide some guidelines on how to implement it effectively in your projects.

# The Story of two Developers

Is all code equal for an LLM? When observing how different developers work with LLMs on their codebases, I have noticed that this is not true. You can be mindful about writing your code in a way that will make it more readable not only for humans, but for LLMs as well.

To illustrate the importance of structuring your code in an LLM-friendly way, let's consider the story of two developers.

The first one is a skilled programmer who has been working with GPT for quite some time. She understands the limitations of the model and knows that keeping the codebase organized and modular is critical for the AI's performance. As a result, she spends a significant amount of time refactoring code and keeping it well-structured, ensuring that GPT can understand and interact with the codebase effectively. She also tries to keep her code very modular and concise, so that most of the parts can be analyzed by LLM in a single message.

On the other hand, the second developer is relatively new to working with LLMs. He is an experienced programmer but has never considered the importance of organizing his codebase with LLMs in mind. His code is often long and convoluted, making it difficult for GPT to maintain context and generate accurate responses.

First developer will extract most value from using LLMs, and will have a significant productivity boost from using LLM-aided programming techniques.

# Guidelines for Implementing LLM-Oriented Programming

Surprisingly, to effectively implement LLM-oriented programming, you should follow common best practices and write clean code. Making it more readable and maintainable by humans will help LLMs as well. Next, let's see a list of things to keep in mind when writing code with LLMs in mind. Some points are extracted from my experience and conversations with my colleagues. In addition, I have decided to ask GPT-4 to hint on how to write code better for it to understand. Here is the final compilation:

## Keep it modular

Break your codebase down into smaller, self-contained modules or microservices. This makes it easier for LLMs to maintain context and engage with your code effectively.

## Emphasize tests
LLM outputs are not reliable. And it's quite error-prone to validate outputs manually due to productivity boost it gives you. So, we should put extra emphasis on automated testing and code validation. You can use LLM to aid you in writing tests, but make sure to write a lot of them so that any changes to your code can be quickly validated.

## Limit dependencies
Use of many dependencies, especially lesser-known libraries, will make it harder for LLM to understand what the code does.

## Be mindful of abstraction levels
When designing your codebase with LLM-oriented programming in mind, it's important to consider the abstraction levels of your code. Strive to create well-defined, self-contained components that encapsulate a single aspect of the overall functionality. This not only helps in keeping the codebase modular, but also makes it easier for LLMs to understand the code and provide relevant suggestions.

## Use clear and concise naming conventions
Good naming conventions make your code more readable and easier to understand for both humans and language models. Use descriptive names for functions, classes, and variables that reflect their purpose and functionality. By doing this, you make it easier for the LLM to understand the context and provide more accurate suggestions or insights.

## Implement comprehensive comments and documentation
A well-documented codebase is not only beneficial for human developers but also for LLMs. Detailed comments and documentation help LLMs grasp the context, purpose, and intricacies of your code. This, in turn, allows them to provide more relevant and accurate suggestions, as they have a better understanding of the codebase.

## Consider using code patterns and idioms
Adopting common code patterns and idioms can make your code more predictable and easier to understand for both humans and LLMs. By leveraging familiar patterns, you allow the LLM to recognize and understand the intent behind your code quickly, leading to better assistance and suggestions.


# Iterate and refine
LLM-oriented programming is an evolving practice. As you work with LLMs and incorporate their assistance into your development process, you'll likely discover new ways to optimize your codebase and improve LLM comprehension. Be open to experimentation and iteration to find the best practices for your specific use case.

In conclusion, LLM-oriented programming is an approach that challenges the way we code. It can help us to improve codebases and development processes. By organizing your codebase in a way that fits within the LLM's message context, you can make the most of the insights and suggestions provided by these models. Embracing testing, modularity, clear naming conventions, comprehensive documentation, and common code patterns are just a few ways to make your codebase more LLM-friendly. As you experiment and iterate, you'll discover even more ways to harness the power of LLMs to enhance your development process.

P.S. This post was co-written with Chat GPT by processing some conversations I had with different people regarding the post topic. Some manual editing and adding more points to the general article structure were done manually by me.

