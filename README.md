# llama hosting: how much VRAM 8B to 70B models actually need, real costs, and when Vultr's hourly GPUs beat fixed monthly plans

Hosting a Llama model on a rented server comes down to two numbers: how much VRAM your model needs, and how many hours a day you'll actually keep the GPU busy. Get those right and the plan choice makes itself. Llama 3.1 8B fits in about 5 GB of VRAM at the 4-bit quantization Ollama applies by default. Llama 3.3 70B needs around 43 GB. Your laptop, which also has a browser and a Slack app fighting for memory, is the wrong home for either.

The other big decision is billing. You'll run into two models: hourly GPU clouds such as Vultr, where you pay per hour and destroy the instance when you're done, and fixed-monthly hosts that pre-install Ollama bundles and hand you a ready endpoint. The cheaper option flips depending on how you use the card, and the crossover point is concrete enough to calculate — so this guide does the math.

**Quick picks**

> - **Experimenting, model choice still up in the air** → hourly GPU. Vultr's A100 80GB runs $2.37/hour, and the meter stops when you destroy the instance. [👉 see Vultr's Cloud GPU plans and hourly rates](https://www.vultr.com/products/cloud-gpu/?ref=7268283)
> - **One known model, running 24/7 for months** → fixed monthly wins. Database Mart's A100 80GB dedicated server is $1,559/month; their RTX A6000 (48 GB) is $409/month.
> - **A personal 7–8B chatbot with zero server admin** → AIHostBox's managed llama.cpp plans start at $13.99/month and handle a 7–8B model at Q4 on CPU.

## What llama hosting actually involves

Hosting means running a server that keeps model weights in memory and answers API requests around the clock. Three pieces: hardware with enough VRAM, an inference server, and an endpoint your apps can call. Ollama handles the messy middle — model download, quantization handling, GPU memory management — behind one command and an HTTP API on port 11434. It ships 4-bit Q4 weights by default, which is what makes mid-size models fit on single GPUs.

Sizing rule of thumb: roughly 0.6 GB of VRAM per billion parameters for the weights, plus headroom for the KV cache, which grows with context length. That cache detail bites people. A model that fits with a 2K context can overflow the same card at 32K, so leave slack instead of buying exactly the model's weight size.

Ollama is the easy runtime. vLLM is the high-throughput option when the model needs to serve real traffic. Both run on any of the hardware below.

## VRAM cheat sheet: which GPU fits which Llama model

The short answer: 8B-class models need an 8 GB card, 70B-class models need 48 GB at minimum, and the 405B and Llama 4 Maverick tiers are multi-GPU territory. Full breakdown:

| Model (Ollama tag) | Weights at 4-bit | Practical VRAM floor | Typical speed |
| --- | --- | --- | --- |
| llama3.2:1b | 1.3 GB | 4 GB | 28–100 tok/s |
| llama3.2:3b | 2.0 GB | 4–6 GB | 20–90 tok/s |
| llama3.1:8b | 4.9 GB | 8 GB | 21–84 tok/s |
| llama3.2-vision:11b | 7.8 GB | 16 GB | 38–71 tok/s |
| llama3 / 3.1 / 3.3:70b | 40–43 GB | 48 GB | 13–27 tok/s |
| llama3.2-vision:90b | 55 GB | 80 GB (or 2×40 GB) | 12–20 tok/s |
| llama4:16x17b (Scout) | 67 GB | 80 GB | 10–18 tok/s |
| llama3.1:405b, llama4:128x17b | 243–245 GB | 4×80 GB class | — |

The speed ranges span entry-level cards to flagships, so don't expect the top number on a budget plan. For the 70B tier, an A6000-class 48 GB card delivers roughly 13–27 tokens per second — fine for a small team's chatbot, slow for a public API under load.

One practical note: the 8B models are where most searches start, and they're forgiving. Nearly any modern GPU runs llama3.1:8b at 21–84 tok/s. The decision that actually costs money starts at 70B.

## Vultr Cloud GPU plans for llama hosting

Vultr takes a different approach from the specialized hosts: it rents raw NVIDIA GPU instances by the hour, gives you full root access, and leaves the software stack to you. There's no pre-installed Open WebUI bundle waiting at first login — you run the Ollama install script yourself, which takes about two minutes. In exchange you get 30+ data center regions to choose from (every specialized host covered in this guide runs out of the US), hourly billing with a monthly cap, and a general-purpose cloud around the GPU — block storage, VPC, Kubernetes — for when the model becomes part of an actual product.

The plans that map to Llama sizes:

| Plan | VRAM | Runs comfortably (4-bit) | Price | ≈24/7 per month | Deploy |
| --- | --- | --- | --- | --- | --- |
| Cloud GPU — H100 | 80 GB | 70B under heavy load, 90B vision | $3.99/hr | ~$2,913 | [ Deploy H100 (80GB)](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU — A100 | 80 GB | 70B, 90B vision, Llama 4 Scout | $2.37/hr | ~$1,730 | [ Deploy A100 (80GB)](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU — L40S | 48 GB | 8B–34B with big context; 70B fits but tight | $1.77/hr | ~$1,292 | [ Deploy L40S (48GB)](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU — A40 | 48 GB | 8B with long context; 70B fits but tight | $1.18/hr | ~$861 | [ Deploy A40 (48GB)](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |

*GPU instances are billed per hour with a cap at the equivalent monthly rate. Vultr refreshes the lineup as NVIDIA releases new cards, so the live page is the source of truth for current options and rates.*

Which plan for what: the A100 80GB is the 70B workhorse — 43 GB of Q4 weights plus real KV-cache room, and it can even hold Llama 4 Scout at 67 GB. The two 48 GB cards handle everything from 1B to 34B comfortably and squeeze in a Q4 70B with little headroom; the L40S is the newer, faster card of the pair, while the A40 is the budget route. The H100 earns its price when you're serving many concurrent users, not when one person chats with a model.

Resist the urge to buy 80 GB for an 8B model. Any 48 GB plan runs llama3.1:8b all day with room for long contexts, at a third of the hourly rate. [👉 Check current Vultr GPU pricing by plan](https://www.vultr.com/pricing/?ref=7268283) before you settle — smaller options come and go as the lineup refreshes.

## Installing Ollama on a Vultr GPU instance

Four steps from empty account to working endpoint:

1. Deploy a Cloud GPU instance from Vultr's console — pick the size from the table above and an Ubuntu image. New accounts typically get free trial credit, so you can test before paying anything.
2. Confirm the NVIDIA driver is loaded:

bash
nvidia-smi


3. Install Ollama and pull your model:

bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.1:8b
ollama run llama3.1:8b


That's it for a private setup — you're chatting over SSH. To let other apps or teammates call it, set `OLLAMA_HOST=0.0.0.0` and open port 11434 in the firewall.

One warning before you expose anything publicly: Ollama ships with no authentication. A bare public URL is an open API key for anyone who finds it. Put a reverse proxy with TLS and an API key in front, or keep the endpoint private and tunnel in over SSH. If your apps already use the OpenAI SDK, point the base URL at the server's `/v1` path — Ollama speaks OpenAI-compatible chat completions, so most existing code works with a one-line change.

For production traffic, swap Ollama for vLLM on the same instance. Same GPU, same model, much better batching behavior under concurrent load.

## Hourly vs fixed monthly: where the money actually goes

The crossover for the A100 80GB — the one card both camps sell — sits at about 658 hours a month, or roughly 22 hours a day. Vultr charges $2.37/hour, which caps out near $1,730 if the instance never sleeps; Database Mart's A100 80GB is $1,559/month flat. Keep the GPU busier than 22 hours a day and the fixed plan wins. Let it idle more and hourly wins.

Real usage is rarely 24/7. Forty hours of experiments a month on Vultr's A100 costs about $95 — the other 690 hours of the month cost you nothing, because the instance doesn't exist. On a fixed plan you'd pay the full $1,559 whether you touch the card or not.

The same logic scales down. A L40S run twenty hours a week is about 87 hours a month, or $154 — cheaper than any fixed 48 GB plan, which start at $409. And at the very bottom of the market, AIHostBox's $13.99 managed CPU plan serves a 7–8B Q4 model at 5–15 tokens per second. Fine for a personal chat front-end; hopeless for anything concurrent.

So the honest summary of the money question: hourly pricing is an option on usage patterns, and fixed monthly pricing is a bet that your usage is constant. [👉 Open Vultr's Cloud GPU page and run the math for your own hours](https://www.vultr.com/products/cloud-gpu/?ref=7268283).

## Where Vultr is the wrong choice

Three situations where a different kind of host makes more sense.

You want zero server administration. Database Mart and Cloud Clusters both ship pre-installed Open WebUI + Ollama bundles — their Llama 3.1 70B setup on an A100 80GB runs $1,559/month and works out of the box. On Vultr you're the sysadmin: updates, firewall, and monitoring are yours.

The model runs 24/7 and cost is the top priority. A dedicated A100 80GB at $1,559/month undercuts Vultr's always-on equivalent by about $170 a month, and the 48 GB tier gap is wider — Database Mart's RTX A6000 at $409/month versus roughly $861 for a 24/7 A40 on Vultr. If the card never idles, take the fixed price.

The workload is tiny and fixed. AIHostBox's managed llama.cpp plans ($13.99 for a 7–8B model, $24.99 for 13B, both at Q4) handle patching, TLS, nightly backups, and API keys for you. Vultr is overkill — and more expensive — for a one-model personal assistant.

None of these are knockouts against Vultr; they're just cases where its elasticity buys you something you won't use.

## llama hosting FAQ

**How much VRAM for Llama 3.3 70B?**
About 43 GB at Ollama's default 4-bit quantization. That means a 48 GB card as the practical minimum and an 80 GB card for comfortable context lengths. Higher-precision formats roughly double the requirement.

**Can I host Llama without a GPU?**
Up to about 13B, yes — a 7–8B Q4 model runs at 5–15 tokens per second on a decent CPU plan. The 70B tier is a non-starter on CPU; even the specialized CPU hosts top out at 13B for a reason.

**Is Ollama's API OpenAI-compatible?**
Yes. It serves OpenAI-compatible endpoints under `/v1`, so OpenAI SDK code works by changing the base URL. Its native API also exposes `/api/chat`, `/api/generate`, and `/api/pull`.

**Where should the server be located?**
Closest region to your users for latency, or an EU region if data residency matters. Worth noting: the specialized hosts compared above run their plans out of the US only, while Vultr offers 30+ regions — sometimes the deciding factor on its own.

## Quick decision guide

Match the plan to the pattern, not to the biggest number on the spec sheet:

- Model still undecided, usage spiky, or you're benchmarking quantization options → hourly GPU on Vultr, destroy the instance when you close the laptop.
- Model chosen, running 24/7, cost-first → fixed monthly. A 48 GB card at $409/month covers a Q4 70B; the 80 GB tier at $1,559/month adds the vision and Llama 4 Scout sizes.
- Personal assistant, 7–8B, you never want to see a firewall config → managed CPU hosting at $13.99/month.

Two checks before you order anywhere: confirm the card's VRAM leaves room for your longest context, and never put an unauthenticated Ollama port on the public internet.

If the spiky-usage case is yours, [👉 deploy a Vultr Cloud GPU instance and have a running Llama endpoint in about ten minutes](https://www.vultr.com/products/cloud-gpu/?ref=7268283) — an hour of A100 time costs less than a coffee, and you'll have real token speeds instead of guesswork.
