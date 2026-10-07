# Enhancement proposal: one approval path for important actions

I reviewed the current permission, connector and agent flow in the repo. I think the biggest gap is that the parts that control a real world action are already there, but they are split across a few different places.

My proposal is to add one small layer called the Action Gateway.

The idea is simple:

> When an agent wants to do something that changes something outside the app, the action goes through one path first.

For a safe read, nothing changes.

For an action that is allowed automatically, it runs as it does today.

For an action that needs the owner, the gateway saves the exact action, asks for approval, and runs that exact action only after approval.

## Why I think this matters

The project already has most of the building blocks.

src/shared/agentDefinition.ts already defines off, ask and auto levels. It also lists outward capabilities such as email.send, social.post, site.publish, calendar.write, books.write and money.move.

It also caps outward capabilities from imported packs at ask.

src/main/integrationBroker.ts already gives workers a short lived capability token. The worker can call the broker, but the real integration secret stays in the main process.

src/shared/mailboxes.ts already separates email into capabilities such as read and send, including Draft only.

src/shared/quickbooks.ts already separates QuickBooks access into Read only and Can make changes.

src/main/hooks.ts and src/main/control.ts already have a way to deny a tool call at the hook boundary.

The missing part, in my view, is one common lifecycle for the action itself.

The permission decision, connector access check, terminal approval flow and human question flow are different concepts today. That makes it harder to answer one basic question:

What exactly did an agent ask permission to do, what did the owner approve, and what exactly was executed?

## What the Action Gateway would do

Every external write would first become an action record.

    interface ActionRequest {
      id: string;
      agentId: string;
      capability: string;
      target: string;
      operation: string;
      summary: string;
      payload: unknown;
      risk: 'low' | 'medium' | 'high';
      status: 'pending' | 'approved' | 'rejected' | 'expired'
        | 'running' | 'done' | 'failed';
      idempotencyKey: string;
      createdAt: number;
      expiresAt: number;
      result?: string;
    }

The important part is that the payload is saved before the owner decides.

Example:

    Send email

    To: customer@example.com
    Subject: Revised quote
    Body: Hi Raj, here is the updated quote...
    Agent: Sales
    Reason: customer asked for the revised price

The owner is approving that exact email, not a general permission to send email.

## Proposed flow

    Agent
      |
      | propose action
      v
    Action Gateway
      |
      +--> capability check
      |
      +--> risk and approval rule
      |
      +--> auto --------------------> execute
      |
      --> ask
             |
             v
         save exact action
             |
             v
         owner approves
             |
             v
         re-check permission
             |
             v
         execute exact saved action
             |
             v
         save result
             |
             v
         tell agent what happened

## Why the exact payload matters

The approval should be tied to the actual action.

For example, an agent should not be able to ask:

Send an email to Raj for the quote.

and then change the recipient or email body after approval.

The gateway should calculate a stable hash from the action payload and keep that hash with the approval.

If the payload changes, the old approval is no longer valid.

That also helps with retries. If the app crashes after the owner approves a payment, the same action should not be executed twice when the worker retries. The idempotency key handles that case.

## What I would move behind the gateway first

### 1. Email send

The existing mail broker already knows the agent, mailbox and send permission.

The gateway would add the missing approval step for send.

Drafting would stay automatic.

### 2. QuickBooks changes

The current QuickBooks code already has Read only versus Can make changes.

The gateway would make a change such as creating or updating a record a first class action and allow the owner to approve the exact change.

### 3. Integration writes

src/main/integrationBroker.ts already provides a common path for registered REST integrations.

That makes it a good place to connect action execution after the gateway has approved the request.

### 4. Other outward capabilities

The same model can later cover social.post, site.publish, calendar.write and money.move without creating a different approval system for each one.

## What the owner would see

I would keep the UI very simple.

    Needs your approval

    Send email
    Sales agent

    To: customer@example.com
    Subject: Revised quote

    Why:
    The customer asked for a revised price.

    [Approve] [Reject]

    Expires in 10 minutes

For money movement:

    Needs your approval

    Record payment
    Finance agent

    Amount: ₹42,500
    Vendor: ABC Supplies

    Why:
    Invoice INV-1842 is still unpaid.

    [Approve] [Reject]

The goal is not to add a big admin console. It is to make the decision easy to understand.

## How it fits the existing Hive

I would not create a second task system.

The action record can live alongside the existing hive state.

    hive/
      actions/
        <action-id>.json

      tasks.json
      log.jsonl

One file per action keeps the same file based approach already used for mailboxes and messages.

The append only log.jsonl can record the important state changes:

    action proposed
    action approved
    action rejected
    action started
    action completed
    action failed

This gives the owner a simple history without putting secrets or full external responses into the UI.

## Important safety rules

I would make these rules part of the gateway, not just prompt instructions.

1. Default deny for an unknown capability.
   If the gateway cannot map the action to a known capability, it does not run it.

2. Approval is bound to the exact action.
   Changing the target, arguments or payload creates a new approval.

3. Permission is checked again before execution.
   The agent may have been changed or disabled after asking.

4. Approval expires.
   An old approval should not stay valid forever.

5. No duplicate execution.
   The same idempotency key must not execute twice.

6. Fail closed.
   If the approval state is missing or corrupted, the action is not executed.

7. Keep secrets out of the action record.
   Credentials must stay in the existing secret store.

## Suggested implementation split

I would keep the first version small.

### 1. New shared type

src/shared/actionGateway.ts

This would contain action types, action states, risk levels, payload hashing and validation.

### 2. Main process gateway

src/main/actionGateway.ts

This would own creating actions, checking capabilities, storing pending actions, approval and rejection, expiry, idempotency and execution.

### 3. Connect existing executors

Reuse:

- src/main/integrationBroker.ts
- src/main/mail.ts
- the existing QuickBooks connector path
- the existing hook checks

The gateway should decide whether an action may run. The existing broker or integration code should still decide how the request is sent.

### 4. Owner UI

Add a small approval list to the existing owner facing surface.

The owner should never have to understand tokens, connector ids or internal tool names.

### 5. Result back to the agent

After execution, the agent should receive a short result such as:

    Approved and completed.

    The email was sent to customer@example.com.

or:

    Rejected by the owner.

    Do not retry this action.

## Tests I would add

I would focus the first test set on the cases that can cause real damage.

### Approval

- an ask action creates one pending record
- approval runs the stored payload
- rejection never runs it
- an expired action never runs

### Integrity

- changing the payload after approval causes the action to be rejected
- changing the agent capability after approval causes a re-check failure

### Retry

- the same idempotency key cannot execute twice
- an app restart does not lose a pending action

### Access

- off never reaches the executor
- ask waits for the owner
- auto runs without waiting
- a connector disabled between approval and execution is refused

### Failure

- a missing secret fails closed
- an executor failure is recorded and returned to the agent
- a malformed action never reaches an external service

## Why I prefer this over a new integration

A new connector would make the product support one more service.

The Action Gateway improves the behavior of every connector and every agent that performs a consequential action.

It also uses things the repo already has instead of replacing them:

- capability levels
- the integration broker
- mailbox access rules
- QuickBooks access rules
- hook based denial
- hive task and log state

That is why I think it is a good next step for the project.

## First version scope

I would keep version one limited to:

1. email send
2. QuickBooks changes
3. registered REST writes

Once that is stable, the same gateway can be used for the other outward capabilities.

I would not add a new approval framework for every integration separately.

## Expected result

After this change, the system has one clear rule:

**Agents can think and prepare work freely, but every important external change goes through one controlled action path.**

That should make the system easier to reason about, easier to audit and safer to run without watching every agent.

This is a design proposal only. I would implement it in a follow up PR after the shape of the action record and the approval rules are agreed.
