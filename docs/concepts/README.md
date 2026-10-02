# Concepts

Deep dives into individual mechanisms, written from the [concept template](../contributing/templates/concept-template.md). Reference pages and guides link here for the "why," not just the "how."

- [Invoice lifecycle and statuses](./invoice-lifecycle.md)
- [Payment flows: full, partial, and batch](./payments.md) — preconditions, fee splitting, rounding, and batch atomicity for `pay_invoice`, `pay_invoice_partial`, and `pay_invoices_batch`.
- [Escrow](./escrow.md)
- [Subscriptions and recurring billing](./subscriptions.md)
- Invoices, drafts, and signed invoices — *planned*.
- [Refunds and voids](./refunds-and-voids.md) — full refunds, partial refunds, voids, amendments, and buyer-initiated expiry claims.
- [Fees, volume discounts, and time-locked fee changes](./fees.md) — `calculate_fee`, merchant volume discounts, and the `propose_fee` / `execute_fee` timelock.
- Escrow and arbiter release — *planned*.
- Subscriptions and recurring billing — *planned*.
- [Merchants](./merchants.md) — registration, activation, verification, configuration, and the state-to-operation matrix.
- [Event ticketing, dynamic pricing, and resale](./event-ticketing.md)
- [Merchant analytics and transaction history](./analytics-and-history.md)
- [Auto-withdrawal and merchant settlement](./auto-withdrawal.md)
- [Payment payloads, swap routing, and the cross-chain bridge placeholder](./payment-payloads-and-routing.md)
- Fiat-pegged campaign goals and oracle pricing — *planned*.
- Creator fund vesting — *planned*.
- Dynamic hard-cap voting — *planned*.
- Stretch goals — *planned*.
- DAO governance and contract upgrades — *planned*.
- Cross-chain bridge deposits and pledges — *planned*.
- Reentrancy guard — *planned*.

← [Back to documentation home](../README.md)
