---
title: 'Vibe Coding: A 10-Day Journey from Zero to Building a Full-Stack RSS Validator Tool'
date: 2025-04-19 23:08:11+09:00
tags: [Deno, JavaScript, TypeScript, Agentic Coding, AI]
---

10 days (2025/4/8 to 2025/4/18), From zero to https://kamusis-my-opml-sub.deno.dev/

The code implemented in the entire project so far includes backend and some frontend by Claude 3.7 Sonnet (sometimes Claude 3.5), while a larger portion of the frontend is by OpenAI GPT-4.1 (in Windsurf, this model is currently available for free for a limited time).

Project URL: [https://kamusis-my-opml-sub.deno.dev/](https://kamusis-my-opml-sub.deno.dev/)

## User Story
I’ve been using RSS for like… 15 years now? Over time I’ve somehow ended up with 200+ feed subscriptions. I know RSS isn’t exactly trendy anymore, but a handful of these feeds are still part of my daily routine.

The problem? My feed list has turned into a total mess:
- Some feeds are completely dead
- Some blogs haven’t been updated in years
- Others post like once every six months
- And a bunch just throw 404s now

I want to clean it up, but here’s the thing:
Going through each one manually sounds like actual hell.
My reader (News Explorer) doesn’t have any built-in tools to help with this.
I tried Googling things like “rss feed analyze” and “cleanup,” but honestly didn’t come across any useful tools.

So the mess remains… because there’s just no good way to deal with it. Until I finally decided to just build one myself—well, more like let AI build it for me.

## Background of Me

- Can read code (sometimes need to rely on AI for interpretation and understanding.)
- Have manually written backend code in the past, but haven't written extensive backend code in the last twenty years.
- Have never manually written frontend code and have limited knowledge of the basic principles of frontend rendering mechanisms.
- Started learning about JavaScript and TypeScript a month ago. 
- A beginner with Deno. Understand the calling sequence and respective responsibilities from components to islands to routes API, then to backend services, and finally to backend logic implementation.

## Tools

- Agentic Coding Editor (Windsurf)
- Design and Code Generater LLM (Claude 3.5/3.7 + openAI GPT-4.1)
We need a subscription to an Agentic Coding Editor, such as Cursor, Windsurf, or Github Copilot, for design and coding.
- Code Reviewer LLM (Gemini Code Assist)
Additionally, we need Gemini Code Assist (currently considered free) to review code and consult on any code-related questions. Gemini Code Assist is also very effective, and it can be said that Gemini is the best model to help you understand code.
- MCP Server (sequential-thinking)

## Process

1. **Design Phase**
   - Write the design and outline original requirements
   - Let AI write the design (experience shows Claude 3.5 + sequential-thinking MCP server works well; theoretically, any LLM with thinking capabilities is better suited for overall design)
   - Review the design, which should include implementation details such as interaction flow design, class design, function design, etc.
   - If you are trying to develop a full-stack application, you should write design documents for both frontend and backend
   - Continue to ask questions and interact with AI until you believe the overall design is reasonable and implementable (This step is not suitable for people who have no programming knowledge at all, but it is very important.)

2. **Implementation Planning**
   - Based on the design, ask AI to write an implementation plan (Claude 3.5 + sequential-thinking MCP server)
   - Break it down into steps
   - Ask AI to plan steps following a senior programmer's approach
   - Review steps, raise questions until the steps are reasonable (This step is not suitable for people who have no programming knowledge at all, but it is very important.)

3. **Implementation**
   - Strictly follow the steps
   - Ask AI to implement functions one by one (Claude 3.5/3.7)
   - After each function is implemented, ask AI to generate unit tests to ensure they pass

4. **Oversee**
   - If you have no programming experience, you might not be able to understand what the AI is doing or identify potential risks. As a result, you wouldn’t be able to oversee the AI or question its output, and would have to hope the AI makes no mistakes at all. This could make the implementation process much harder down the line.
   - Ensure strict monitoring of what AI is actually doing
   - For example: AI might implement underlying function calls in test cases rather than generating test cases for the target file, which would make it appear that tests pass when in fact there is no effective testing of the target file
   ![20250418171722](https://s2.loli.net/2025/04/18/U7QKqbAi4psHe5k.png)
   - Sometimes AI will take the initiative to use mocks for testing; we need to know when it's appropriate to use mocks in tests and when to test real functionality
   - This requires us to know whether we're doing Integration/Component Testing or Pure Unit Testing
   ![20250418171748](https://s2.loli.net/2025/04/18/NY75BW2golesZ1z.png)

5. **Code Review and Design Update**
   - Ask another AI to read the generated code (experience shows Gemini Code Assist is very suitable for this work)
   - Compare with the original design
   - Have AI analyze whether the original design has been fully implemented; if not, what's missing
        - Evaluate missing content and decide whether to implement it now
   - Or whether functionality beyond the design has been implemented
        - Evaluate functionality beyond the design and decide whether to reflect it back into the design
        - Why update the design? Because subsequent work may need to reference the design document, so ensuring the design document correctly reflects the code logic is a good practice
        - You don't necessarily need to document every single implementation detail (like the specific batch size in batchValidate), but changes to public interfaces and communication protocols are definitely worth updating.
    ![20250418171854](https://s2.loli.net/2025/04/18/lDY7tNzLk3jyqbe.png)


6. **Continuous Review**
   - After completing each requirement, ask AI to review the design document again to understand current progress and what needs to be done
   ![20250418171956](https://s2.loli.net/2025/04/18/SGCFPs16RjNYW2w.png)
   ![20250418172014](https://s2.loli.net/2025/04/18/4ruVdyUvfnXpRkx.png)
   ![20250418172023](https://s2.loli.net/2025/04/18/qFpkw1RolUBz9AK.png)

   - When major milestones are completed or before implementing the next major task, have AI review the completed work and write a new development plan
   - Always read the development plan completed by AI and make manual modifications if necessary
   ![20250418172046](https://s2.loli.net/2025/04/18/hDo2LBbfCGdSPAn.png)

   - After reaching a milestone, have AI (preferably a different AI) review progress again
   ![20250418172133](https://s2.loli.net/2025/04/18/hbxZ1vydXm4fSjW.png)

Repeat the above steps until the entire project is completed.

## Learning from the Project
### Git and GitHub
- Make good use of git; commit after completing each milestone functionality
- When working on significant, large-scale features—like making a fundamental data structure change from the ground up—it’s safer to use GitHub PRs, even if you’re working solo. Create a issue, create a branch for this issue, make changes, test thoroughly, and merge after confirming everything is correct.

### Debugging
When debugging, this prompt is very useful: "Important: Try to fix things at the cause, not the symptom." We need to adopt this mindset ourselves because even if we define this rule in the global rules, AI might still not follow it. When we see AI trying to fix a bug with a method that treats the symptom rather than the cause, we should interrupt and emphasize again that it needs to find the cause, not just fix the symptom. This requires us to have debugging skills, which is why Agentic Coding is currently not suitable for people who have no programming knowledge at all. Creating a familiar Snake game might not require any debugging, but for a real-world software project, if we let AI debug on its own, it might make the program progressively worse.

The sequential-thinking MCP server is very useful when debugging bugs involving multi-layer call logic. It will check and analyze multiple files in the call path sequentially, typically making it easier to find the root cause. Without thinking capabilities, AI models might not have a clear enough approach to decide which files to check.
![20250418172250](https://s2.loli.net/2025/04/18/pazwuMm18lTEyAQ.png)
![20250418193434](https://s2.loli.net/2025/04/18/5iWpbQ4wLzHqAcv.png)

For completely unfamiliar code sections, if bugs occur, we can only rely on AI to analyze and fix them itself, which significantly increases the frequency of interactions with AI and the cost of using AI. For example, when debugging backend programs, the Windsurf editor spends an average of 5 credits because I can point out possible debugging directions; but once we start debugging frontend pages, such as table flickering during refresh that must be fixed by adjusting CSS, because I have almost no frontend development experience, I have no suggestions or interventions, resulting in an average of 15 credits spent. When multiple modifications to a bug have no effect, rolling back the changes to the beginning stage of the bug and then using the sequential-thinking tool to think and fix will have better results.
![20250418193823](https://s2.loli.net/2025/04/18/alR9YzpLygPBONq.png)

### Refactoring
Refactoring is often essential because we don't review every line of AI-generated code, so we might miss some errors made by the AI. For example, in my project, when implementing a feature, the AI didn't use the interface previously defined in types.d.ts, but instead created a new interface with a similar name based on its understanding, and continued using this new interface throughout the feature implementation. After discovery, refactoring was necessary.
![20250418193509](https://s2.loli.net/2025/04/18/TkjuXLp6hPmAoJM.png)
![20250418193659](https://s2.loli.net/2025/04/18/P4fBlqVZQ7Ht2vK.png)
![20250418193708](https://s2.loli.net/2025/04/18/m8FJnfNsTbyQh19.png)

### Multi-Model mutual argumentation
When an AI offers suggestions and you’re unsure about them, a solid learning trick is to run those ideas by another AI for a second opinion. Take, for example, deciding if an endpoint should be defined with POST or GET.
I had Claude 3.7 whip up some code, then passed it over to Gemini for a quick check. Gemini suggested switching to GET, saying it might align better with common standards.
![20250418194455](https://s2.loli.net/2025/04/18/Kd8O62HauxbGItl.png)

When sending the suggestion back to Claude 3.7, Claude 3.7 still believed using POST was better.
![20250418194505](https://s2.loli.net/2025/04/18/RA1uqhJ3MGFTE5r.png)

When sending Claude 3.7's reply back to Gemini, Gemini agreed.
![20250418194515](https://s2.loli.net/2025/04/18/BYc7FXRm1Gh6b5T.png)

This is a fascinating experience, like being part of a team where you watch two experts share their opinions and eventually reach a consensus.

I hope in the future there will be a more convenient mechanism for Multi-Model mutual argumentation (rather than manual copy-pasting), which would greatly improve the quality of AI-generated code.



