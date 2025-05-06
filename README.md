# Natural Language Understanding Design for In-Car Voice Assistant

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)]
(https://colab.research.google.com/github/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/In_car_voice_assistant_NLU_module.ipynb)

## Acknowledgment
I would like to thank the following individuals and organizations that made this project possible.
•	Groq for providing me free access to their API key and thereby allowing me to gain hands-on experience in making API calls without having to constantly worry about token limits.

## Abstract
Traditional in-car voice assistant NLU modules built on classical seq2seq models face several limitations. These include the manual effort required to maintain extensive glossaries of world knowledge with incomplete coverage, the need for massive amounts of training data to handle natural language effectively, and slow iteration cycles that take hours, if not, days to train, evaluate, and test. Furthermore, these models often struggle to generalize, requiring repeated tweaking to handle specific utterances. Attempting to unify conflicting intent-slot schemas across multiple car brands within a single model typically results in subpar performance. Some brand-specific customizations, by overriding model defaults to align with proprietary intents and slots, often fail to yield reliable results. Finally, enumerating every possible intent-slot combination leads to bloated, unreadable specifications that remain incomplete and difficult to maintain.
This project explores a solution using a large language model (LLM)-based assistant, which aims to overcome these challenges through prompt-based intent classification and slot extraction, offering greater flexibility, scalability, faster iteration, and better adaptability across brands.

## Development Notes
### Guide, Not Teach
Large language models (LLMs) already come with the knowledge of natural languages. The task for NLU developers is to pull out that innate knowledge in LLMs and channel it in a certain direction. Fine-tuning a model is expensive, and the good news is that a lot can be done with prompt engineering—which is cheaper and faster. The languages, the model already knows. What it needs is guidance on how to classify intents and slots found in utterances in ways that suit brand-specific, custom classification systems.

Instead of an exhaustive list of intent-slot combinations for each domain, the model is given:

- A list of domains
- A list of intents and slots, each with a short description
- A single annotated example

The model breaks down the utterance into atomic segments (each containing an action or a piece of information), classifies the utterance into a domain, and tags the segments with the most contextually appropriate intent and slot.

The role of an in-car voice assistant is to distill a wide variety of human utterances into a finite set of direct commands. LLMs are excellent interpolators. Given just the intent name, slot list with descriptions, and a few examples, they generalise to novel, valid combinations without exploding the schema size.

### Efficiency Gains:
- No need to maintain an exhaustive list of intent-slot combinations
- Robust against unsupported combinations (filtered later by the grounding module)
- Encourages modular architecture: 
  - The **NLU module** handles recognition
  - The **grounding module** handles enforcement of brand capabilities
- Multilingual flexibility with a single-source prompt

This model capability is supported by Anthropic’s 2024 paper, *Tracing the Thoughts of a Large Language Model*, which shows that LLMs build a language-agnostic understanding of concepts and rephrase them in any language.

---

### Domain-Intent Segregation

An utterance is first classified into a **domain** by a domain classifier, then into an **intent-slot** pair by a domain-specific classifier. This separation enables domain-specific prompts tailored for accurate, specialised tagging.

Multi-intent utterances are broken into sub-utterances, each containing one intent-slot pair, and processed in parallel by the relevant domain-specific classifiers. This routing mechanism ensures efficient and scalable handling of compound user commands.

---

### Clear Reasoning Leads to Better Classification

I added a `rationale` field to the model's output, alongside a confidence score. Asking the model to explain its reasoning for domain, intent, and slot choices increases classification accuracy.

During human-in-the-loop evaluation, these rationales provide insight into the model’s thought process, helping evaluators refine prompts and improve model behaviour.

---

### Dynamic and Flexible Decision Making

A prompt is a set of instructions. If written well, the model can follow them dynamically. Like a rule-based system, each rule can generalise to a wide variety of utterances. No need for an exhaustive list of examples—just describe the rule clearly.

The model remains robust even when:
- Word order is slightly changed
- New or niche technical terms are introduced
- Grammar is non-standard but still legal

This is especially valuable for languages like Japanese, where word order is flexible and meaning is heavily context-dependent. The model infers **roles** of unknown words based on the utterance’s structure and semantics.

---

### Implicit or Indirect Commands

The model can handle implicit utterances and indirect commands like:

> "I'm cold"

A human would infer that the speaker is requesting warmer conditions. I use chain-of-thought prompting to model this reasoning and ask the LLM to mimic it.

Instead of asking for clarification, the model extrapolates the intended direct command—e.g. increasing the temperature or turning on the heater.

---

### Overfitting

Overfitting, in the classical ML sense, is not a concern with prompt engineering. A pretrained LLM is a frozen model—its parameters don't change based on your inputs. There is no training unless you explicitly fine-tune.

You can reuse the same utterances and prompts during development without affecting the model’s long-term behaviour.

However, **prompt overspecialization** is a real concern:
- If you test only on a narrow set of examples
- And never generalise to new ones

The model may appear to perform well, but fail on unseen data. It’s not overfitting in the traditional sense, but it's still a robustness issue.

---

### Handling Ambiguity

Natural language is ambiguous. For example:

- **(A)** “Turn up the a/c” could mean increase the fan power *or* adjust the vent direction.
- **(B)** “Skip back two songs” doesn’t translate smoothly into Japanese and may confuse users.
- **(C)** “Avoid traffic” vs. “Show alternative roads to avoid traffic” differ in intent—rerouting vs. visual information.

I address these ambiguous cases with explanations and contrasting example pairs in the prompt to teach the model how to handle them.

---

### Scalability

In production, an in-car assistant may need to manage hundreds of carrier phrases per domain, resulting in long lists of intents and slots.

To address this:
- A RAG (Retrieval-Augmented Generation) system is required for dynamic, location-aware questions (e.g., "What’s the speed limit here?")
- Vector storage is needed for questions about the car manual

To scale intent-slot definitions:
- I store domain-wise JSON schemas in a Python object
- Load them as strings into prompts dynamically
- Since they’re memory-resident, lookup latency is negligible (sub-millisecond)

---

## Future Work

### Brand Customization

Custom brand-specific intent-slot combinations can be supported by tagging them appropriately in the domain-specific prompts. The NLU module can switch between brands by accepting a brand identifier along with the driver’s utterance.

To simplify maintenance and feature rollout, I recommend keeping a **single unified prompt** for all brands. Variations should be handled with modular schema differences—not by duplicating prompts.

### Scalability for Industry-Level Voice Assistants
In a production setting, an in-car voice assistant must handle hundreds of carrier phrases per domain, resulting in a substantial list of intents and slots. To manage this complexity without compromising responsiveness, the system must be both modular and scalable.
For real-time, dynamic utterances - such as “What does that road sign mean?” or “What’s the speed limit here?” - a Retrieval-Augmented Generation (RAG) system is essential. These queries require live context, possibly involving sensor input, map data, or legal driving rules. Similarly, when drivers ask questions related to the vehicle manual, such as “What does this warning light mean?”, a vector-based retrieval system provides the necessary semantic search capability.
To scale the intent-slot classification system efficiently, each domain's schema can be stored in memory as a JSON object and dynamically inserted into the prompt at runtime. This avoids bloating a single static prompt with an exhaustive list while ensuring that only relevant information is loaded when needed. Because the schemas are already preloaded in memory, lookup latency remains negligible—typically in the sub-millisecond range—making the system responsive and performant at scale.

## Results
### Domain: audio media
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_1.jpg?raw=true)

### Domain: climate control
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_2.jpg?raw=true)

### Domain: communication
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_3.jpg?raw=true)

### Domain: device control
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_4.jpg?raw=true)

### Domain: implicit-to-direct
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_5.jpg?raw=true)

### Domain: implicit-to-direct
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_6.jpg?raw=true)

### Domain: navigation
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_7.jpg?raw=true)

### Domain: multi-intent
![Alt text for screen reader](https://github.com/renabracha/nlu_design_for_in-car_voice_assistant/blob/main/screenshot_8.jpg?raw=true)