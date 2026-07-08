# Troubleshooting Case Study: ACL Blocking Unexpected Direction of Traffic

## Summary
While implementing an access control policy to isolate the Sales VLAN from the Management VLAN, ping testing revealed that traffic was being blocked in *both* directions — including from Management to Sales, which was never intended to be blocked. This case study documents the diagnosis and resolution.

---

## Environment
- **Devices involved:** HQ-R1 (Cisco IOS router)
- **VLANs involved:** VLAN 10 (Sales, 10.10.10.0/24), VLAN 30 (Management, 10.10.30.0/24)
- **Requirement:** Sales should not be able to reach Management. No restriction was intended on Management reaching Sales.

---

## Symptom
After applying the following ACL to the Sales-facing subinterface:

```
access-list 100 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255
access-list 100 permit ip any any

interface GigabitEthernet0/0.10
 ip access-group 100 in
```

Testing showed:
- ✅ `ping` from Sales-PC1 → Mgmt-PC1: **failed as expected** (this was the intended behavior)
- ❌ `ping` from Mgmt-PC1 → Sales-PC1: **also failed** (this was *not* expected — Management should have been able to reach Sales)

---

## Diagnosis

### Step 1 — Confirm the ACL was hitting on the unexpected traffic
Ran `show access-lists` on HQ-R1 after attempting the Mgmt → Sales ping:

```
Extended IP access list 100
    10 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 (34 matches)
    20 permit ip any any (112 matches)
```

The match counter on the deny line was incrementing even though the ping was initiated *from* Management, not Sales — a strong signal the ACL was catching something other than the initial request packet.

### Step 2 — Trace packet direction through the router
Walked through the actual packet flow for a single ping:

1. **Echo request**: Mgmt-PC1 (10.10.30.x) → Sales-PC1 (10.10.10.x). Enters router on the Mgmt subinterface (Gi0/0.30) — no ACL applied there, so it passes through and is forwarded to Sales.
2. **Echo reply**: Sales-PC1 (10.10.10.x) → Mgmt-PC1 (10.10.30.x). This reply packet has **source = Sales, destination = Mgmt**, and it enters the router inbound on **Gi0/0.10** — exactly where the deny rule is applied, and exactly what it matches.

### Root Cause
Extended ACLs in Cisco IOS are **stateless** — they evaluate each packet independently based on its source/destination address, with no awareness of whether it's part of an already-permitted conversation. The deny rule was written to block Sales-sourced traffic destined for Management, but it had no way to distinguish "Sales-initiated traffic" from "a reply to Management-initiated traffic" — both have the same source/destination address pattern. As a result, the ACL was silently discarding the return traffic for every Management-initiated ping, breaking a direction of communication that was never meant to be restricted.

---

## Resolution

Two options were considered:

1. **Reflexive ACLs** — would allow true one-directional isolation (Mgmt can initiate to Sales, Sales cannot initiate to Mgmt, and return traffic is dynamically permitted only for legitimately initiated sessions). More accurately matches the original requirement, but adds meaningful configuration complexity for a policy this small.

2. **Explicit bidirectional deny** — block both directions outright, since in practice most VLAN isolation policies of this kind are intended to fully segment two groups from each other rather than allow one-way asymmetric access.

**Decision:** Option 2 was implemented, as it matched the practical intent of the policy (segment Sales and Management entirely) without introducing reflexive ACL complexity that wasn't otherwise needed in this design.

```
access-list 100 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255
access-list 100 deny ip 10.10.30.0 0.0.0.255 10.10.10.0 0.0.0.255
access-list 100 permit ip any any
```

---

## Verification After Fix
- `ping` from Sales-PC1 → Mgmt-PC1: fails (expected)
- `ping` from Mgmt-PC1 → Sales-PC1: fails (now consistent, intentional)
- `ping` from Sales-PC1 → IT-PC1: succeeds — confirms the ACL is correctly scoped and not overly broad
- `show access-lists` confirms both deny lines are actively matching traffic, and the `permit ip any any` catch-all is matching everything else

---

## Key Takeaway
Standard and extended ACLs on Cisco IOS filter packet-by-packet with no concept of connection state. A deny rule written for one direction of traffic will also silently catch return traffic traveling the reverse path, which can produce ACL behavior that looks broken but is actually working exactly as configured. Achieving true one-directional access control requires either reflexive ACLs, a stateful firewall (e.g., a Zone-Based Firewall or an external firewall appliance), or accepting a fully bidirectional policy when that better matches the actual security intent.

---

## Addendum: Follow-Up Fix — ACL Not Applied to Both Interfaces

After implementing the bidirectional deny rules described in the Resolution section, initial testing still showed **Mgmt-PC1 → Sales-PC1 traffic passing through**, even though the second deny line (Mgmt → Sales) had been added to the ACL.

### Diagnosis
Checked which interfaces the ACL was actually applied to:

```
show ip interface GigabitEthernet0/0.10 | include access list
show ip interface GigabitEthernet0/0.30 | include access list
```

Only `Gi0/0.10` (Sales) showed the ACL applied inbound. `Gi0/0.30` (Management) had no ACL applied at all.

### Root Cause
An inbound ACL only inspects traffic **entering the router through that specific interface** — it has no relationship to a packet's destination. Applying the ACL solely on the Sales interface meant only Sales-sourced traffic was ever being evaluated. Traffic sourced from Management and destined for Sales entered the router through the Management interface (Gi0/0.30), which had no ACL applied, so the second deny line was never actually being evaluated against real traffic.

### Fix
Applied the same ACL inbound on both interfaces, since there were now two distinct source subnets to filter:

```
interface GigabitEthernet0/0.10
 ip access-group 100 in

interface GigabitEthernet0/0.30
 ip access-group 100 in
```

### Verification
```
show ip interface GigabitEthernet0/0.10 | include access list
show ip interface GigabitEthernet0/0.30 | include access list
```
Both interfaces now show `access list 100` applied. Re-ran ping tests:
- Sales-PC1 → Mgmt-PC1: fails (expected)
- Mgmt-PC1 → Sales-PC1: fails (expected, now correctly enforced)
- Sales-PC1 → IT-PC1: succeeds (confirms the ACL remains correctly scoped, not overly broad)

### Additional Takeaway
An ACL's logic being correct on paper doesn't guarantee correct behavior — where it's applied (which interface, and in which direction) is just as critical as what it contains. A best practice when troubleshooting "my ACL isn't working" is to check the interface assignment first (`show ip interface <int> | include access list`) before assuming the rule logic itself is wrong.
