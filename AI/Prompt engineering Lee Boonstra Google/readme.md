# Prompt Engineering 

Author: Lee Boonstra

# intro (from [kaggle](https://www.kaggle.com/whitepaper-prompt-engineering))

When thinking about a large language model input and output, a text prompt (sometimes accompanied by other modalities such as image prompts) is the input the model uses to predict a specific output. You don’t need to be a data scientist or a machine learning engineer – everyone can write a prompt. However, crafting the most effective prompt can be complicated. Many aspects of your prompt affect its efficacy: the model you use, the model’s training data, the model configurations, your word-choice, style and tone, structure, and context all matters. Therefore, prompt engineering is an iterative process. Inadequate prompts can lead to ambiguous, inaccurate responses, and can hinder the model’s ability to provide meaningful output.
When you chat with the Gemini chatbot, you basically write prompts, however this whitepaper focuses on writing prompts for the Gemini model within Vertex AI or by using the API, because by prompting the model directly you will have access to the configuration such as temperature etc.
This whitepaper discusses prompt engineering in detail. We will look into the various prompting techniques to help you getting started and share tips and best practices to become a prompting expert. We will also discuss some of the challenges you can face while crafting prompts.

# Links

[Google Cloud Prompt Engineering Guide](https://cloud.google.com/discover/what-is-prompt-engineering?hl=en)
[AI Watch: Google Publishes 68-page Booklet on Prompt Engineering (For Using AI) ](https://www.discoveriesinhealthpolicy.com/2025/04/ai-watch-google-publishes-68-page.html?utm_source=chatgpt.com)
[Google’s Prompt Engineering Best Practices](https://www.prompthub.us/blog/googles-prompt-engineering-best-practices?utm_source=chatgpt.com)

# Description and resume

This whitepaper serves as an in-depth manual for crafting effective prompts when interacting with large language models (LLMs), particularly Google's Gemini via Vertex AI. It's also applicable to other models like GPT, Claude, and open-source alternatives.

Core Prompting Techniques

    Zero-shot, One-shot, and Few-shot Prompting: Utilizing varying numbers of examples to guide model responses.

    System and Role Prompting: Setting the model's behavior or persona to influence outputs.

    Contextual Prompting: Providing background information to enhance response relevance.

Advanced Techniques

    Chain-of-Thought (CoT) Prompting: Encouraging step-by-step reasoning.

    ReAct (Reason + Act) Prompting: Combining reasoning with actions, such as tool usage.

    Tree-of-Thought (ToT) Prompting: Exploring multiple reasoning paths before concluding.

    Step-Back Prompting: Prompting the model to consider general principles before specifics.

    Automatic Prompt Engineering (APE): Using the model to generate and refine its own prompts.

Model Configuration Guidance

The guide delves into configuring model parameters to optimize outputs:

    Temperature, Top-p, and Top-k: Adjusting these settings to balance creativity and determinism in responses.

    Output Length: Managing token limits to control response size and relevance.

Structured Output Techniques

Strategies for obtaining structured outputs, such as JSON formatting, are discussed, ensuring consistency and ease of parsing in applications.

Practical Applications

The whitepaper includes real-world examples and templates for tasks like:

    Summarization

    Classification

    Translation

    Code generation

    Data extraction

These examples are tailored for use with APIs and tools like Firebase Studio