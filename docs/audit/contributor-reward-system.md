# Contributor / Researcher Reward System (April 2026)

## Principle
Reward high-signal contributions that improve the system. Do NOT reward raw usage, distress disclosures, or crisis content. Internal Researcher Mode only — never public mental-health gamification.

## Reward Actions
| Action | XP | Purpose |
|---|---|---|
| label_supportedness | 3 | Mark answer as supported/partial/unsupported |
| submit_rare_term_query | 4 | Ask rare/specialized clinical questions |
| ab_compare | 6 | Compare two candidate answers, pick safer one |
| citation_fix | 5 | Flag missing inline citation |
| promote_failure_to_eval | 10 | Turn real failure into permanent eval case |
| report_crisis_miss | 12 | Flag missed crisis detection |

## Daily Missions
- Label 3 answers → +8 XP bonus
- Submit 2 rare-term queries → +10 XP bonus
- Do 2 A/B reviews → +12 XP bonus
- Promote 1 failure to eval → +15 XP bonus

## Level Formula
`level = max(1, floor(sqrt(totalXp / 25)) + 1)`

## Neon Tables
- `kira_contributor_users` (user_id, role)
- `kira_user_profiles` (display_name, last_active_date, streak_days)
- `kira_reward_ledger` (user_id, event_key [unique dedup], action_type, points, trace_id, metadata)
- `kira_feedback_items` (user_id, trace_id, kind, payload)
- `kira_daily_missions` (code, title, trigger_action, target_count, bonus_points, max_completions_per_day)
- `kira_user_daily_progress` (user_id, mission_id, day_key, count, completed_count)

## API Routes
- `POST /api/rewards/complete` — Award points for an action (x-user-id header)
- `GET /api/rewards/summary` — Get total XP, level, streak, recent rewards, mission progress

## UI Component
`AnswerFeedbackBar` — Placed under each Kira answer. Buttons: Supported (+3), Partial (+3), Unsupported (+3), Missing citation (+5), Promote to eval (+10). Shows XP/Level/Streak after action.

## Key Files
- `src/lib/rewards.ts` — Core logic (awardPoints, getRewardSummary, assertContributor, updateStreak, applyMissionProgress)
- `src/app/api/rewards/complete/route.ts` — POST endpoint
- `src/app/api/rewards/summary/route.ts` — GET endpoint
- `src/components/AnswerFeedbackBar.tsx` — Client component
- `src/app/api/cron/generate-research-missions/route.ts` — Optional daily mission rotation

## Future Additions (not built yet)
1. Internal leaderboard (contributors only)
2. A/B answer review (side-by-side comparison)
3. Wire "Promote to eval" to actually insert into eval_cases table
4. Reward multipliers: unsupported clinical claim 2x, rare-term miss 2x, crisis miss 3x
