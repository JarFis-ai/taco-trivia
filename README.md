# Taco Trivia - case study

**[Read the full case study →](https://jarfis-ai.github.io/taco-trivia/)**

Taco Trivia is a Jeopardy-style trivia platform for ESL classrooms. The teacher projects the
board onto a smartboard and runs the whole game as sole operator - students never take out a
device, never log in, and never see a second screen.

It is built for hagwons and academies in Korea: a class split into teams, shouting answers at
each other across a room, with the teacher awarding points.

This repository is the case study and screenshots. The application source is private.

---

## The decision that shapes everything

**Host-controlled, with no multiplayer at all.**

There is no student client, no realtime channel, and no server-side game session. A live game
lives entirely in the teacher's browser tab as a single `useReducer` store - teams, scores,
played tiles and the phase machine. Games are stored as reusable templates and fetched once;
nothing about a running game is ever persisted.

That removes an entire category of work - sockets, lobbies, reconnects, session state, sync
bugs - and it is why a game cannot collapse mid-lesson because the school Wi-Fi hiccupped.

## Built with

Next.js 16 (App Router) · React 19 · TypeScript · Tailwind CSS v4 · Supabase (PostgreSQL, Auth,
Row-Level Security, Storage) · Paddle Billing · Vercel

Access control is enforced in two layers: RLS policies are the real security boundary,
middleware guards the admin routes. Entitlements are granted in exactly one place - a
signature-verified `transaction.completed` webhook - and nowhere else in the codebase.

Paddle rather than Stripe because Stripe does not support payouts to Korea, and because as
merchant of record Paddle absorbs the cross-border VAT compliance a solo developer would
otherwise carry personally.

## Where it stands

| | |
|---|---|
| Games | 10 authored - 4 free, 6 premium |
| Questions | 368, across 60 categories |
| Board sizes | 5 to 8 columns wide |
| Media | 411 images, all re-hosted - no hotlinks |
| Schema | 8 versioned migrations, 12 RLS policies |
| Codebase | ~4,500 lines of TypeScript and React |
| Payments | Full loop verified end to end in Paddle sandbox |

## Three problems worth writing down

1. **My own schema let users promote themselves.** An RLS `using` clause scopes *rows*, not
   *columns*. The "users can update their own profile" policy was row-correct and column-blind,
   so any signed-up user could patch their own row with the public anon key and set
   `role = 'admin'` or `has_all_access = true`. Fixed in three layers - column grants, a
   `with check` on the policy, and a `before update` trigger that reverts the privilege columns
   for anon and authenticated callers - then verified live against the running database.

2. **The paywalled games advertised "0 questions".** The public library derived each card's
   question count from an RLS-filtered `questions` embed, which is empty for premium games
   unless the caller already owns the pass. The security model was working perfectly and
   quietly destroying the funnel: every locked game showed zero questions on the page whose job
   was to sell it. The listing route now tallies counts with the service-role client, while the
   gameplay route still enforces the paywall unchanged.

3. **It fit on my monitor and broke on a classroom TV.** The team panels overlapped the board
   and "Return to board" sat below the fold - on the TV only. One root cause for both: the
   projected screens were sized off the viewport's *width* while the thing running out was its
   *height*. Reproduced with Playwright at five viewport sizes (the board overlapped by 49px,
   129px and 97px at the short ones, and not at all at 2560×1400). Three rules came out of it:
   cap type on both axes, make fractional grid rows `minmax(0, 1fr)` so they can actually
   shrink, and give each screen exactly one flexible block to absorb the leftover height.

## The feature I deleted

Tiles could be flagged as chaos cards - steal, shield, switch, bomb - with a splash screen and
a branch of game logic behind them. The splash screen only ever *described* the effect; the
teacher still applied the score change by hand. It was a rules engine that announced rules and
enforced none of them, so it came out.

What replaced it: the admin drops an ordinary-looking image onto any tile, and when the class
flips it the teacher decides on the spot what happens, using the score buttons that already
exist. Same surprise, none of the code. It survives publicly as a labelling flag that puts a
"Special Cards" badge on a game card - because the thing that sells was never the state
machine, it was the moment the class realises a tile is not a question.

## Related

- [Mango Mafia - case study](https://jarfis-ai.github.io/mango-mafia/) - Mafia for an ESL classroom
- [59 Seconds - case study](https://jarfis-ai.github.io/) - a projector-first ESL speaking game

---

Jacobus Barnard · Seoul, South Korea · [github.com/JarFis-ai](https://github.com/JarFis-ai)
