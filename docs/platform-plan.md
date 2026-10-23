# Open Treasury

A Cardano treasury-withdrawal platform. Andamio is the backbone. A native-script multisig wallet holds the ada. x402 charges a platform fee on the major steps.

## Why Andamio

Andamio already runs the project on-chain. This app connects to that API. It does not build a second project system.

- **Project management.** The proposal is an Andamio project. The submitter is the owner. Managers, tasks, and status come from Andamio.
- **Tasks drafted on-chain.** Milestones are Andamio tasks. The draft, the evidence, and the administrator’s decision are recorded by Andamio on Cardano. We read them back with the API.
- **Funds release follows the API.** The administrator acts in Andamio. When the API shows that administrator has accepted a task, the custody wallet pays that task’s amount. The release rule is Andamio’s. We execute the payment the API describes.

## Roles

| Piece | Role |
| --- | --- |
| Andamio | Backbone. Project, on-chain task drafts, and the administrator decision that authorizes a release. |
| Custody wallet | A prepared native-script multisig. Assigned when the proposal is presented. This is the account that receives and pays the withdrawal. |
| x402 | Cardano `exact` payment to our address before a gated action completes. Proposal submission, milestone approval, and the same class of events. This fee is separate from the withdrawal. |
| This app | Constitution form, immutable proposal document, Andamio link, wallet assignment, task view, and payout signing. |

Preprod first: `https://preprod.api.andamio.io`. Network and API key change for mainnet.

## Who can submit

Both must be true before the form can be presented:

1. The connected wallet is the Andamio project owner for this proposal (Access Token, alias, and owner on `project/create`).
2. That wallet holds at least **100,000 ADA**, the amount required to post the treasury-withdrawal governance deposit.

If either check fails, submission stays closed.

## Flow

1. **Connect.** CIP-30 wallet. Andamio login (`/auth/challenge`, sign, `/auth/verify`). Confirm owner and the 100,000 ADA balance.
2. **Write the proposal.** Each field shows the constitution clause and the exact content required. See the field map.
3. **Present.**
   - x402 fee to our address for proposal submission.
   - Publish the finished document to an immutable URL and store its hash. The on-chain action must match this document.
   - Create the Andamio project. The submitter signs and is the owner. Store `project_id`.
   - Assign a prepared custody wallet. Show its address, signers, and threshold.
4. **On-chain withdrawal.** The governance action uses the document URL and hash, the 100,000 ADA deposit, and the custody wallet as the receiving stake account. The account is delegated to Always Abstain and is not delegated to an SPO.
5. **After it passes.** Ada sits in the custody wallet. Milestones are drafted as Andamio tasks, on-chain. We read those tasks from the API.
6. **Funds release.** The administrator accepts the task in Andamio. When the API reports that acceptance, the submitter pays the x402 fee and the custody-wallet signers pay that task’s ada to the recipient.

```text
form → x402 (submit) → Andamio project + prepared wallet → governance action
     → ada arrives in the wallet
     → tasks on Andamio
     → x402 (approval) → wallet pays the recipient
```

## Constitution field map

Show this guidance beside every input. Block present until each required field has content. Amounts are in ada.

Sources: [Cardano Constitution](https://cardano.org/constitution/) Section 6, Section 7, and treasury guardrails TREASURY-01a, TREASURY-02a, and TREASURY-03a. Recipient identity, recipient location, the oversight process, and the rule above 1,000,000 ada are from the ecosystem constitution, Section 7.

| Field | Rule | What they must enter |
| --- | --- | --- |
| Title | Section 6. Each proposal includes a title. | The name of this treasury withdrawal. |
| Abstract | Section 6. Each proposal includes an abstract. | What the withdrawal does, in short. |
| Justification | Section 6. Each proposal includes a justification. | Why this withdrawal should be made. |
| Supporting materials | Section 6. Relevant supporting materials. | Links to documents the voter needs. |
| Purpose | Section 7.1. Specify the purpose of the withdrawal. | What the ada will be spent on. |
| Delivery period | Section 7.1. Specify the period for delivery of the proposed activities. | Start date, end date, and the milestone dates inside that period. |
| Costs and expenses | Section 7.1. Specify the relevant costs and expenses. TREASURY-03a: denominated in ada. | Line items and a total, in ada. |
| Refund circumstances | Section 7.1. Circumstances under which the withdrawal might be refunded to the Cardano Treasury. | The cases in which unused ada goes back to the treasury, and who sends it. |
| Recipient identity | Ecosystem Section 7. Identity of the recipient. | Legal name of the person or entity. |
| Recipient location | Ecosystem Section 7. Location of the recipient. | Jurisdiction and address. |
| Prior treasury receipt | Section 7.2. Disclose whether the recipient received ada from the Cardano Treasury in the last 24 months. | Yes or no. If yes: amount, date, and the earlier action. |
| Withdrawal amount | Section 7.3 and TREASURY-02a. A Net Change Limit must be set. The withdrawal must not exceed it. | Requested ada, the current Net Change Limit, and the remaining room. Reject the present step if the request is above the limit. |
| Audit and oversight allocation | Section 7.4. The request must allocate ada for periodic independent audits and oversight metrics. | The ada inside this request reserved for audits and metrics, and what those metrics are. |
| Administrators | Section 7.5. Designate one or more administrators to monitor use of funds and completion of deliverables. | Name of each administrator and what they check. These people are the signers of the custody wallet. |
| Custody account | Section 7.6. While an administrator holds ada before paying the recipient, it stays in a separate account the community can audit. That account is not delegated to an SPO. It is delegated to the predefined abstain option. | Filled by the app after present: custody address, Always Abstain, no SPO. |
| Oversight process | Ecosystem Section 7. Specify a process for oversight of treasury funds. | How the community can see spend, tasks, and remaining balance. |
| Large-request audit | Ecosystem Section 7. If the recipient requests more than 1,000,000 ada in any two-year period: name an independent auditor, publish a report at least yearly, and include the audit cost. | Required only above that line. Auditor, report schedule, and the audit line in the budget. |

Section 6.1: the presented document is published at a URL, the hash of that file is stored, and the file does not change after present. The on-chain governance action uses that same URL and hash. Its content must match the presented document.

## Andamio

We use the gateway. The API key stays on the server. The user signs with their own wallet.

| Step | Call |
| --- | --- |
| Login | `POST /api/v2/auth/challenge`, then `POST /api/v2/auth/verify` |
| Create project on present | `POST /api/v2/tx/instance/owner/project/create` (`project_create`) |
| Read the project | `GET /api/v2/project/user/project/{project_id}` |
| Read tasks | `POST /api/v2/project/user/tasks/list` |

The owner drafts tasks on Andamio. The administrator accepts or rejects them there. This app reads that state from the API and pays from the custody wallet only when the API says the administrator accepted the task.

Transaction path for the project create we do initiate: Andamio returns unsigned CBOR, the owner signs and submits, we `POST /api/v2/tx/register`, and we wait until `final_state` is `updated`.

## Custody wallet

Wallets are prepared ahead of time and assigned at present. One proposal, one wallet.

The wallet is a native-script multisig: a set of signer payment keys and a threshold (`atLeast`, `all`, or `any`). The assigned administrators are the signers. The stake side is delegated to Always Abstain and not to an SPO, so the account matches Section 7.6.

After the withdrawal passes, ada received from the treasury stays in this wallet until a payout. A payout is a spend the signers co-sign, for one accepted Andamio task, to the recipient address, for that task’s ada amount. The app lists the pending spend, who has signed, and the tx hash once it is submitted.

## x402

Gated actions return HTTP 402 with a Cardano `exact` payment requirement: network, amount, and `payTo` set to our address. The user’s wallet signs. The facilitator verifies and submits. The action runs only after settlement. Do not treat mempool inclusion as paid.

| Event | Who pays |
| --- | --- |
| Proposal present / submission | Submitter |
| Milestone approval | The approver |
| Further gated events of the same kind | The user who triggers the event |

Fee amounts are configuration. They are not part of the constitutional budget.

## Backend

One app. Postgres stores our records. Andamio and the chain stay the source of truth for projects, tasks, and balances.

| Record | Holds |
| --- | --- |
| User | Wallet address, Andamio alias, access-token id |
| Proposal | Constitution fields, status, document URL, document hash, `project_id`, owner alias |
| Custody wallet | Address, signers, threshold, script, abstain delegation, linked proposal |
| Payout | Task id from Andamio, amount, signatures, tx hash |
| Fee | x402 event, payer, amount, tx hash |

Status: `draft` → `presented` → `submitted` → `passed` → `in_delivery` → `complete`.

`presented` means the document, Andamio project, and wallet exist. `submitted` means the governance action is on-chain. `passed` means the withdrawal ada is in the custody wallet. `in_delivery` means Andamio tasks are being worked and paid.

Authorization:

- Present and project create: session wallet is the Andamio owner, balance at least 100,000 ADA, x402 settled.
- Read tasks: the proposal is linked to that `project_id`.
- Sign a payout: session address is a signer on that proposal’s wallet, the Andamio task is accepted, x402 for approval is settled, and the amount matches the task.

## Build order

1. Constitution form with the field map, 100,000 ADA check, and Andamio owner check.
2. Present: x402, immutable document, Andamio `project/create`, assign a prepared wallet.
3. Read Andamio tasks onto the proposal page.
4. After pass: payout from the custody wallet when a task is accepted, with x402 on approval.

## v1 is done when

A submitter who owns an Andamio project and holds 100,000 ADA can present a constitution-complete proposal, pay the x402 fee, receive a prepared custody wallet, and see Andamio tasks on the proposal. When a task is accepted, an approval fee is paid to our address and the signers can pay that task from the custody wallet.
