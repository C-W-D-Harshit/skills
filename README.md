# Skills

Agent skills I use on real projects. They work with Claude Code, Codex, and any agent that supports skills.

## Skills

- **[billing-audit](./skills/billing-audit/SKILL.md)**: Checks your payments and subscriptions for the edge cases agents usually miss, and reports what's wrong, worst first, with file references.

## Install

Pick one. Installing both gives you every skill twice.

**Any agent, with [skills.sh](https://skills.sh).** Copies the skill files into your project so you can edit them.

```bash
npx skills@latest add C-W-D-Harshit/skills
```

**Claude Code, as a plugin.**

```bash
claude plugin marketplace add C-W-D-Harshit/skills
claude plugin install harshit-skills@harshit
```

## billing-audit

AI agents write subscription code that compiles and looks right, then breaks on the first real renewal. A product gets set up as one-time instead of recurring. A subscription period matches the billing interval, so every plan expires after one cycle. A failed-payment webhook lands after the successful retry and revokes access the customer already paid for.

Run it like this:

```
Use billing-audit to check our billing and subscription edge cases. Tell me everything we're doing wrong.
```

The agent first maps your money flow from your docs: what's sold, who pays whom, and which billing decisions are on purpose. Then it finds your billing code, reads your provider's current docs, and traces every case through the real code path. It reports:

- **The money flow**, in three lines, so you can catch a wrong assumption early.
- **Problems, worst first.** Wrong charges, then wrong access, then lost revenue. Each one comes with a concrete scenario, the file and line, a one-line fix, and how sure the agent is.
- **Can't tell from code.** Dashboard settings and business decisions it needs you to answer.
- **Handled well.** What it checked and found fine.
- **Doesn't apply.** Cases it skipped and why.

It covers provider setup, webhooks and API calls, access, failed renewals, cancels, refunds, plan changes, duplicate purchases, what the customer sees, and monitoring. It works with any provider, including Stripe, Dodo, Paddle, Lemon Squeezy, Polar, and Razorpay. It only reads your code and doesn't change anything unless you ask.

## License

[MIT](./LICENSE)
