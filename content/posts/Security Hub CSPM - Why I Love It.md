---
title: "Security Hub CSPM: Why I Love It"
date: 2026-04-02
draft: false
---

I didn’t expect to like AWS Security Hub as much as I do.

It’s the kind of service you turn on because someone in a retrospective said “we should probably have this” and then nobody thinks about it for a fortnight. And then one day you’re in a meeting and someone asks whether encryption is enforced across all your buckets, and instead of going “uhh give me ten minutes and a prayer,” you just… know. That’s when it clicks.

## The bit where it actually clicks

What makes Security Hub (and CSPM in general) genuinely useful is that it turns “are we still doing the right things everywhere?” from a periodic manual audit into something that’s just continuously answered. You’re not writing one-off scripts. You’re not grepping through Terraform state. It’s evaluated, it’s consistent, and it tells you when something drifts.

The other thing — and this took me a while to appreciate — is that it’s not purely about alerts. It gives you *context*. The finding links back to the control, which links to the rationale, which is actually readable. It’s one of those rare AWS experiences where the docs surface at the right moment rather than three tabs deep in a rabbit hole.

## Which frameworks are worth enabling

Security Hub is only as good as the standards you actually turn on — and “turn everything on” is a reliable way to immediately stop trusting the tool.

**AWS Foundational Security Best Practices** is the obvious starting point, and AWS basically assumes you’re running it. It’s opinionated, maintained by people who know how the services are actually supposed to work, and catches the things that are embarrassingly easy to miss — public S3 buckets, missing encryption, IAM policies that are basically “do whatever you want.” Also KMS key deletion protection, because we’ve all either done it or been in a post-incident review about someone who did, and losing access to everything encrypted with a deleted key is not a fun afternoon.

**CIS AWS Foundations Benchmark** is where you go when you need to have conversations with external stakeholders about your security posture and want a shared language. It overlaps with AWS Foundational in places, but it’s more prescriptive and less “AWS says so” — which is actually useful when you need to defend a control to someone who doesn’t work at AWS.

**CIS v3** is the tighter, more modern version. It reflects how people actually run AWS now — multi-account, lots of automation, the usual sprawl — and it’s noticeably stricter. There’s an adjustment period where it surfaces things you’d quietly filed under “we should sort that” and now have to actually sort, but once you’ve done that work, it’s a meaningful step up.

They overlap, they sometimes disagree, and when they disagree it’s usually worth thinking about *why* rather than just picking one to suppress. That tension is useful.

## Plugging it into your actual workflow

One thing that took us a while to sort out is how Security Hub fits into day-to-day operations rather than just being a dashboard you check occasionally.

Something I picked up from [Mark Tonks](https://uk.linkedin.com/in/mark-tonks-25242213) — who is full of excellent ideas and I’ve learnt loads from — is having a Lambda generate a daily summary of all HIGH/CRITICAL findings and post it to Microsoft Teams. It means the team gets a morning digest without needing to log into the console, and it’s easy to spot when something newly bad has appeared overnight. Simple, but it makes a real difference to how much attention the findings actually get.

AWS have also just shipped native publishing of Security Hub findings into CloudWatch (https://aws.amazon.com/about-aws/whats-new/2026/03/amazon-cloudwatch-securityhub-findings/), which looks like it could simplify a lot of the EventBridge glue we’ve been writing to get findings into alarms. Haven’t had a chance to properly try it yet, but it’s on the list.

## Dealing with the real world: suppressing findings properly

Here’s the thing nobody tells you when you first set this up: real environments are messy, and if you don’t tune your findings, your engineers will stop looking at the dashboard within about two weeks.

The specific case that forced us to actually think about this: we have accounts in our org where we don’t have root access. We didn’t set them up, we don’t manage the root user, we’re not responsible for it. AWS Foundational has **IAM.6 – Hardware MFA should be enabled for the root user**, which is a completely correct and sensible control. It also shows as **CRITICAL** across every account we don’t own, which is not information we can do anything with.

If you leave that unaddressed, it poisons the dashboard. CRITICAL findings that are months old start to feel like wallpaper, and then you miss the CRITICAL finding that’s actually new and actually yours.

So we suppress it — but specifically. Not “disable IAM.6 everywhere” but “suppress this control, for these accounts, with a note explaining why.” The standard stays intact, the noise is removed, and when someone looks at the suppression list they can see it’s a deliberate decision rather than a forgotten one.

That last bit matters more than it sounds. There’s a real difference between *ignoring* a finding and *acknowledging* it. The suppression should be a record that someone thought about it, decided the risk is understood, and owns that decision. Otherwise you’re not managing risk, you’re just hiding it.

## The gripe: when AWS markets at you via findings

I’ll be honest, this one still mildly annoys me.

Occasionally a new set of controls appears and you end up with findings for things like Inspector not being enabled or Macie not being enabled. Which, fine, they’re decent services. But “you’re not using this AWS service” being surfaced as a security finding feels slightly like being told you have a cavity by the person trying to sell you a specific brand of toothpaste.

In practice: you might already be covering the same risk elsewhere. You might have cost constraints. The service might not fit your workloads. But now your dashboard — which was green — has new findings at HIGH severity, and nothing about your actual security posture has changed. The resulting conversation of “no, nothing is actually wrong, AWS just wants us to buy Macie” is tedious every single time.

Same fix as before. Review the controls, suppress what’s not relevant, document why. But there’s always a brief moment of “oh come on” before you get there.

---

Security Hub isn’t the kind of service that grabs headlines or looks cool in an architecture diagram. It’s the one that’s quietly doing its thing in the background, telling you when something has drifted, making the “are we secure?” question much less scary to answer. Once it’s properly tuned it mostly disappears — and you only notice it when it surfaces something real. That’s kind of the point.
