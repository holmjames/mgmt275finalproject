MCP Manifest Generator
Turn any API into a production-ready MCP server in minutes with AI-readability scoring, safety scoping, and a test harness built in.
🔗 Live demo: jholm.co/mcp/#step-1
📄 PR-FAQ: (https://ucla-my.sharepoint.com/:w:/g/personal/omarmadkour_ucla_edu/IQCWt5gOsyi2TrZUOh5L8MLzAaYh-66YfIf_0oppBEp2DDw?e=8lZx4g)
🎥 2-min walkthrough: [Loom link]

Screenshot: <img width="1305" height="1498" alt="Screenshot 2026-05-16 at 9 46 11 PM" src="https://github.com/user-attachments/assets/2e35f0e8-49a9-4ae7-ae58-c2b47c611b90" />

What it does
Hand-authoring MCP manifests is slow, error-prone, and produces tool definitions Claude can't reliably pick against. The MCP Manifest Generator replaces that workflow with a five-step wizard that produces a signed, tested, safety-scoped manifest in under five minutes.
It's built as a single HTML file with no backend. The headline demo uses Stripe's API; any OpenAPI 3.x spec or Postman collection works.
The five-step flow

Parse — Reads an OpenAPI spec or Postman collection and normalizes every operation into an MCP tool definition.
Optimize — Rewrites tool descriptions for AI-readability, with a side-by-side diff and a selection-accuracy delta per tool.
Classify — Flags every destructive operation (deletes, writes, charges, sends) with rationale and example invocation. Overrides require explicit confirmation.
Test — Runs 50 simulated prompts per tool in a sandboxed harness. Surfaces pass rates and flags tools that need re-review.
Ship — Exports a signed manifest.json and server.json, with a summary card showing what's being deployed.

Try it locally
bashgit clone https://github.com/holmjames/mgmt275finalproject.git
cd mgmt275finalproject
open index.html
No build step, no dependencies to install. Just open the file.
Tech stack
Single HTML file. Vanilla JavaScript, no framework. js-yaml (loaded via CDN) for OpenAPI YAML parsing. Playfair Display + DM Sans for type. Deployed on Netlify.
Repo structure
/
├── index.html               # The entire product — one file, no build
├── SOURCE_OF_TRUTH.md       # Canonical spec — design, components, kill criteria, scope
├── PR-FAQ.pdf               # Press release + FAQ + appendices (Google Doc export)
├── README.md                # This file
└── docs/
    └── screenshot.png       # Demo screenshot
What's intentionally not in v2
This is a class-project prototype with deliberate scope boundaries. The optimizer scoring and test harness results are simulated — there are no live API calls to Claude or any other service, which makes the demo deterministic and credential-free. See SOURCE_OF_TRUTH.md for the full list of intentional v2 exclusions and the reasoning behind each one.
Source of Truth
The product's canonical spec lives in SOURCE_OF_TRUTH.md. If the code and the spec disagree, the spec wins. Any contributor (human or AI agent) should read it before touching code.
License
MIT — see LICENSE.

Originally built for UCLA Anderson MGMT 275 (Product Management), Spring 2026
