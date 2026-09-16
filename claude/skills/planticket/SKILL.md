---
name: planticket
description: Fetch the Linear ticket for the current branch, check requirements and Figma designs, then write an implementation plan
---

Follow these steps precisely:

## 1. Identify the Linear ticket from the current branch

Run `git branch --show-current` to get the current branch name.

Extract the ticket identifier from the branch name. Branches follow the pattern `feature/XXX-NNN-description` or similar — the ticket ID is `XXX-NNN` (case-insensitive).

If no ticket ID can be extracted from the branch name, tell the user and stop.

## 2. Fetch the Linear ticket

Use the `get_issue` Linear tool with the extracted ticket ID (e.g. `XXX-123`) to fetch the full ticket details including title, description, labels, priority, and status.

Also use `list_comments` to fetch any comments on the ticket that may contain additional context or requirements.

Read the ticket description and comments carefully. Note:
- The acceptance criteria / requirements
- Any technical notes or constraints
- Any linked issues or dependencies
- Any Figma links (URLs containing `figma.com`)

## 3. Check Figma designs (if available)

If any Figma links were found in the ticket description or comments:

For each Figma link, extract the `fileKey` and `nodeId` from the URL:
- URL format: `https://figma.com/design/:fileKey/:fileName?node-id=:nodeId`
- The `nodeId` in the URL uses `-` as separator (e.g. `1-2`) but should be passed as `1:2`

Use the `get_design_context` Figma tool to fetch the design context, including a screenshot and reference code. This gives you the visual design and component structure to inform the implementation plan.

If there are multiple Figma links, fetch them in parallel.

Summarize the key design details: layout, components used, colors, typography, spacing, and any interactive states visible in the designs.

## 4. Explore the codebase for relevant context

Based on the ticket requirements, use Glob and Grep to find the most relevant existing files:
- Components that will need to be modified or extended
- API endpoints or services involved
- Types and interfaces that are relevant
- Similar patterns already implemented in the codebase

Keep this exploration focused — find enough to inform the plan but don't read every file.

## 5. Write the implementation plan

Enter plan mode using the EnterPlanMode tool, then write a detailed implementation plan that includes:

### Overview
- Ticket: `XXX-NNN` — title
- Link to the Linear ticket
- Brief summary of what needs to be done

### Requirements
- List the acceptance criteria from the ticket
- Note any Figma design requirements discovered

### Technical Approach
- Which files need to be created or modified
- What components, services, or modules are involved
- Key implementation decisions and trade-offs
- Any patterns from the existing codebase to follow

### Steps
- Numbered, actionable implementation steps
- Each step should be specific enough to execute
- Group related changes together

### Open Questions
- Anything unclear from the ticket
- Design decisions that need user input
- Edge cases worth discussing

If $ARGUMENTS are provided, incorporate them as additional context or constraints for the plan.

Present the plan to the user for review via ExitPlanMode.
