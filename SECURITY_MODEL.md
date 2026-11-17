# AIEcourse threat model

**Scope:** Current default-branch source for `joshgeron/AIEcourse` (reviewed at `e96a951`), including its Docker build and deployment instructions. The repository owner confirmed the app only runs locally or is not deployed, and asked to proceed without a separate infrastructure repository.

## System and trust boundaries

| Component | Boundary and code |
| --- | --- |
| **Chat UI** | A browser loads the Chainlit interface, establishes a chat session, submits text, and renders streamed responses. Browser input and generated output are untrusted. `.chainlit/config.toml`, `.chainlit/translations/`, `chainlit.md`. |
| **Chat Service** | Chainlit dispatches chat-start and message events to Python. The service loads credentials from its environment, formats the browser's text into a user message, writes prompt content to standard output, and streams provider output back to the requesting UI. `app.py:11–80`. |
| **Model Provider** | OpenAI sits outside the app's trust boundary. The Python SDK sends user input plus the fixed system instruction using the server's credential, then reads streamed completion chunks. No model tool calls, retrieval integration, or persisted chat database are evident in the application source. `app.py:13–19,40–80`, `requirements.txt`. |
| **Build and Hosting** | The host builds a container from the source tree and Python dependencies, runs it as an unprivileged user, exposes the application on port 7860, and supplies the API key. Build inputs, image access, host settings, and log access have their own trust boundaries. `Dockerfile:1–11`, `requirements.txt`, `README.md:90–156`, `.gitignore:138–144`. |

## Entry points and callers

- **Browser → Chat Service:** The local operator's browser is the expected client. No app-defined sign-in callback or per-user quota is visible. The public Hugging Face Space described in the deployment guide is a future deployment scenario, not a current exposure. Local network reach, port forwarding, and container port binding are unverified.
- **Chat Service → Model Provider:** Each handled chat message can trigger a provider request billed to the configured credential. The user does not supply that credential through the chat UI.
- **Maintainer → Build and Hosting:** Maintainers supply build context, choose dependency versions, and configure runtime secrets. Hosting and repository operators may have privileged access, but actual roles and log visibility are unverified.

## Assets and controls

| Asset | Exposure and observed controls |
| --- | --- |
| **OpenAI credential and spend (owner priority)** | The key is intended to live in local environment configuration or a hosting secret; `.env` is ignored by Git. The app fixes model and completion settings and caps output tokens per response, not cumulative usage. Git ignore is not a container context control. |
| **Chat contents and derived logs** | User text crosses into the service, its standard output, and OpenAI. Chat confidentiality depends on access to those systems and their retention settings; no app-level redaction is visible. |
| **Service and image integrity** | The container uses an unprivileged runtime user, and named direct Python dependencies are pinned. Transitive resolution, host policy, and base image contents are outside those controls. |
| **UI and session integrity** | The configuration disables unsafe HTML and sets a session timeout; the prompt playground and multimodal setting are enabled. These settings do not establish authorization. |

## Threat actors and triage assumptions

- **Local operator, other local clients, and log readers:** Treat untrusted browser input and generated output as untrusted even on a local instance. Consider exposure to anyone who can reach a locally published port or read local logs or images; such access is not confirmed.
- **Anonymous internet visitors and automation:** Not a current attacker through the app's web interface given the owner-confirmed local-only or undeployed status. Reassess if it is made public. Do not assume this limits any public exposure of Git, images, or credentials.
- **Compromised build or hosting inputs:** Consider untrusted dependency updates or inappropriate access to build artifacts and host settings. Test neither live services nor third-party systems.

## User-confirmed facts and open questions

- The repository owner identified the OpenAI credential and API spend as the top priorities, confirmed local-only or undeployed status. Risks that require an internet-accessible deployment can rank lower; do not treat unrelated risks as accepted or dismiss an issue merely because the app is local. Source-derived assumptions do not change severity on their own.
- Open questions: Whether local access extends beyond the operator, whether sensitive text is used, who can read logs or images, and what provider budget and credential restrictions are configured.
