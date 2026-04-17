# CI Triage Daemon with Experience Replay 🧠⚙️

Building an LLM-based agent that works perfectly once is a parlor trick. Building one that doesn't repeat the exact same mistake later requires real infrastructure.

In modern software development, a flaky test or a cryptic build failure can stall an entire engineering team. When an AI agent swoops in to fix a bug, it's impressive. But when that identical bug reappears three weeks later and the AI wastes time exploring the exact same dead ends it tried last time? That is a system failure.

This repository contains a Continuous Integration (CI) triage daemon built natively in Julia. Instead of relying on stateless LLM API calls that hallucinate fixes and repeat past mistakes, this daemon uses Hindsight Experience Replay to give the debugging agent long-term, structured memory.

---

## ⚠️ The Problem: Stateless Agents Don't Learn

Without state, an AI agent operates in a permanent Groundhog Day. Initially, standard LLM wrappers attempt to debug CI failures by guessing blindly based solely on their generalized training data.

Consider a scenario where a pipeline throws a complex `OutOfMemoryError`.

* **The Stateless Guess:** The agent might instinctively suggest increasing garbage collection frequency.
* **The Failure:** This fails to resolve the issue.
* **The Pivot:** After several retries, the agent eventually realizes it's a dangling TCP socket issue and patches it.

The problem? The next time an `OutOfMemoryError` occurs, the stateless agent will again start by suggesting a garbage collection tweak. If it fails, it forgets the failure instantly. If it succeeds, it cannot reuse that success for future runs.

Developers often try to solve this by dumping thousands of lines of historical logs into the prompt, leading to massive token costs, high latency, and degraded reasoning quality.

---

## 💡 The Solution: Hindsight Memory Integration

Inspired by Reinforcement Learning, we solve this by implementing **Experience Replay**.

When a build fails, the daemon wakes up, tails the logs, and extracts a precise error signature. However, before it ever queries an LLM to generate a fix, it queries the Hindsight API to retrieve a trajectory of past experiences tied to similar signatures.

The agent recalls a structured timeline:

* **The Context:** What the exact error signature was
* **The Trajectory:** What sequence of actions were previously taken
* **The Outcome:** Whether those actions successfully patched the issue or failed spectacularly

By injecting this structured memory into the agent's system prompt as a set of strict constraints, fix times drop drastically. The agent actively bypasses hallucinated "fixes" because it possesses the historical context that those approaches have already proven unsuccessful in this specific codebase.

---

## 🛠️ Architecture

* **Language:** Julia (Base)
  Chosen for its high-performance string manipulation and speed when tailing massive CI log streams.

* **Agent Memory:** Hindsight by Vectorize
  Used for vector-based semantic retrieval of past event trajectories.

* **Data Structure:** C++-style `ExperienceRecord` structs
  These structs serialize cleanly into JSON payloads to ensure the memory retains its strict schema.

---

## 🚀 Quickstart (Running the Daemon)

To run the demonstration daemon locally and watch the agent recall past trajectories in real-time:

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/ci-hindsight-daemon.git
cd ci-hindsight-daemon
```

### 2. Run the Julia daemon

```bash
julia ci_hindsight_daemon.jl
```

### ✅ Expected Output

You will see the daemon:

* Intercept a simulated pipeline crash
* Query the Hindsight API
* Retrieve a past failed attempt
* Immediately pivot to the correct socket-level patch

This demonstrates real-time learning and memory reuse.

---

## 📝 Lessons Learned

### 1. Relevance Beats Context Size

The "Needle in a Haystack" problem is real. Stuffing massive prompts with raw historical logs doesn't help the agent—it confuses it. Precise, highly relevant memories retrieved via vector search provide better guardrails than massive context windows.

### 2. Failures Matter as Much as Successes

Storing a failed debugging attempt is just as critical as storing a success. Supplying the agent with **negative constraints** prevents it from falling down the same rabbit hole twice and drastically reduces API latency.

### 3. Structure the Memory

If you feed an LLM unstructured text of past events, it struggles to differentiate between what was suggested and what actually worked. Forcing the LLM to output its actions into strict, defined fields (`Tool`, `Input`, `Output`, `Success`) reduces hallucinations to near zero.

### 4. Memory Decay is the Next Frontier

Codebases evolve rapidly. A successful fix from six months ago might be an anti-pattern today. Implementing a **Time-To-Live (TTL)** on stored experiences ensures the agent doesn't forcefully apply outdated architecture patterns to modern code.
