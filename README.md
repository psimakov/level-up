# Level Up: An Agentic LLM System for System Design Interview Practice

**Level Up** is an interactive, AI-powered system that helps you prepare for system design interviews. It simulates a realistic interview experience, giving you a structured environment to practice, refine your skills, and receive consistent feedback.

## Available Challenges

The best way to prepare for an interview is to **practice in real-time using ChatGPT’s live voice mode** -- it feels just like sitting across from an interviewer. It's **free**, but you may run out of voice quat quickly... You can start immediately with our well-known interview challenges, designed to sharpen your skills under realistic conditions:

* [**Design a Rate Limiter**](https://chatgpt.com/g/g-68c2229f7a308191bf87fea2e1e37002-level-up)

## How It Works

The system is built around a structured workflow that ensures high-quality, consistent interview practice:

1. **`<expert />` Design** – A subject-matter expert creates a detailed interview `<challenge />`, outlining a specific system design problem.
2. **`<curator />` Assistance** – An LLM agent, the `<curator />`, collaborates with the expert to produce a comprehensive `<cheat-sheet />`. This document highlights correct and incorrect solutions, trade-offs, and common pitfalls.
3. **`<coach />` Simulation** – Another LLM agent, the `<coach />`, uses the `<cheat-sheet />` to act as your `<interviewer />`. It presents the challenge, asks follow-up questions, and evaluates your performance in a simulated interview session.

This agent-based approach ensures every practice session is consistent, thorough, and aligned with expert-defined criteria.

## Adding a New Challenge

Extending the system with new interview questions is simple. Experts can contribute by creating additional `<challenge />` dossiers. The system provides specialized tools to streamline this process:

* `<check-challenge />` – Validates the completeness of a new challenge.
* `<make-cheat-sheet />` – Automatically generates a structured cheat sheet from challenge content.
* `<check-prompts />` – Ensures the integrity and consistency of the entire agentic system.

## Author

Developed by [**Pavel (Pasha) Simakov**](https://softwaresecretweapons.com/) using Gemini CLI and ChatGPT.