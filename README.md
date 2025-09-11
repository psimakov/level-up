# Level Up: An Agentic LLM System for System Design Interview Practice

**Level Up** is an interactive, AI-powered system that helps you prepare for system design interviews. It simulates a realistic interview experience, giving you a structured environment to practice, refine your skills, and receive consistent feedback.

## Available Challenges

The best way to prepare for an interview is to **practice in real-time using ChatGPT’s live voice mode** -- it feels just like sitting across from an interviewer. Practice now with our well-known interview challenges:

* *medium* difficulty
    * [**Design A Rate Limiter**](https://chatgpt.com/g/g-68c2229f7a308191bf87fea2e1e37002-level-up)
    * [**Design FB News Feed**](https://chatgpt.com/g/g-68c2371e2e608191992474be4ced2a2f-level-up-design-social-news-feed)
* *hard* difficulty
    * [**Design Ad Click Aggregator**](https://chatgpt.com/g/g-68c237fe2fc08191ab80bbea5a0ddc19-level-up-design-ad-click-aggregator) 
    * [**Design YouTube**](https://chatgpt.com/g/g-68c2388e17a48191be8ce66e4fa66455-level-up-design-video-upload-streaming-service)
    * [**Design URL Shortener**](https://chatgpt.com/g/g-68c2395859048191a66e713f595b692c-level-up-design-url-shortener)

> It’s **live now** and **free to use** -- available in **ChatGPT** web or mobile app.

## How It Works

The system is built around a structured workflow that ensures high-quality, consistent interview practice:

1.  **`<expert />` Design** – A subject-matter expert creates a detailed interview `<challenge />`, outlining a specific system design problem.
2.  **`<curator />` Assistance** – An LLM agent, the `<curator />`, collaborates with the expert to produce a comprehensive `<cheat-sheet />`. This document highlights correct and incorrect solutions, trade-offs, and common pitfalls.
3.  **`<coach />` Simulation** – Another LLM agent, the `<coach />`, uses the `<cheat-sheet />` to act as your `<interviewer />`. It presents the challenge, asks follow-up questions, and evaluates your performance in a simulated interview session.

This agent-based approach ensures every practice session is consistent, thorough, and aligned with expert-defined criteria.

The agentic behavior is defined by a set of prompts stored in the [`./prompts`](./prompts) directory:

*   [`system.md`](./prompts/system.md): Defines the roles of all actors (`<expert />`, `<curator />`, `<coach />`), the assets they produce (`<challenge />`, `<cheat-sheet />`), and the tools available to them.
*   [`curator.md`](./prompts/curator.md): Contains the instructions for the `<curator />` agent, guiding it to help the `<expert />` produce a high-quality `<cheat-sheet />`.
*   [`coach.md`](./prompts/coach.md): Provides the script for the `<coach />` agent to simulate a realistic `<interviewer />`.

The output of the `<curator />` is a `<cheat-sheet />`, which is stored in the [`./cheat-sheets`](./cheat-sheets) directory. For example, [`ad-click-aggr.md`](./cheat-sheets/ad-click-aggr.md) contains a detailed guide for the `<coach />` on how to conduct an interview for the "Design Ad Click Aggregator" challenge.

## Adding a New Challenge and Creating `<cheat-sheet />`

To add a new challenge, follow these steps:

1.  **Create a Challenge File**: Add a new YAML file (e.g., `my-challenge.yaml`) to the `./challenges` directory.
2.  **Populate the Challenge**: Fill the file with comprehensive details about the challenge. You can source this information from online articles, books, or other resources.
3.  **Use Gemini CLI Tools**:
    *   Run `<check-challenge />` to validate the challenge's completeness.
    *   Execute `<make-cheat-sheet />` to automatically generate a structured `<cheat-sheet />` from the challenge.

## Publishing a New Challenge as a Custom GPT

Publishing a new challenge as a Custom GPT is easy:

1.  Go to [**chatgpt.com/gpts**](https://chatgpt.com/gpts) and click the **"Create"** button.
2.  Paste the content of [`agents/customChatGPT.md`](./agents/customChatGPT.md) into the **Instructions** field.
3.  Rename the cheat sheet file for your challenge (e.g., `my-challenge.md`) to `challenge.md`.
4.  Upload `coach.md` and `challenge.md` to the **Knowledge** section.
5.  Test your new GPT and publish it.

Open your new GPT and start chatting with it—either by text or, even better, by **voice**!

## Author

Developed by [**Pavel (Pasha) Simakov**](https://softwaresecretweapons.com/) using Gemini CLI and ChatGPT.