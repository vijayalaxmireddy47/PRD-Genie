# PRD-Genie
PRD Genie: an AI-powered assistant that turns rough ideas, notes, meeting transcripts and conversations into clear, structured Product Requirements Documents (PRDs) in minutes.
PRD Genie
An AI-powered, multi-agent assistant that turns meeting transcripts, product briefs and stakeholder notes into a standard-template PRD and prioritised user stories, without inventing anything that wasn't discussed.
Built for Neuron Forge Technologies' Product and Innovation team as a six-week capstone program.
Platform: n8n workflow, OpenAI LLM (configurable), Langfuse observability.
![PRD Genie architecture](docs/PRD_Genie_Architecture_Diagram.png)
---
Table of contents
Why PRD Genie
What it does
How it works
Run modes
Grounding and safety
Quick start
Evaluation: baseline tests T1 to T12
Observability and cost
Success criteria
Program plan
Risks and mitigations
Scope
Repository structure
Roadmap
Contributing and change control
Author
---
Why PRD Genie
Product managers (PMs) and technical program managers (TPMs) at Neuron Forge lose most of their time on the steps between a meeting ending and engineering being able to estimate the work:
Gathering scope and requirements from meetings takes a disproportionate amount of time.
Turning discussions into structured PRDs, epics and user stories is manual and iterative.
Each PM writes PRDs in a different format, which makes engineering estimation difficult.
Transcripts and notes sit in documents nobody revisits, so valuable requirements are lost.
The result is delays, inconsistent tracking and slower time to market.
Vision: meetings stop being a graveyard of lost requirements. Paste a transcript and get a template-conformant PRD, prioritised stories and a list of open questions in minutes, with every requirement traceable to a quote.
What it does
Capability	Description
Requirement extraction	Pulls out features, requirements, constraints, assumptions, stakeholders and deadlines. Stated items are kept apart from ambiguous ones, and every item carries an exact source quote.
PRD generation	Fills a fixed 10-section template (overview, goals, personas, feature list, acceptance criteria, assumptions, open questions and more) following `prd_template.md`.
Epic and story breakdown	Epics, then Features, then User Stories ("As a... I want... so that..."), each prioritised as Must Have, Should Have or Nice to Have.
Gap analysis	Surfaces clarification questions, conflicts and risks at the start of planning rather than mid-sprint. Questions are capped and ranked.
Scope estimation	Gives each story an advisory size (S / M / L / XL) and flags unclear stories instead of sizing them.
Stop path	For empty or vague input, returns clarification questions instead of a PRD.
Who it is for
Primary users: PMs and TPMs who write PRDs.
Secondary users: engineering teams who estimate from PRDs and stories, and stakeholders who answer clarification questions.
How it works
PRD Genie is a multi-agent n8n workflow with five LLM agents, two routers and a quality layer.
```
Transcript / brief / notes
        |
        v
  Webhook  POST /prd-genie
        |
        v
  Config and Input Guard  --(empty or vague)-->  Stop path: clarification questions
        |
        v
  Route: Entry
        |
        v
  1. Requirement Extractor      stated vs ambiguous, exact source quotes
        |
        v
  Route: After Extraction  --(insufficient)-->  Stop path
        |
        v
  2. PRD Generator              fills 10-section template, unknowns -> Open Questions
        |
        v
  3. Story Breakdown            Epics -> Features -> User Stories, Must / Should / Nice
        |
        +-----------------+
        v                 v
  4. Gap Analyzer   5. Scope Estimator     (run in parallel)
        +--------+--------+
                 v
               Merge
                 |
                 v
  Assemble PRD Package   PRD + stories + sizes + gaps + cost
                 |
                 v
  Grounding and Eval Checks  ---> Langfuse (traces, tokens, cost, scores)
                 |
        +--------+--------+
        v                 v
   PRD Package        Eval Report
   PRD.md + JSON      pass / fail per test, tokens and cost
```
The five agents
#	Agent	Input	Output	Main guardrail
1	Requirement Extractor	Transcript, brief or notes	Requirements JSON with quotes	Quote per item, grounding check, temperature 0
2	PRD Generator	Requirements, PRD template	`PRD.md` and JSON	Template in config, unknowns go to Open Questions
3	Story Breakdown	PRD	Prioritised story tree	Acceptance criteria come only from the PRD
4	Gap Analyzer	Requirements, PRD, stories	Questions for stakeholders	Questions are capped and ranked
5	Scope Estimator	User stories	Size per story, unclear-story flags	Sizes are advice, unclear stories are flagged and not sized
All agents use the OpenAI API (default `gpt-4.1-mini`, configurable) with JSON output at temperature 0. A full run makes 5 LLM calls. A run that stops early makes 1.
Run modes
The webhook accepts three modes.
Mode	What it does
`full`	Runs every agent end to end and returns the complete PRD package.
`extract_only`	Runs only the Requirement Extractor and returns requirements and questions for PM review.
`from_extraction`	Takes a PM-reviewed extraction and generates the PRD, stories, gaps and sizes from it.
Recommended PM review loop
Run `extract_only`.
The PM checks and edits the requirements.
Run `from_extraction`.
The PM stays accountable for whatever is published. Outputs are labelled as AI-generated drafts.
Grounding and safety
PRD Genie is designed to refuse to invent requirements when the input is thin.
Quote per requirement. Every requirement carries an exact quote from the source, and the grounding check verifies each quote is found in the input.
Numbers unchanged. Figures in the output must exactly match the input.
Temperature 0 and JSON output for every agent.
Unknowns stay open. Conflicts and missing information go to the Open Questions section, never into requirements.
Input guard and stop path. Empty or insufficient input returns clarification questions and never a PRD.
Human in the loop. The review modes let a PM check and correct requirements before anything is generated.
Quick start
> The steps below describe the expected setup. Adjust names and paths to match the files in this repository.
Prerequisites
An n8n instance (self-hosted or cloud)
An OpenAI API key (an account approved by your security and legal teams for transcript data)
A Langfuse project (public and secret keys)
Setup
Import the workflows into n8n: the main workflow and the evaluation harness (`workflows/`).
Add credentials in n8n for OpenAI and Langfuse.
Set configuration in the Config node: model name, path to the PRD template, and the empty-input rules.
Load the template and prompts from `template/prd_template.md` and `prompts/`.
Activate the workflow and note the production webhook URL.
Protect the webhook with access control before sharing it, because meeting transcripts can be sensitive.
Example request
The field names below are illustrative. Match them to the webhook node in your imported workflow.
```bash
curl -X POST "https://<your-n8n-host>/webhook/prd-genie" \
  -H "Content-Type: application/json" \
  -d '{
        "mode": "full",
        "input": "<paste meeting transcript, product brief or stakeholder notes here>"
      }'
```
Example output (illustrative)
For a transcript that says single sign-on is needed for a customer portal by end of Q1, and that a mobile app might need it too:
Stated: single sign-on for the customer portal, with its source quote. Deadline: end of Q1.
Ambiguous: mobile app support, flagged as unclear.
User story: "As a customer, I want to sign in with SSO so that I can reach the portal without a separate password." Priority: Must Have.
Open question: Is a mobile app in scope for SSO?
For a vague input such as "Let's make the app better," PRD Genie stops and returns clarification questions instead of a PRD.
Evaluation: baseline tests T1 to T12
A baseline set of 12 test inputs with expected results lives in the evaluation harness. Every prompt or workflow change must pass all 12 tests before it ships. Any new failure blocks the change.
Tests called out in the charter:
Test	Checks
T1, T7	Numbers are not altered (exact-match check)
T2, T5, T9	Vague or empty input produces no PRD (stop-path check)
T11	All 10 PRD template sections are present (structure check)
T12	Stories are in the standard format with a priority (format check)
The harness runs every test input through the workflow and produces an Eval Report with pass or fail per test, tokens and cost.
Observability and cost
Every run is traced in Langfuse, with tokens, cost and scores per trace.
Estimates from the charter (replace with Langfuse figures once available):
Item	Estimate
Cost per full run	about $0.015 (target under $0.05)
100 full runs per month	about $1.52
Full baseline suite (12 tests)	about $0.11
Running cost is small next to people time, so the case for the program rests on hours saved.
Success criteria
The program succeeds if all quality gates are met at the end of week 5 and the pilot confirms the time saving in week 6. Targets marked proposed are for the sponsor to confirm.
Metric	Target	How measured
Baseline tests passed	12 of 12	Eval harness pass or fail report
Ungrounded requirements in output	0 on the baseline set	Grounding check: every quote found in the source
Numbers altered (T1, T7)	0	Exact-match check
Vague or empty input that produces a PRD (T2, T5, T9)	0 cases	Stop-path check
PRD template sections present	10 of 10	Structure check on T11
Stories in standard format with priority	100 percent (T12)	Format check on T12
Cost per full run	Under $0.05 (estimate $0.015)	Langfuse cost per trace
PM time saved on first draft	Majority of drafting time (proposed)	Pilot PMs compare against their usual process
Pilot PM satisfaction	4 or more out of 5 (proposed)	End-of-pilot survey
Program plan
A six-week program in five phases, each ending with a sponsor gate.
Phase	Weeks	Focus	Gate
1. Discovery and design	1	Interview PMs, fix the 10-section template, agree tests T1 to T12	Sponsor approves template and tests
2. Core build	2 to 3	Input guard, routers, Requirement Extractor, PRD Generator, Story Breakdown	T1, T11 and T12 pass end to end
3. Extend and instrument	4	Gap Analyzer and Scope Estimator in parallel, review modes, Langfuse, grounding checks	All five agents traced, parallel merge works
4. Evaluate	5	Run T1 to T12, tune prompts, rerun the full set after each change, review cost	12 of 12 pass with zero ungrounded content
5. Pilot and handoff	6	Two or three PMs on real meetings in review mode, feedback, documentation, demo	Sponsor go or no-go for team rollout
Rollout
Internal pilot (week 6). Two or three PMs run PRD Genie on real meetings in review mode.
Team-wide release. Open to all PMs and TPMs after the sponsor's go decision, with a one-page how-to, the template and a feedback channel.
Expansion. Integrations and further features, each approved separately by the sponsor.
Rollback: if a serious grounding failure or data issue appears, disable the webhook, return PMs to their manual process, and reopen the program at the last passed gate.
Risks and mitigations
Risk	Mitigation	Fallback
The LLM invents requirements	Quote per requirement, grounding check, temperature 0, stop path	PM review mode becomes mandatory for every run
PMs over-trust generated PRDs	Open Questions section, PM review mode, visible AI-generated label, sampled human audits	Sampled human audits
Sensitive meeting data goes to a third-party LLM	Approved API account, webhook access control, limited trace retention	Pause the pilot until security and legal approve
Vague input produces a confident PRD	Input guard and After-Extraction router stop the run and return questions (T2, T5, T9)	PM review mode allows a human override
Prompt changes cause silent regressions	Rerun T1 to T12 after every change	Any new failure blocks the change
Model or price changes	Model is a config setting, cost tracked per run in Langfuse	Switch model in config
PMs do not adopt the tool	Involve pilot PMs from week 1, keep review mode simple, collect feedback early	Act on feedback early
Template does not suit all teams	Agree the template at Gate 1 with engineering and design	Sponsor-approved template changes later
Scope
In scope
Text inputs: meeting transcripts, product briefs and stakeholder notes
Requirement extraction separating stated from ambiguous items, with people, deadlines, conflicts and missing information
PRD generation against the standard 10-section template
Epic, feature and user story breakdown with Must, Should or Nice priority
Gap Analyzer and Scope Estimator running in parallel
PM review loop using the `extract_only` and `from_extraction` modes
Stop path for empty or insufficient input
Langfuse tracing and the baseline evaluation harness
Out of scope (future work)
Audio recording and transcription
Direct sync to Jira, Notion or other tools
Version comparison of two transcripts
Fine-tuning a model and retrieval over past PRDs
Multi-language input
Final approval of a PRD. The PM remains accountable for what is published.
Assumptions: inputs are text, in English, and contain enough detail for at least some requirements. PMs review output before sharing it.
Repository structure
A suggested layout. Adjust to match what is committed.
```
prd-genie/
├── README.md
├── workflows/
│   ├── prd-genie-main.json          # main n8n workflow
│   └── prd-genie-eval-harness.json  # evaluation harness (T1 to T12)
├── template/
│   └── prd_template.md              # standard 10-section PRD template
├── prompts/                         # prompt set for the five agents
├── tests/                           # baseline inputs T1 to T12 and expected results
├── results/                         # baseline results with outputs
└── docs/
    ├── PRD_Genie_Architecture_Diagram.png
    ├── PRD_Genie_Sponsor_Deck.pptx
    └── PRD_Genie_Demo_5min.mp4
```
Roadmap
Planned as separate changes, each approved by the sponsor:
Integrations with Jira and Notion
Version comparison of two transcripts
Retrieval over past PRDs
Audio transcription
Multi-language input
Contributing and change control
Prompt and workflow tuning inside the approved scope does not need approval, but must pass the full T1 to T12 baseline suite.
Changes to scope, template sections, success targets or the LLM provider are raised with the program owner, assessed for effect on schedule, cost and risk, and approved or rejected by the sponsor.
Stakeholders
Stakeholder	Role
Product and Innovation team	Sponsor. Approves charter, scope changes and the go or no-go decision
AI PM/TPM	Program owner. Designs and builds the workflow, owns evaluation results
Pilot PMs and TPMs	Primary users. Supply real transcripts and give feedback
Engineering leads	Confirm the template and story format are estimable
Design leads	Confirm personas and flows are captured
Security and legal	Approve data handling for transcripts and the LLM account
Author
Vijayalaxmi, TPM, Neuron Forge Technologies capstone submission, November 2026.
