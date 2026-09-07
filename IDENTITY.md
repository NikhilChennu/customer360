# Identity resolution — sample SQL

The identity service or CRM writes `identity.customer_id_map` from stored vendor identifiers. The warehouse does not match on name, date of birth, or address.

`internal_customer_id` is created in the warehouse. It is not written back to Salesforce, Amplitude, or core, and it is not the same value as `core_customer_id`.

---

## 1. Identity map

One row per person. Vendor columns may be NULL until that system is linked.

```sql
CREATE TABLE identity.customer_id_map (
    internal_customer_id     VARCHAR(64) NOT NULL,  -- warehouse key (PK)
    salesforce_customer_id   VARCHAR(64),           -- Salesforce Contact id
    amplitude_customer_id    VARCHAR(64),           -- Amplitude user_id
    core_customer_id         VARCHAR(64),           -- core person id; not the warehouse key
    status                   VARCHAR(16) NOT NULL,  -- ACTIVE | UNMAPPED | SEVERED
    updated_at               TIMESTAMP   NOT NULL,
    PRIMARY KEY (internal_customer_id)
);
```

```sql
-- Person A: present in Salesforce, Amplitude, and core
INSERT INTO identity.customer_id_map (
    internal_customer_id,
    salesforce_customer_id,
    amplitude_customer_id,
    core_customer_id,
    status,
    updated_at
) VALUES (
    'ic_1001',
    'salesforce_aaa',
    'amp_88',
    'core_42',
    'ACTIVE',
    GETDATE()
);

-- Person B: Salesforce and Amplitude only; no bank account yet (core_customer_id is NULL)
INSERT INTO identity.customer_id_map (
    internal_customer_id,
    salesforce_customer_id,
    amplitude_customer_id,
    core_customer_id,
    status,
    updated_at
) VALUES (
    'ic_1002',
    'salesforce_bbb',
    'amp_91',
    NULL,
    'ACTIVE',
    GETDATE()
);

-- Person C: Salesforce contact not yet published to 360 or CAR
INSERT INTO identity.customer_id_map (
    internal_customer_id,
    salesforce_customer_id,
    amplitude_customer_id,
    core_customer_id,
    status,
    updated_at
) VALUES (
    'ic_1003',
    'salesforce_ccc',
    NULL,
    NULL,
    'UNMAPPED',
    GETDATE()
);
```

Amplitude events with no `user_id` are not stored on this map. They remain in curated Amplitude tables only.

---

## 2. Assemble `product_360`

Derived marts keep each source’s customer id. Join them only through the map, and only for `ACTIVE` rows. Do not join Salesforce, Amplitude, or core to each other on name, email, or date of birth.

Person B (`core_customer_id` NULL) still returns a row; core columns are NULL until core is linked. Person C (`UNMAPPED`) is excluded.

```sql
SELECT
    m.internal_customer_id,
    pref.segment,
    pref.lifecycle,
    pref.assigned_rm_id,
    pref.opt_in_email,
    pref.opt_in_sms,
    pref.opt_in_push,
    port.product_flags,
    port.balances,
    eng.engagement_30d,
    cap.capability_flags,
    sup.support_summary
FROM identity.customer_id_map AS m
LEFT JOIN salesforce.derived_communication_preferences AS pref
       ON pref.salesforce_customer_id = m.salesforce_customer_id
LEFT JOIN salesforce.derived_support_case_summary AS sup
       ON sup.salesforce_customer_id = m.salesforce_customer_id
LEFT JOIN amplitude.derived_app_engagement_30d AS eng
       ON eng.amplitude_customer_id = m.amplitude_customer_id
LEFT JOIN amplitude.derived_app_capability_usage AS cap
       ON cap.amplitude_customer_id = m.amplitude_customer_id
LEFT JOIN core.derived_account_portfolio AS port
       ON port.core_customer_id = m.core_customer_id
WHERE m.status = 'ACTIVE';
```
