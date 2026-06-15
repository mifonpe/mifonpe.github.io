---
layout: post
published: false
title: "Building production-grade agents with Strands & Bedrock AgentCore 🤖☁️"
subtitle: A hands-on walkthrough of a RAG agent with memory, tools, guardrails and a chat UI — all deployed on AWS
tags: [aws, bedrock, agentcore, strands, agents, ai, terraform]
comments: true
lang: en
lang-ref: strands-agentcore-agents
---

<p style='text-align: justify;'>Everyone is building AI agents these days, but going from a notebook demo to something you would actually let a user talk to is a different sport. You need a model, sure, but you also need <strong>memory</strong> across sessions, <strong>retrieval</strong> from your own knowledge, <strong>tools</strong> the agent can call safely, <strong>guardrails</strong> for what comes in and out, and somewhere to <strong>run all of it</strong> without reinventing infrastructure every time.</p>

<p style='text-align: justify;'>That is exactly the gap <a href="https://strandsagents.com" target="_blank" rel="noopener">Strands Agents</a> and <a href="https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore.html" target="_blank" rel="noopener">Amazon Bedrock AgentCore</a> try to close. Strands is the SDK you write the agent in, and AgentCore is the managed AWS platform that runs it, remembers things for it, exposes tools to it and filters its inputs and outputs. This is also the topic of <a href="/talks" rel="noopener">a talk I gave at SpainClouds Summit 2026</a> — and to play with the full stack end-to-end, I put together a companion repository — <a href="https://github.com/mifonpe/strands-agentcore-agents" target="_blank" rel="noopener">strands-agentcore-agents</a> — that you can clone, <code>terraform apply</code>, and chat with in under fifteen minutes.</p>

<h2>An agent is more than an LLM 🧠</h2>

<p style='text-align: justify;'>Before getting to the code, it's worth being explicit about what we actually mean by "agent". A model alone is a chatbot. An <em>agent</em> is the model plus the scaffolding that lets it act:</p>

<ul>
<li><strong>LLM</strong> → planning and reasoning</li>
<li><strong>SDK / framework</strong> → makes it adaptive and proactive</li>
<li><strong>Prompts / instructions</strong> → goals and behavior</li>
<li><strong>Memory</strong> → continuity across turns and sessions</li>
<li><strong>Tools</strong> → the ability to do things, not just say things</li>
<li><strong>Knowledge bases</strong> → grounding in your data</li>
<li><strong>Protocols (MCP, A2A)</strong> → collaboration with other systems and other agents</li>
</ul>

<p style='text-align: justify;'>Every box in that list has a counterpart on AWS, and that's where the next disclaimer comes in.</p>

<h2>Bedrock ≠ AgentCore ⚠️</h2>

<p style='text-align: justify;'>This is a confusion I keep hearing, so let's get it out of the way: <strong>Bedrock and AgentCore are not the same thing</strong>. They are complementary, but they answer different questions.</p>

<ul>
<li><strong>Amazon Bedrock</strong> is the "AI menu" — Models, Knowledge Bases, Guardrails, Flows, Prompt Routers, Evals, Bedrock Agents… It is where the model lives and where the foundational primitives sit.</li>
<li><strong>Amazon Bedrock AgentCore</strong> is the <em>agent-specific</em> ecosystem on top — Runtime, Memory, Gateway, Identity, Tools. It is what turns a model call into a long-running, stateful, tool-using agent that you can actually deploy.</li>
</ul>

<p style='text-align: justify;'>You'll use both in the same app, but knowing which job goes where will save you a lot of head-scratching.</p>

<p align="center">
<figure class="wp-block-image size-large"><img src="/assets/img/strands-agentcore/intro.png" alt="Strands Agents on Bedrock AgentCore" /><figcaption style="text-align: center;">Placeholder: hero image / architecture diagram</figcaption></figure>
</p>

<h2>What we are building 🧩</h2>

<p style='text-align: justify;'>The repo ships <strong>two agents</strong> on purpose, because they show two very different deployment philosophies that AgentCore supports:</p>

<ul>
<li><strong><code>agent/</code></strong> — a full RAG assistant with a Knowledge Base, long-term Memory, MCP-based tools served from a Gateway, Bedrock Guardrails and a Chainlit chat UI. Everything is provisioned with Terraform and deployed as a container to AgentCore Runtime.</li>
<li><strong><code>sdk-agent/</code></strong> — a tiny code-interpreter agent deployed with the <code>agentcore</code> CLI in a single command. No Terraform, no Docker, no infra files. Perfect to see how light the SDK can feel when you don't need the full pipeline.</li>
</ul>

<p style='text-align: justify;'>Both share the same brain — Claude Sonnet 4.6 via Amazon Bedrock — but the surrounding scaffolding is wildly different. That contrast is the most valuable thing the repo tries to show.</p>

<h2>The architecture in one picture 🏗️</h2>

<p style='text-align: justify;'>Before diving into the pieces, here is the bird's-eye view of how the full agent talks to its dependencies inside AgentCore:</p>

<p align="center">
<figure class="wp-block-image size-large"><img src="/assets/img/strands-agentcore/architecture.png" alt="High-level architecture" /><figcaption style="text-align: center;">Placeholder: high-level architecture diagram (Runtime + Memory + Gateway + KB)</figcaption></figure>
</p>

<p style='text-align: justify;'>The Chainlit UI calls <code>invoke_agent_runtime</code> on the AgentCore Runtime ARN. The runtime container then orchestrates calls to the Knowledge Base (RAG), to Memory (long-term context), and to the Gateway (which speaks MCP and fronts Lambda-based tools). Guardrails sit in the middle, filtering both prompts and completions.</p>

<h2>Strands: the agent in 30 lines 🧵</h2>

<p style='text-align: justify;'>The thing I like the most about Strands is that an agent really is just a model, a system prompt and a list of tools. Here is the <em>entire</em> SDK agent from the repo, code interpreter included:</p>

```python
from strands import Agent
from bedrock_agentcore import BedrockAgentCoreApp
from strands_tools.code_interpreter import AgentCoreCodeInterpreter

code_interpreter_tool = AgentCoreCodeInterpreter(region="eu-central-1")

SYSTEM_PROMPT = """You are an AI assistant that validates answers through code execution.
When asked about code, algorithms, or calculations, write Python code to verify your answers."""

app = BedrockAgentCoreApp()
agent = Agent(
    tools=[code_interpreter_tool.code_interpreter],
    system_prompt=SYSTEM_PROMPT,
)

@app.entrypoint
async def agent_invocation(payload):
    stream = agent.stream_async(payload.get("prompt", ""))
    async for event in stream:
        yield event

if __name__ == "__main__":
    app.run()
```

<p style='text-align: justify;'>That is it. No HTTP server boilerplate, no streaming plumbing — <code>BedrockAgentCoreApp</code> wraps everything. Shipping it to AWS is a one-liner: <code>agentcore deploy</code>. The framework packages, pushes and registers the agent for you.</p>

<h2>The full RAG agent 📚</h2>

<p style='text-align: justify;'>The bigger sibling under <code>agent/</code> is what you would actually deploy at work. It still looks like Strands at heart, but it is wired into four AgentCore primitives that are worth understanding individually:</p>

<h3>Knowledge Base + S3 Vectors</h3>

<p style='text-align: justify;'>The agent's <code>retrieve</code> tool searches a Bedrock Knowledge Base backed by <strong>S3 Vectors</strong>, the new vector store AWS announced earlier this year. You upload PDFs, Markdown or DOCX files to an S3 bucket, trigger an ingestion job, and the KB chunks, embeds and indexes them automatically. From the agent's perspective, RAG is just another tool call.</p>

<p align="center">
<figure class="wp-block-image size-large"><img src="/assets/img/strands-agentcore/kb.png" alt="Knowledge Base with S3 Vectors" /><figcaption style="text-align: center;">Placeholder: Knowledge Base console / RAG flow</figcaption></figure>
</p>

<h3>AgentCore Memory</h3>

<p style='text-align: justify;'>Memory is the piece most demo agents quietly skip. AgentCore Memory gives you a managed, structured store with first-class concepts for <em>session summaries</em>, <em>user preferences</em> and <em>extracted facts</em>. The repo wires it in with sensible retrieval thresholds:</p>

```python
_warmup_config = AgentCoreMemoryConfig(
    memory_id=MEMORY_ID,
    session_id="default-session",
    actor_id="default-user",
    retrieval_config={
        "/summaries/{actorId}/{sessionId}/": RetrievalConfig(top_k=5, relevance_score=0.5),
        "/preferences/{actorId}/":          RetrievalConfig(top_k=10, relevance_score=0.3),
        "/facts/{actorId}/":                RetrievalConfig(top_k=10, relevance_score=0.3),
    },
)
```

<p style='text-align: justify;'>This is the difference between an agent that says <em>"nice to meet you"</em> every time and one that actually remembers you preferred Python over Go three conversations ago.</p>

<h3>AgentCore Gateway and MCP tools</h3>

<p style='text-align: justify;'>The Gateway exposes Lambda functions as <a href="https://modelcontextprotocol.io/" target="_blank" rel="noopener">MCP</a> tools the agent can call. The repo ships two on purpose — one silly, one practical — so you can see the pattern without drowning in business logic:</p>

<ul>
<li><strong><code>get_commit_message</code></strong> — hits <a href="https://whatthecommit.com" target="_blank" rel="noopener">whatthecommit.com</a> and returns a random funny commit message. Useful for nothing, fun for everything.</li>
<li><strong><code>kv_store</code></strong> — <code>put</code> and <code>get</code> against a DynamoDB table, i.e. the simplest possible "stateful side effect" a tool can have.</li>
</ul>

<p style='text-align: justify;'>Auth is SigV4 over IAM, so the agent's execution role is what grants it access to tools — no API keys floating around.</p>

<p align="center">
<figure class="wp-block-image size-large"><img src="/assets/img/strands-agentcore/gateway.png" alt="AgentCore Gateway with Lambda tools" /><figcaption style="text-align: center;">Placeholder: Gateway + Lambda tools diagram</figcaption></figure>
</p>

<h3>Bedrock Guardrails</h3>

<p style='text-align: justify;'>Plugging a guardrail in is one extra kwarg on the model:</p>

```python
_model_kwargs.update(
    guardrail_id=GUARDRAIL_ID,
    guardrail_version="DRAFT",
    guardrail_trace="enabled",
)
model = BedrockModel(**_model_kwargs)
```

<p style='text-align: justify;'>The Terraform module configures content filtering (hate speech), PII anonymisation on the way out, topic denial for politics, and a profanity filter. Tweak it to whatever your compliance team needs.</p>

<h2>Running it 🚀</h2>

<p style='text-align: justify;'>The repo leans on a single <code>Makefile</code>, so the path from zero to a working agent is short:</p>

```bash
make tf-init && make tf-apply   # provision everything
# copy outputs into .env
make publish                    # build & push the ARM64 image to ECR
make deploy                     # update the AgentCore Runtime + endpoint
make deploy-ui                  # launch the Chainlit UI on :8000
```

<p style='text-align: justify;'>And if you just want to iterate on the agent code without deploying anything, <code>make dev</code> runs the agent and the UI together via <code>docker-compose</code>. The UI then bypasses AgentCore Runtime and calls the FastAPI service directly — much faster feedback loop.</p>

<p align="center">
<figure class="wp-block-image size-large"><img src="/assets/img/strands-agentcore/ui.png" alt="Chainlit UI chatting with the agent" /><figcaption style="text-align: center;">Placeholder: screenshot of the Chainlit UI chatting with the agent</figcaption></figure>
</p>

<h2>Pretty… but is anyone actually using this? 🛠️</h2>

<p style='text-align: justify;'>Fair question. The honest answer is that the agent shape we've just built maps almost one-to-one onto a handful of patterns I see again and again in platform teams:</p>

<ul>
<li><strong>Developer support agents</strong> — first line of triage for the inevitable "how do I deploy this?" / "why did my pipeline fail?" questions, grounded in your own runbooks via the Knowledge Base.</li>
<li><strong>Incident management agents</strong> — pulling alert context, correlating signals, drafting the first version of an incident timeline while a human takes over.</li>
<li><strong>Code review agents</strong> — running a first pass on PRs with project-specific guidelines pulled from the KB, leaving the nuanced calls to the human reviewer.</li>
</ul>

<p style='text-align: justify;'>My favourite framing for the platform-team use case is what I called <strong>J.A.R.V.I.S.</strong> in the talk — <em>Joint Agent for Resilience, Visibility & Infrastructure Services</em>. Yes, the acronym is on purpose. The idea is the same RAG-plus-tools-plus-memory shape, pointed at the kind of work platform engineers do day to day.</p>

<h2>What surprised me 💡</h2>

<p style='text-align: justify;'>A few things stood out after living inside this codebase for a while:</p>

<ul>
<li><strong>Strands is unopinionated in the right places.</strong> You can drop it into FastAPI, into an <code>agentcore</code> CLI app, or anywhere else — it does not drag the rest of the framework with it.</li>
<li><strong>AgentCore primitives compose cleanly.</strong> Memory, Gateway and Guardrails are independent. You can start with just the runtime and add the rest as you grow.</li>
<li><strong>The "boring" parts are where the value is.</strong> The model is fungible — Claude today, something else tomorrow. The IAM roles, the SigV4-signed gateway, the streaming SSE handler, the session manager… <em>that</em> is what makes the difference between a demo and something you can hand to users.</li>
<li><strong>This is just the single-agent story.</strong> Strands already supports <em>multi-agent</em> setups, <em>agents-as-tools</em>, <em>agent-to-agent</em> protocols and <em>dynamic tools</em>. The repo is intentionally a stepping stone — once you're comfortable with one agent, those are the next levers to pull.</li>
</ul>

<h2>Wrapping up 🎁</h2>

<p style='text-align: justify;'>If you want to skip the "hello world" stage and look at a Strands + AgentCore setup that already has all the production-grade pieces wired up, clone the repo and break things:</p>

<p style='text-align: justify;'>👉 <a href="https://github.com/mifonpe/strands-agentcore-agents" target="_blank" rel="noopener">github.com/mifonpe/strands-agentcore-agents</a></p>

<p style='text-align: justify;'>I will keep evolving it as AgentCore adds features, so issues and PRs are very welcome. And if you build something interesting on top of it, ping me — I would love to see it. </p>

<p style='text-align: center;'>
⎈☁️🤖 Stay curious, never stop learning! 🤖☁️⎈
</p>
