# Payments & data-integrity audit of a production Telegram VPN shop

Write-up of an audit and hardening pass I ran on the billing, renewal and storage layer of my
production Telegram bot (≈14k user records, card payments, auto-renewal, partner program).
**Source is closed** — this repository contains the method, findings and results only.

I ran the audit with AI coding assistants as tooling, under my direction: I scoped the work,
reviewed every change, staged it on a copy of production, ran the test suites, deployed with a
backup and one-command rollback, and verified the result in production.

## Result

| | Before | After |
|---|---|---|
| Handler fuzzing (every button/message handler × 4 client profiles) | 30 crash points in 4,278 calls | **0 crashes** in 3,888 calls (client + admin) |
| Regression tests (no network, no DB, stubbed Telegram/HTTP/MySQL) | — | **45 tests**; the new ones fail on the old code and pass on the fixed code |
| Background loops | one bad record could silence a loop for everyone | per-record isolation + alert to the owner after 3 consecutive failures |

## What triggered it

A single malformed customer record (a plain string where a key object was expected) made the
expiry-reminder loop crash on every iteration for about two days. Nothing alerted me; renewals
dipped. Root cause was not the record itself but the fact that **one bad row could take down a
loop that serves every customer**.

## Findings (all fixed)

**Payments**
- Non-idempotent payment-processing retry loop: if the panel had already extended the
  subscription and a later step failed, the retry extended it again or created a second key.
  Fix: the loop remembers what has already been applied on the panel and never repeats it.
- A flaw in my own first fix (found by adversarial review of the diff): a retry after key
  creation fell into the renewal branch and extended the new key twice.
- Purchases that went through the "ask for an e-mail" step lost the device tier, so the key got
  the default limit and auto-renewal later charged the wrong price.
- When the pending-payment state was lost (second invoice, restart), the webhook rebuilt it from
  the order id and amount only — tier and promo code were lost. Fix: both travel in the payment
  provider's metadata and are restored from there.
- Payments in a secondary flow completed after a 15-minute timeout (or after a restart) were
  never credited. Fix: longer retention, reconciliation loop covers the flow, auto-checks restart
  on boot.
- Data top-up could be credited twice (check and write separated by an await); fixed with a
  per-order lock, and a blocking subprocess call was moved off the event loop.

**Data integrity**
- Heal-on-load and heal-on-refresh for malformed records, `null` instead of `{}` defaults, and
  type guards in every loop; every heal is logged with its source so the writer can be found.
- Storage layer: parallel threads shared one temp file name (possible truncated snapshot);
  fingerprints were updated before a write succeeded; a shrunken in-memory dict could trigger a
  mass `DELETE`; writes made while the database was down were lost after it came back. Fixes:
  unique temp files, fingerprints only after success, delete-guard for large tables, an offline
  journal replayed on recovery.
- A refresh from the database could roll back a half-applied renewal; refresh is now skipped
  while a payment lock is held.

**Business-logic leaks**
- Referral program: existing paying customers (including the owner of the link on a second
  account) could be attached to a partner and generate commission on their own renewals.
  Withdrawal amount could exceed the balance. Fixed and covered by tests.
- Single-use promo codes could be applied to two open invoices; promo code was dropped when the
  customer pressed "Back". Fixed with reservation across open invoices and a TTL.
- Promo file and usage counters were read synchronously on every button press; now cached by
  mtime / short TTL.

**Operations**
- Loop supervisor now counts consecutive failures per loop and notifies the owner on the third
  (rate-limited), instead of failing silently.
- Reminder sender has a per-pass cap so a long outage cannot turn into a message burst.

## Method

1. Zone-by-zone read-through (payments, keys/trial, referrals/partner, admin/broadcasts, storage,
   web API), every claim verified against the source before touching code.
2. Adversarial review of my own diffs — found the hole described above.
3. A handler fuzzer that invokes every callback and message handler with every callback-data
   template found in the source, for four customer profiles (normal, two sets of wrong types in
   every field, missing record). Only real exceptions are kept after filtering stub noise.
4. Regression tests that reproduce each defect on the old code first.
5. Rollout: patch applied to a copy of production, both suites run there, deploy script that
   takes a backup, restores the production-only secrets, compiles, restarts and checks logs;
   rollback is one command.

## Stack

Python 3 · aiogram · FastAPI · MySQL / PostgreSQL · YooKassa · Xray / PasarGuard / Remnawave.

---

## Кратко по-русски

Аудит биллинга, продлений и хранилища моего продакшен-бота (≈14 тыс. клиентов). Поводом стала одна
битая запись клиента, из-за которой цикл напоминаний молча падал двое суток. Исправлены:
неидемпотентные повторы обработки платежа, потеря тарифа и промокода, неучтённые платежи
после таймаута, гонки в хранилище, дыры партнёрской программы. Итог: 45 регрессионных тестов,
фаззер обработчиков — 0 падений (было 30), оповещение владельцу о падающих циклах.
Исходный код закрыт.
