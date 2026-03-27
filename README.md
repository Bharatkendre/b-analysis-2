# Deal Lifecycle: Shipment & Vault Integration Flow

> This document shows how Shipment and Vault processing integrates with a Deal's lifecycle, from creation through settlement and archival.

---

## 1. High-Level Overview

```mermaid
flowchart TB
    subgraph DEAL_ENTRY["PHASE 1: DEAL ENTRY"]
        A1[Deal Created / Amended] --> A2{Commit?}
        A2 -->|Yes| A3[Set SHIPPING_PEND<br/>Set VAULT_PEND]
        A2 -->|No| A4[Set DEALS_INCOMPLETE<br/>Still set VAULT_PEND]
        A3 --> A5[Validate Vault Dates<br/>per Product Type]
        A5 --> A6[Check Inventory Limits<br/>at Vault1 and Vault2]
        A6 -->|Breach| A7[InventoryLimitsBreachException<br/>unless override comment provided]
        A6 -->|Pass| A8[Deal Saved]
    end

    subgraph SHIPMENT_ASSIGN["PHASE 2: SHIPMENT ASSIGNMENT"]
        B1[Create Shipment Record<br/>SH-YYYYMMDD-NNNNN] --> B2[Attach Deal Legs<br/>to Shipment Record]
        B2 --> B3[Set leg.shipmentRecords<br/>Set leg.shippingStatusId<br/>Set leg.shipmentValReq<br/>Set leg.insuranceValReq]
        B3 --> B4[Calculate Shipment Value<br/>Buy: shipment + insurance<br/>Sell: shipment only]
    end

    subgraph BO_APPROVAL["PHASE 3: BO APPROVAL"]
        C1[BO Verification Queue] --> C2{Approve?}
        C2 -->|Approve to Level 3| C3{Product is<br/>CONS/DISW/DISN<br/>ECIB/ECIS?}
        C3 -->|Yes| C4[updateInventoryForDealQueues<br/>Adjust vault inventory]
        C3 -->|No| C5[No vault impact]
        C2 -->|Reject from Level 3+| C6[Reverse vault inventory<br/>for same products]
        C4 --> C7[Generate Settlements<br/>PAYMENTS_PEND]
        C5 --> C7
    end

    subgraph VAULT_PROC["PHASE 4: VAULT PROCESSING"]
        D1[Vault Queue] --> D2[Select Shipment Record]
        D2 --> D3[Choose Destination<br/>Vault Status]
        D3 --> D4[Update Vault Inventory<br/>Double-entry pattern]
        D4 --> D5[Update Leg Vault Status]
        D5 --> D6[Update Deal Vault Status]
        D6 --> D7[Log DealStatusTrail<br/>+ AuditTrail]
    end

    subgraph SHIP_RELEASE["PHASE 5: SHIPMENT RELEASE"]
        E1[Release Shipment] --> E2{All deals<br/>BO approved?}
        E2 -->|No| E3[ERROR: OERR003]
        E2 -->|Yes| E4{Stale legs?}
        E4 -->|Yes| E5[ERROR: DERR9997]
        E4 -->|No| E6{CUF deals<br/>settled?}
        E6 -->|No| E7[ERROR: ERRVAULTCUF]
        E6 -->|Yes| E8[Set SHIPPING_RELEASED]
    end

    subgraph AUTO_SETTLE["PHASE 6: AUTO-SETTLEMENT"]
        F1[Auto-Settlement<br/>Triggered] --> F2{Shipping =<br/>RELEASED?}
        F2 -->|No| F3[Skip]
        F2 -->|Yes| F4{Vault =<br/>DISPATCHED or<br/>PIECE COUNT?}
        F4 -->|No| F3
        F4 -->|Yes| F5{Product =<br/>BKN_NORM or TCQ?<br/>DealType = INTRA?<br/>Customer = INTRA?}
        F5 -->|No| F3
        F5 -->|Yes| F6[PAYMENTS_SETTLED<br/>Auto-Settlement Complete]
    end

    A8 --> B1
    B4 --> C1
    C7 --> D1
    D7 --> E1
    E8 --> F1
```

---

## 2. Detailed Flow: Deal Creation

```mermaid
flowchart TD
    START([Deal Entry Screen]) --> NEW{New or<br/>Amendment?}

    NEW -->|New Deal| COMMIT{Commit<br/>or Draft?}
    COMMIT -->|Commit| SET_STATUS["Set all 4 statuses:<br/>DEALS_PEND<br/>PAYMENTS_PEND<br/>SHIPPING_PEND<br/>VAULT_PEND"]
    COMMIT -->|Draft| SET_INCOMPLETE["Set DEALS_INCOMPLETE<br/>Still set:<br/>PAYMENTS_PEND<br/>SHIPPING_PEND<br/>VAULT_PEND"]

    NEW -->|Amendment| EDIT_CHECK{"Is Deal<br/>Editable?"}
    EDIT_CHECK -->|"Shipping isDealEditable != Y"| ERR32["ERROR 00D000032:<br/>Cannot amend -<br/>shipping has progressed"]
    EDIT_CHECK -->|"Vault isDealEditable != Y"| ERR33["ERROR 00D000033:<br/>Cannot amend -<br/>vault has progressed"]
    EDIT_CHECK -->|Both editable| AMEND_TYPE{Deal Split?}

    AMEND_TYPE -->|Normal Amendment| RESET_AMEND["Reset DEALS_PEND<br/>Reset PAYMENTS_PEND<br/>Shipping NOT reset<br/>Reset VAULT_PEND"]
    AMEND_TYPE -->|Deal Split| RESET_SPLIT["Only reset:<br/>VAULT_PEND"]

    SET_STATUS --> VALIDATE_DATES
    SET_INCOMPLETE --> VALIDATE_DATES
    RESET_AMEND --> VALIDATE_DATES
    RESET_SPLIT --> VALIDATE_DATES

    VALIDATE_DATES[Validate Vault Dates<br/>per Product Type] --> DATE_OK{Dates Valid?}
    DATE_OK -->|No| DATE_ERR["ERROR 00D0DATV1/V2/G2:<br/>Vault date validation failed"]
    DATE_OK -->|Yes| INV_CHECK[Check Inventory Limits]

    INV_CHECK --> BUILD_HASH["Build cumulative<br/>inventory hash for<br/>Vault1 and Vault2"]
    BUILD_HASH --> CHECK_BREACH{"Inventory<br/>Breach?"}
    CHECK_BREACH -->|"No breach"| SAVE_DEAL[Deal Saved Successfully]
    CHECK_BREACH -->|"Breach + no override"| BREACH_ERR["InventoryLimitsBreachException:<br/>List of breached<br/>denominations"]
    CHECK_BREACH -->|"Breach + override comment"| SAVE_DEAL

    style ERR32 fill:#f99
    style ERR33 fill:#f99
    style DATE_ERR fill:#f99
    style BREACH_ERR fill:#f99
    style SAVE_DEAL fill:#9f9
```

---

## 3. Vault Date Validation Rules by Product

```mermaid
flowchart LR
    subgraph STANDARD["Standard Products<br/>BKN_NORM, TCQ_TCQN,<br/>BKN_CONS, BKN_UNRR,<br/>BKN_COLS, BKN_DISN"]
        S1["Required: trade, value,<br/>release, vault"]
        S2["trade <= vault"]
        S3{"Buy or Sell?"}
        S3 -->|Buy| S4["release <= vault"]
        S3 -->|Sell| S5["vault <= release"]
    end

    subgraph ECI_BS["ECI Buy/Sell<br/>BKN_ECIB, BKN_ECIS"]
        E1["Required: trade, value,<br/>release, vault, vault2"]
        E2["trade <= vault<br/>trade <= vault2"]
        E3["vault == vault2<br/>(must be equal)"]
    end

    subgraph ECI_RT["ECI Receive/Transfer<br/>BKN_ECIR, BKN_ECIT,<br/>BKN_DISW"]
        R1["Required: trade, vault ONLY"]
        R2["trade <= vault"]
    end

    subgraph CAEX["Cash Exchange<br/>BKN_CAEX, BKN_DISC"]
        C1["Required: trade, release,<br/>release2, vault, vault2"]
        C2["trade <= vault<br/>trade <= vault2"]
    end

    subgraph PHYFLOW["Physical Flow<br/>BKN_PHYFLOW"]
        P1["Required: trade, release,<br/>release2, vault, vault2"]
        P2["trade <= vault<br/>trade <= vault2"]
    end

    subgraph OFFSHORE["Offshore<br/>BKN_OFFS"]
        O1["vault = NULL<br/>vault2 = NULL<br/>(no vault dates)"]
    end
```

---

## 4. Shipment Assignment Flow

```mermaid
flowchart TD
    SEARCH[Search Deals] --> SELECT[Select Deal Legs]
    SELECT --> CREATE_SR{Create Shipment<br/>Record?}

    CREATE_SR -->|Manual| MANUAL["Create ShipmentRecords<br/>Set Method/Type/Basis/<br/>Arrangement/Shipper/Consignee<br/>Set routing legs"]
    CREATE_SR -->|Auto-Shipment| AUTO["Auto-create from<br/>region config properties<br/>AUTO_SHIPMENT_*"]

    MANUAL --> LINK_LEGS
    AUTO --> LINK_LEGS

    LINK_LEGS[Link Deal Legs<br/>to Shipment Record] --> CONCURRENT{"Leg already<br/>assigned to<br/>another shipment?"}
    CONCURRENT -->|Yes| CONC_ERR["ERROR:<br/>concurrentaccess.exception"]
    CONCURRENT -->|No| SET_LEG_FIELDS

    SET_LEG_FIELDS["Set on each BankNotesDealsLegs:<br/>shipmentRecords = SR<br/>shippingStatusId = SHIPPING workflow<br/>shipmentValReq = Y (always)<br/>insuranceValReq = Y (Buy) / N (Sell)"] --> CALC_VALUE

    CALC_VALUE["Calculate Values:<br/>Buy: shipmentUSD += convertToUSD(amount)<br/>Buy: insuranceUSD += convertToUSD(amount)<br/>Sell: shipmentUSD += convertToUSD(amount)<br/>Sell: insuranceUSD += 0<br/><br/>Convert all to base currency<br/>ACCUMULATE onto existing ShipmentRecords values"] --> LINK_COMMISSION

    LINK_COMMISSION["Link Commission Deals:<br/>BankNotesDeals.commissionShipmentRecordId<br/>= shipmentRecord.finId"] --> VALIDATE

    VALIDATE{Validate} --> DONE[Shipment Created<br/>Status: PENDING or ATTACHED]

    style CONC_ERR fill:#f99
    style DONE fill:#9f9
```

---

## 5. Vault Processing State Machine

```mermaid
stateDiagram-v2
    [*] --> VAULT_PEND

    state "OUTBOUND" as outbound {
        VAULT_PEND --> VAULT_PACKING: status only
        VAULT_PACKING --> VAULT_PACKED: status only
        VAULT_PACKED --> VAULT_DISPATCHED: INVENTORY IMPACT\n(Sell legs debit source vault)
        VAULT_DISPATCHED --> VAULT_BULK_COUNT: INVENTORY IMPACT\n(Notes to OTHERS sub-vault)
        VAULT_BULK_COUNT --> VAULT_PIECE_COUNT: INVENTORY IMPACT\n(OTHERS to MAIN_STOCK)
    }

    state "INBOUND" as inbound {
        VAULT_PEND2: VAULT_PEND
        VAULT_RECEIVED2: VAULT_RECEIVED
        VAULT_BULK2: VAULT_BULK_COUNT
        VAULT_PIECE2: VAULT_PIECE_COUNT

        VAULT_PEND2 --> VAULT_RECEIVED2: status only
        VAULT_RECEIVED2 --> VAULT_BULK2: INVENTORY IMPACT
        VAULT_BULK2 --> VAULT_PIECE2: INVENTORY IMPACT
    }

    state "LOCAL / CROSS-BORDER" as local {
        VAULT_PEND3: VAULT_PEND
        VAULT_PACKED3: VAULT_PACKED
        VAULT_DISPATCHED3: VAULT_DISPATCHED
        VAULT_RECEIVED3: VAULT_RECEIVED
        VAULT_BULK3: VAULT_BULK_COUNT
        VAULT_PIECE3: VAULT_PIECE_COUNT

        VAULT_PEND3 --> VAULT_PACKED3: status only
        VAULT_PACKED3 --> VAULT_DISPATCHED3: SELL legs only\n(source vault debited)
        VAULT_DISPATCHED3 --> VAULT_RECEIVED3: status only
        VAULT_RECEIVED3 --> VAULT_BULK3: BUY legs only\n(dest vault credited)
        VAULT_BULK3 --> VAULT_PIECE3: BUY legs only
    }

    note right of outbound
        All transitions are REVERSIBLE.
        Reversal negates the inventory amounts.
        Multi-step transitions available (JP region):
        PEND -> PIECE COUNT in one call.
    end note
```

---

## 6. Vault Inventory Double-Entry Pattern

```mermaid
flowchart LR
    subgraph "BUY Deal Legs"
        BUY_FROM["FROM Vault (Vault1)<br/>+amount"]
        BUY_TO["TO Vault (Vault2)<br/>-amount"]
    end

    subgraph "SELL Deal Legs"
        SELL_FROM["FROM Vault (Vault1)<br/>-amount"]
        SELL_TO["TO Vault (Vault2)<br/>+amount"]
    end

    subgraph "Vault Keys Per Leg"
        K0["Key 0: vault1Id:denomId:typeId:ccyId<br/>(Source Vault)"]
        K1["Key 1: vault2Id:denomId:typeId:ccyId<br/>(Destination Vault)"]
        K2["Key 2: mainVault_OTHERS:denomId:typeId:ccyId<br/>(Staging Vault)"]
        K3["Key 3: ECI_OTHERS:denomId:typeId:ccyId<br/>(ECI Staging)"]
        K4["Key 4: mainVault_DISCREPANCY:denomId:typeId:ccyId<br/>(Discrepancy Vault)"]
    end

    subgraph "Dispatch (Outbound)"
        D1["Sell legs:<br/>Key 0 (Source) -= amount<br/>Key 1 (Dest) += amount"]
    end

    subgraph "Bulk Count"
        BC1["Notes to OTHERS:<br/>Key 2 (Others) += amount"]
    end

    subgraph "Piece Count"
        PC1["OTHERS to MAIN_STOCK:<br/>Key 2 (Others) -= amount<br/>Key 0 (Main) += amount"]
    end
```

---

## 7. BO Approval Vault Impact

```mermaid
flowchart TD
    BO_QUEUE[BO Verification Queue] --> APPROVE{Action?}

    APPROVE -->|"Approve to Level 3"| CHECK_PRODUCT{Product Type?}
    CHECK_PRODUCT -->|"BKN_CONS, BKN_DISW,<br/>BKN_DISN, TCQ_DISN,<br/>TCQ_DISW, BKN_ECIB,<br/>BKN_ECIS"| UPDATE_INV["updateInventoryForDealQueues(false)<br/>Add/subtract vault inventory<br/>using construcAmtToUpdateMap"]
    CHECK_PRODUCT -->|"All other products"| NO_INV["No vault inventory impact<br/>on approval"]

    APPROVE -->|"Reject from Level 3+"| CHECK_PRODUCT2{Same product<br/>list?}
    CHECK_PRODUCT2 -->|Yes| REVERSE_INV["updateInventoryForDealQueues(true)<br/>REVERSE vault inventory<br/>using construcAmtToUpdateReversalOutBoundMap<br/>AuditTrail entity: BOAPPR / BO_REJECT"]
    CHECK_PRODUCT2 -->|No| NO_INV2["No vault impact<br/>on rejection"]

    APPROVE -->|"Reject + Cash Settlement"| CASH_REV{"Settlement status =<br/>RELEASED or SETTLED?"}
    CASH_REV -->|Yes| CASH_REVERSE["reverseInventoryForCashReleasedDeals:<br/>Buy: vault += netSetlAmt<br/>Sell: vault -= netSetlAmt<br/>Denom: settlCcy__MIXED_BKN<br/>Type: FIT"]
    CASH_REV -->|No| NO_CASH["No cash reversal needed"]

    UPDATE_INV --> GEN_SETL["Generate Settlements<br/>if genSettlements = Y"]
    NO_INV --> GEN_SETL

    style UPDATE_INV fill:#ff9
    style REVERSE_INV fill:#f99
    style CASH_REVERSE fill:#f99
```

---

## 8. Shipment Release and Auto-Settlement Flow

```mermaid
flowchart TD
    RELEASE_BTN([Release Shipment Button]) --> VALIDATE_BO

    VALIDATE_BO["For each deal in shipment:<br/>Check BO status workflowLevel > 2<br/>(SELL legs only - BUY legs SKIPPED)"] --> BO_OK{All verified?}
    BO_OK -->|No| ERR_BO["ERROR OERR003:<br/>BO status not verified"]
    BO_OK -->|Yes| CHECK_STALE

    CHECK_STALE["Check stale deal legs:<br/>Compare version in leg vs<br/>current deal version"] --> STALE_OK{Any stale?}
    STALE_OK -->|Yes| ERR_STALE["ERROR DERR9997:<br/>Stale deal legs detected"]
    STALE_OK -->|No| CHECK_CUF

    CHECK_CUF["CUF Deal Validation:<br/>CashUpFrontCheckingService<br/>.isValidForShipmentRelease()"] --> CUF_OK{CUF OK?}
    CUF_OK -->|No| ERR_CUF["ERROR ERRVAULTCUF:<br/>CUF deals not settled"]
    CUF_OK -->|Yes| DO_RELEASE

    DO_RELEASE["Set ShipmentRecords.workflowStates<br/>= SHIPPING_RELEASED<br/><br/>For each deal:<br/>Set DealsStatus.shippingStatusId<br/>= SHIPPING_RELEASED"] --> AUTO_SETTLE

    AUTO_SETTLE["processAutoSettlement(shipmentId)"] --> GET_DEALS["getDealsFromShipment()"]

    GET_DEALS --> CHAIN["Validation Chain<br/>for each deal:"]

    CHAIN --> V1{"1. AutoSettlement<br/>Required?<br/>(region property)"}
    V1 -->|No| SKIP1([Skip - Not Auto-Settleable])
    V1 -->|Yes| V2{"2. Product =<br/>BKN_NORM or TCQ?"}
    V2 -->|No| SKIP2([Skip])
    V2 -->|Yes| V3{"3. DealType =<br/>BKN_NORM_INTRA<br/>or TCQ_*?"}
    V3 -->|No| SKIP3([Skip])
    V3 -->|Yes| V4{"4. CustomerType<br/>contains<br/>INTRA_COMPANY?"}
    V4 -->|No| SKIP4([Skip])
    V4 -->|Yes| V5{"5. ShipmentStatus<br/>= SHIPPING_RELEASED?"}
    V5 -->|No| SKIP5([Skip])
    V5 -->|Yes| V6{"6. VaultStatus =<br/>VAULT_DISPATCHED<br/>or VAULT_PIECE COUNT?"}
    V6 -->|No| SKIP6([Skip])
    V6 -->|Yes| SETTLE["Auto-Settle Deal:<br/>PAYMENTS_SETTLED<br/>(4-eyes check SKIPPED)"]

    style ERR_BO fill:#f99
    style ERR_STALE fill:#f99
    style ERR_CUF fill:#f99
    style DO_RELEASE fill:#9f9
    style SETTLE fill:#9f9
```

---

## 9. Deal Cancellation Guards

```mermaid
flowchart TD
    CANCEL_REQ([Cancel Deal Request]) --> CHECK_DEAL{"Deal Status<br/>isDealEditable = Y?"}
    CHECK_DEAL -->|No| ERR40["ERROR 00D000040"]
    CHECK_DEAL -->|Yes| CHECK_PAY{"Payment Status<br/>isDealEditable = Y?"}
    CHECK_PAY -->|No| ERR41["ERROR 00D000041"]
    CHECK_PAY -->|Yes| CHECK_SHIP{"Shipping Status<br/>isDealEditable = Y?"}
    CHECK_SHIP -->|No| ERR42["ERROR 00D000042:<br/>Shipping has progressed"]
    CHECK_SHIP -->|Yes| CHECK_VAULT{"Vault Status<br/>isDealEditable = Y?"}
    CHECK_VAULT -->|No| ERR43["ERROR 00D000043:<br/>Vault has progressed"]
    CHECK_VAULT -->|Yes| CAN_CANCEL["Create Cancelled Version<br/>Set action = CANCEL<br/>Set status = CANCELLED<br/><br/>Shipment data NOT copied<br/>to cancelled version"]

    style ERR40 fill:#f99
    style ERR41 fill:#f99
    style ERR42 fill:#f99
    style ERR43 fill:#f99
    style CAN_CANCEL fill:#9f9
```

---

## 10. Deal Split Guards

```mermaid
flowchart TD
    SPLIT_REQ([Split Deal Request]) --> CHECK_S{"Shipping Status<br/>= SHIPPING_PEND?"}
    CHECK_S -->|No| ERR50["ERROR 00D000050:<br/>Cannot split -<br/>shipping not pending"]
    CHECK_S -->|Yes| CHECK_V{"Vault Status in<br/>[VAULT_PEND,<br/>VAULT_PACKING,<br/>VAULT_RECEIVED]?"}
    CHECK_V -->|No| ERR51["ERROR 00D000051:<br/>Cannot split -<br/>vault already processed"]
    CHECK_V -->|Yes| CHECK_VAULT_MATCH{"Vault1 of original<br/>= Vault1 of split?"}
    CHECK_VAULT_MATCH -->|No| ERR154["ERROR 00OD00154:<br/>Vault mismatch"]
    CHECK_VAULT_MATCH -->|Yes| SPLIT_OK["Split Allowed<br/>Vault status reset to VAULT_PEND"]

    style ERR50 fill:#f99
    style ERR51 fill:#f99
    style ERR154 fill:#f99
    style SPLIT_OK fill:#9f9
```

---

## 11. Order-to-Deal Conversion (Shipment Data Transfer)

```mermaid
flowchart TD
    ORDER[BankNotesOrdersLegs<br/>with shipmentRecordId<br/>and vaultStatusId] --> CONVERT["Convert Order to Deal<br/>updateDealNoInOrders()"]

    CONVERT --> TRANSFER["Transfer to BankNotesDealsLegs:<br/>1. shipmentRecords = order.shipmentRecordId<br/>2. shippingStatusId = order.shippingStatus<br/>3. vaultStatusId = order.vaultStatusId"]

    TRANSFER --> UPDATE_DEAL["Update DealsStatus:<br/>workflowStatesByVaultStatusId = order.vaultStatus<br/>workflowStatesByShippingStatusId = order.shippingStatus"]

    UPDATE_DEAL --> UPDATE_PKG["Update PackingList records:<br/>dealNo = new deal number<br/>dealLegNumber = new leg number<br/>versionNo = deal version"]

    UPDATE_PKG --> RESULT["Deal now has:<br/>- Same shipment record as order<br/>- Same vault/shipping status as order<br/>- PackingList entries updated"]

    style RESULT fill:#9f9
```

---

## 12. EOD Processing (Shipment/Vault Steps)

```mermaid
flowchart TD
    EOD_START([EOD Start]) --> STEP2["Step 2: RESET SEQUENCE<br/>Reset SQ_BLS_SHIPMENT_FINID<br/>Reset SQ_BLS_PACKING_LIST"]

    STEP2 --> STEP30["Step 30: MARK VAULT INVENTORY<br/>Clone today's VaultsInvCash to nextBizDate<br/>Soft-delete today's records"]

    STEP30 --> HOLIDAY_CHECK{"Is today a<br/>shipping holiday?"}

    HOLIDAY_CHECK -->|Yes| SKIP_CHARGES["Skip charge/invoice<br/>propagation steps"]
    HOLIDAY_CHECK -->|No| STEP32

    STEP32["Step 32: PROPAGATE SHIPMENT CHARGES<br/>saveShipmentChargesToDeal()<br/>Proportional: dealCharge = (dealUSD/totalUSD) * totalCharges"] --> STEP33

    STEP33["Step 33: SAVE SHIPMENT INVOICE<br/>saveShipmentInvoiceToDeal()<br/>Same proportional formula<br/>Both write to SAME field:<br/>BankNotesDeals.shippingChargeAmount"] --> STEP34

    STEP34["Step 34: UPDATE SHIPPING COMMISSIONS<br/>processDailyCommissions()<br/>Shipment method is discriminator<br/>for commission config lookup"] --> ACC_ENTRIES

    ACC_ENTRIES["Accounting Entries:<br/>vaultReversalEntries()<br/>shipmentReversalEntries()"] --> STEP45

    SKIP_CHARGES --> STEP34

    STEP45["Step 45: UPDATE SHIPMENT CHARGE STATUS<br/>Mark fully-propagated records<br/>as SHIPPING_CHARGES_COMPLETE"] --> EOD_END([EOD Complete])

    style STEP30 fill:#ff9
    style STEP32 fill:#ff9
    style STEP33 fill:#ff9
    style STEP45 fill:#ff9
```

---

## 13. Complete Deal Timeline (Single View)

```
TIME ──────────────────────────────────────────────────────────────────────────────────►

DEAL      ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────┐
STATUS    │ CREATED  │─►│   PEND   │─►│ BO VERIFY │─►│ APPROVED │─►│  MATURE  │
          └─────────┘  └──────────┘  └───────────┘  └──────────┘  └──────────┘

SHIPPING  ┌─────────┐               ┌──────────┐   ┌──────────┐
STATUS    │  PEND   │──────────────►│ ATTACHED │──►│ RELEASED │──► triggers auto-settlement
          └─────────┘               └──────────┘   └──────────┘

VAULT     ┌─────────┐  ┌────────┐  ┌────────┐  ┌────────────┐  ┌──────────┐  ┌────────────┐
STATUS    │  PEND   │─►│PACKING │─►│ PACKED │─►│ DISPATCHED │─►│BULK COUNT│─►│PIECE COUNT │
          └─────────┘  └────────┘  └────────┘  └────────────┘  └──────────┘  └────────────┘
                                                      │               │             │
                                                 INVENTORY       INVENTORY      INVENTORY
                                                  IMPACT          IMPACT         IMPACT

SETTLEMENT                                                    ┌─────────┐  ┌─────────┐
STATUS                                                        │RELEASED │─►│ SETTLED │
                                                              └─────────┘  └─────────┘

INVENTORY                                      ┌──────────────────────────────────────┐
CHECKS     inventory limits ◄──────────────────│ On deal save (commit)                │
           checked at Vault1                   │ InventoryLimitsBreachException       │
           and Vault2                          │ if negative position would result    │
                                               └──────────────────────────────────────┘

SPECIAL    ┌──────────────────────────────────────────────────────────────────────────┐
PRODUCTS   │ BKN_CONS/DISW/DISN/ECIB/ECIS/TCQ_DISN/TCQ_DISW:                       │
           │ ► Vault inventory adjusted on BO APPROVAL (Level 3)                     │
           │ ► Vault inventory REVERSED on BO REJECTION (from Level 3+)              │
           └──────────────────────────────────────────────────────────────────────────┘

EOD        ┌──────────────────────────────────────────────────────────────────────────┐
           │ 1. Reset shipment/packing sequences                                     │
           │ 2. Clone vault inventory to next business date                          │
           │ 3. Proportionally propagate shipment charges to deals (skip on holiday) │
           │ 4. Proportionally propagate invoices to deals (skip on holiday)         │
           │ 5. Process commissions (shipment method = discriminator)                │
           │ 6. Generate vault/shipment reversal accounting entries                  │
           │ 7. Mark fully-propagated charge statuses as complete                    │
           └──────────────────────────────────────────────────────────────────────────┘

GUARDS     ┌──────────────────────────────────────────────────────────────────────────┐
           │ CANCEL: Blocked if shipping != PEND or vault not in [PEND/PACKING/RECV] │
           │ AMEND:  Blocked if shipping/vault isDealEditable != Y                   │
           │ SPLIT:  Requires SHIPPING_PEND + vault in [PEND/PACKING/RECEIVED]       │
           │         Vault1 must match between original and split deals              │
           └──────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Products and Their Vault Integration

```
PRODUCT            VAULT1   VAULT2      VAULT      INVENTORY     SPECIAL
                                        DATES      ON BO APPR    NOTES
───────────────────────────────────────────────────────────────────────────
BKN_NORM           Yes      No          1 date     No            Standard flow
TCQ_TCQN           Yes      No          1 date     No            Standard flow
BKN_CAEX           Yes      Yes(swap)   2+2 dates  No            Cash exchange, dual legs
BKN_DISC/TCQ_DISC  Yes      Yes(disc)   2 dates    No            Discrepancy vault key
BKN_CONS           Yes      No          1 date     YES           Consolidation
BKN_CONT           Yes      Yes(cons)   2 dates    No            Consignment top-up
BKN_CONR           Yes      Yes(cons)   2 dates    No            Consignment return
BKN_ECIB           No*      Yes(ECI)    2 dates    YES           *Does not impact vault
BKN_ECIS           No*      Yes(ECI)    2 dates    YES           *Does not impact vault
BKN_ECIR           Yes      No          1 date     No            ECI receive
BKN_ECIT           Yes      No          1 date     No            ECI transfer
BKN_DISN           No*      No          1 date     YES           *Discrepancy notes
BKN_DISW           No*      No          1 date     YES           *Discrepancy wrapper
BKN_UNRR           No*      No          1 date     No            *Unreported
BKN_COLS           No*      No          1 date     No            *Consolidation sell
BKN_OFFS           Yes      No          NO dates   No            Offshore (no vault dates)
BKN_INTERVAULT     Yes      No          1 date     No            Inter-vault transfer
BKN_PHYFLOW        Yes      No          2+2 dates  No            Physical flow
BKN_EFTSETTLE      Yes      No          1 date     No            EFT settlement
BKN_PHYSETTLE      Yes      No          1 date     No            Physical settlement
───────────────────────────────────────────────────────────────────────────
* "No" vault impact means the MAIN vault is not impacted, but specific
  sub-vaults (DISCREPANCY, OTHERS) may still be updated.
  BKN_DISN/DISW update only the discrepancy vault key.
```
