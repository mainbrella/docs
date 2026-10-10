## MainBrella's new prepaid billing: tax implications

I reviewed the October 9–10 billing changes in `mainbrella/backend` and `mainbrella/web`, including the new prepaid-wallet implementation.

The main issue is that money arriving in your Stripe account is not necessarily the same as revenue you've earned—but it may still be taxable income before the customer spends it.

The latest implementation has several important characteristics:

- Customers buy $5–$1,000 of prepaid compute credit through Stripe.
- Unused balances carry forward rather than resetting monthly.
- Compute usage draws down the balance at published rates.
- Promotional discounts can give customers more compute credit than they actually paid for.
- Refunds and disputes revoke associated credits.
- Automatic top-ups can add additional prepaid funds.

Sources: [Prepaid implementation, October 10](https://github.com/mainbrella/backend/commit/05ee4116c4745b24dcaf36d4b0046832a1b8fe38) and [balance-history changes](https://github.com/mainbrella/backend/commit/c01684fff33c7e51d355e957404df4a48f041b70).

There are three distinct issues to manage: financial accounting, income tax, and sales tax. They don't necessarily use the same recognition date.

## 1. When do prepaid balances become taxable income?

Consider a customer who buys $100 of MainBrella compute on December 20, 2026, but only consumes $30 before December 31.

| Treatment                                         | 2026                   | 2027                                 |
| ------------------------------------------------- | ---------------------- | ------------------------------------ |
| Financial accounting: revenue earned              | $30                    | $70 when consumed                    |
| Financial accounting: unused balance at Dec. 31   | $70 liability          | Reduced as used                      |
| Federal tax, cash-basis method                    | Generally $100 taxable | $0 additional on that prepaid amount |
| Federal tax, qualifying accrual deferral election | $30 taxable            | Remaining $70 taxable                |

The accrual-deferral example assumes the $30 was earned in 2026 and the business qualifies for and properly uses the election under Internal Revenue Code §451(c).

That election generally only allows deferral through the following tax year, even if the customer doesn't use the remaining balance for several more years.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.law.cornell.edu\&sz=32)

LII / Legal Information Institute

+1



This means that if MainBrella eventually holds $500,000 in unused, nonrefundable customer credits, you cannot simply assume that $500,000 is untaxed money belonging to customers.

### What if the credits are refundable?

There is an important legal distinction between prepaid revenue and a genuine refundable customer deposit.

The Supreme Court's Commissioner v. Indianapolis Power & Light Co. decision recognized that genuine deposits that the business is obligated to repay, with the customer substantially controlling their disposition, can be excluded from income when received.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.law.cornell.edu\&sz=32)

LII / Legal Information Institute



That could make a refundable-deposit model worth exploring. However, merely calling prepaid compute credits "deposits" would not make them tax-free. The refund rights, contractual obligations, actual business practices, and whether the customer has committed to buying services all matter.

## 2. California has a significant change coming in 2027

&#x20;January 1, 2027

California's recently enacted SB 122 expands sales and use taxes to digital products, including many forms of remotely accessed software and SaaS.

Historically, many California SaaS transactions were outside the state's sales-tax base. That changes on January 1, 2027.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://cdtfa.ca.gov\&sz=32)

cdtfa.ca.gov

+1



MainBrella sells cloud compute with browser and API access. The exact treatment of its infrastructure/compute component needs to be determined, rather than automatically assuming all charges are taxable SaaS. However, this is now an immediate issue to resolve with a California sales-tax specialist or a written CDTFA determination.

Other states already tax various forms of SaaS, software access, and cloud services, subject to state-specific rules and nexus thresholds.

### I found a specific issue in your Checkout code

In `worker/lib/prepaid-billing.ts`, your latest Stripe Checkout flow:

- Does not enable `automatic_tax`.
- Does not explicitly collect or require a billing address for tax determination.
- Validates that `session.total_details.amount_tax === 0`.
- Charges automatic recharges directly through PaymentIntents without a tax-calculation step.

[View the current code](https://github.com/mainbrella/backend/blob/main/worker/lib/prepaid-billing.ts).

Consequently, simply turning on Stripe Tax will not fix the current implementation. The Checkout validation would reject a purchase with nonzero tax, and automatic recharge requires its own tax handling.

You need to determine when tax applies to funding versus actual compute consumption, particularly where credit can carry forward or be used for multiple future services, and ensure you don't tax both events.

### Important clarification: MainBrella may qualify for California's infrastructure exclusion

The CDTFA's official definitions contain an especially relevant exception: digital infrastructure, including qualifying Infrastructure as a Service (IaaS) and Platform as a Service (PaaS), is expressly excluded from the new digital-product definition.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://cdtfa.ca.gov\&sz=32)

cdtfa.ca.gov



That describes MainBrella's core offering rather closely: providing machines on which customers run their own software.

So I would seek confirmation that MainBrella's prepaid compute qualifies as excluded digital infrastructure before building California sales-tax charges into the product. Future add-ons that independently sell taxable software access may need different treatment.

This is a much better position than ordinary SaaS, and it's worth obtaining written confirmation from CDTFA.

## 3. Your current code has a revenue-accounting gap

The biggest issue I found is how promotional credit is recorded.

Suppose somebody uses a 50%-off coupon:

Compute credit

# $20

Customer pays

# $10

Initial liability

# $10

The customer has $20 in spending power, but MainBrella received only $10. If they consume $10 of that spending power, generally $5 of revenue is earned, assuming proportional allocation.

Currently, `worker/lib/prepaid-billing.ts` deliberately replaces the actual payment amount with the selected face value when adding wallet funding.

That works for compute authorization, but not for accounting. It also allows a 100%-off coupon to create paid-looking wallet credit when no money changed hands.

The wallet history records the face value and consumption, not enough information to independently calculate cash receipts, promotional discounts, earned revenue, and remaining deferred revenue.

Under ASC 606, prepaid consideration generally creates a contract liability until the service is delivered.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://dart.deloitte.com\&sz=32)

DART – Deloitte Accounting Research Tool



I would maintain two separate balances: customer-facing compute credit and the remaining dollar value of the actual consideration received.

There's another records issue: `containers/prepaid-wallet.js` retains only the latest 256 completed resource allocations. Lifetime totals survive, but older allocation details are compacted. That is fine for a fast customer dashboard, not sufficient as the sole transaction history supporting tax and financial audits.

## 4. MainBrella is currently described as a sole proprietorship

I checked the current [MainBrella Terms](https://github.com/mainbrella/web/blob/main/terms/index.html). They identify the business as a California sole proprietorship.

Assuming that reflects the current legal structure, net taxable business income flows onto the owner's personal tax return, generally through Schedule C. Self-employment taxes and estimated tax payments may also apply.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.irs.gov\&sz=32)

Internal Revenue Service



This makes prepaid billing particularly important from a cash-flow perspective.

Imagine receiving $50,000 in customer prepayments in December, with most compute usage occurring the following year. Under cash-basis taxation, the payments may increase this year's taxable receipts while the cloud expenses associated with serving those customers are incurred next year.

That doesn't mean paying tax on the entire $50,000 regardless of expenses; tax generally depends on net taxable profit. But the mismatch can be significant.

## 5. What I'd change before scaling billing

First: add an accounting-grade transaction ledger.

Store actual amounts paid, promotional credit, taxes collected, refunds, chargebacks, payment processing fees, funding dates, compute consumption, and customer tax location. Make it append-only and exportable. Keep the current Durable Object wallet for realtime authorization.

Second: implement a monthly accounting close.

Produce three independently reconciled totals: outstanding customer compute credits, outstanding deferred-revenue liability, and taxable advance payments by receipt year. These figures should not be assumed equal.

Third: decide the refund and tax treatment formally.

Your terms currently direct customers to support for refunds, but don't clearly provide an on-demand refund right for all unused prepaid credit. Have a CPA determine whether these should remain ordinary advance payments or whether a genuine refundable-deposit design makes business sense. Also get confirmation of California's IaaS exclusion and evaluate other states as revenue grows.

Fourth: make checkout tax-capable without taxing everything.

Collect appropriate customer location information, classify products and jurisdictions, and update the code to support legally required tax amounts on Checkout and automatic recharge. Tax collected should not become spendable compute credit.

## 6. One other legal consideration

An unlimited, reloadable balance can raise stored-value regulatory questions even when usable only with MainBrella. FinCEN's prepaid-access guidance has a closed-loop exclusion involving a $2,000 threshold, but its application depends on the arrangement.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.fincen.gov\&sz=32)

FinCEN.gov



Similarly, unused balances may raise state unclaimed-property questions; they cannot automatically be written off merely because a customer stops logging in.

## Bottom line

I would keep prepaid billing. It is a sensible model for MainBrella and far simpler for customers than recurring subscriptions plus usage invoices.

But I would not rely on the current wallet balance as an accounting number. Your system now needs to distinguish:

Compute credit ≠ deferred revenue ≠ taxable income ≠ cash in Stripe.

The immediate technical priority is preserving actual cash consideration separately from promotional credit. The immediate tax priority, given the sole-proprietorship terms, is confirming the business's accounting method and whether genuine refundable deposits or the §451(c) accrual deferral are appropriate.

That becomes especially important if customers start maintaining substantial balances that they don't spend for months or years.
