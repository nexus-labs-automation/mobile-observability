---
name: user-journey-tracking
description: Track multi-step user flows, funnels, and conversions. Use for onboarding, checkout, signup, or any multi-screen user journey.
---

# User Journey Tracking

Track completion, drop-off, and timing across multi-step flows.

## Core Concept

Every journey needs:
- `journey_id` - correlates all steps
- `step_started_at` / `step_completed_at`
- `is_final_step` - marks conversion

## When to Use

- Onboarding flows
- Checkout/payment funnels
- Signup/registration
- Any multi-step process

## Key Events

| Event | When |
|-------|------|
| `journey_started` | First step entered |
| `journey_step_completed` | Each step done |
| `journey_abandoned` | Exit without completion |
| `journey_completed` | Final step done |

## Funnel Metrics

- **Completion rate**: completed / started
- **Drop-off point**: step with highest exit
- **Time to value**: first meaningful action

## Mobile-Specific Concerns

- Session interruption (app backgrounded)
- State persistence across app kills
- Anonymous → authenticated identity merge

## Implementation

See `references/user-journeys.md` for:
- Journey ID correlation patterns
- Cross-session persistence
- Deep link attribution
- Error recovery funnels
