---
title: Handle ingestion delay in Microsoft Sentinel | Microsoft Docs
description:  Handle ingestion delay in Microsoft Sentinel scheduled analytics rules.
author: limwainstein
ms.topic: how-to
ms.date: 01/09/2023
ms.author: lwainstein
---

# Handle ingestion delay in scheduled analytics rules

While Microsoft Sentinel can ingest data from [various sources](connect-data-sources.md), ingestion time for each data source may differ in different circumstances.

This article describes how ingestion delay might impact your scheduled analytics rules and how you can fix them to cover these gaps.

## Why delay is significant

For example, you might write a custom detection rule, setting the **Run query every** and **Lookup data from the last** fields to have the rule run every five minutes, looking up data from those last five minutes:

:::image type="content" source="media/ingestion-delay/create-rule.png" alt-text="Screenshot showing the Analytics Rule Wizard - Create new rule window." border="false":::

The **Lookup data from the last** field defines a setting known as a *look-back* period. Ideally, when there's no delay, this detection misses no events, as shown in the following diagram:

:::image type="content" source="media/ingestion-delay/look-back.png" alt-text="Diagram showing a five-minute look-back window." border="false":::

The event arrives as it's generated, and is included in the *lookback* period.

Now, assume there's some delay for your data source. For this example, let's say the event was *ingested* two minutes after it was *generated*. The delay is two minutes:

:::image type="content" source="media/ingestion-delay/look-back-delay.png" alt-text="Diagram showing five-minute look back windows with a delay of two minutes." border="false":::

The event is generated within the first look-back period, but isn't ingested in your Microsoft Sentinel workspace on the first run. The next time the scheduled query runs, it ingests the event, but the time-generated filter removes the event because it happened more than five minutes ago. In this case, **the rule does not fire an alert**.

## How to handle delay

> [!NOTE]
>
> You can either solve the issue using the process described below, or implement Microsoft Sentinel's near-real-time detection (NRT) rules. For more information, see [Detect threats quickly with near-real-time (NRT) analytics rules in Microsoft Sentinel](near-real-time-rules.md).
> 

To solve the issue, you need to know the delay for your data type. For this example, you already know the delay is two minutes. 

For your own data, you can understand delay using the Kusto `ingestion_time()` function, and calculating the difference between **TimeGenerated** and the ingestion time. For more information, see [Calculate ingestion delay](#calculate-ingestion-delay).

After determining the delay, you can address the problem as follows:

- **Increase the look-back period**. Basic intuition tells you that increasing the look-back period size will help. Since your look-back period is five minutes and your delay is two minutes, setting the look-back period to *seven* minutes will help address this problem. For example, in your rule settings:

    :::image type="content" source="media/ingestion-delay/set-look-back.png" alt-text="Screenshot that shows setting the look-back window to seven minutes.":::

    The following diagram shows how the look-pack period now contains the missed event:

    :::image type="content" source="media/ingestion-delay/longer-look-back.png" alt-text="Diagram that shows seven-minute look back windows with a delay of two minutes." border="false":::

- **Handle duplication**. Only increasing the look-back period can create duplication, because the look-back windows now overlap. For example, a different event may look as shown in the following diagram:

    :::image type="content" source="media/ingestion-delay/overlapping-look-back.png" alt-text="Diagram showing how overlapping look-back windows create duplication." border="false":::

    Since the event **TimeGenerated** value is found in both look-back periods, the event fires two alerts. You need to find a way to solve the duplication.

- **Associate the event to a specific look-back period**. In the first example, you missed events because your data wasn't ingested when the scheduled query ran. You extended the look-back to include the event, but this caused duplication. You have to associate the event to the window you extended to contain it.

    Do this by setting `ingestion_time() > ago(5m)`, instead of the original rule `look-back = 5m`. This setting associates the event to the first look-back window. For example:

    :::image type="content" source="media/ingestion-delay/ago-restriction.png" alt-text="Diagram showing how setting the ago restriction avoids duplication." border="false":::

    The ingestion time restriction now trims the extra two minutes you added to the look-back period. And for the first example, the second run look-back period now captures the event:

    :::image type="content" source="media/ingestion-delay/ago-restriction-capture.png" alt-text="Diagram showing how setting the ago restriction captures the event." border="false":::

The following sample query summarizes the solution for solving ingestion delay issues:

```kusto
let ingestion_delay = 2min;
let rule_look_back = 5min;
CommonSecurityLog
| where TimeGenerated >= ago(ingestion_delay + rule_look_back)
| where ingestion_time() > ago(rule_look_back)
```

*Note*: the strategy above will only resolve ingestion delay issue for cases where you are searching for a single event. In cases where there are multiple events you are correlating on there is a possible race condition if the ingestion delay for the events you mean to correlate on differs. Consider the following case:

There exist two alerts, a1 and a2, that are intended to be correlated together with no defined upper limit for the time (based on the time the event actually occurred on the system) that has elapsed between them (the time period for which correlation should occur in this hypothetical case isn’t important to the problem description so long as a1 comes after a2). The search uses the design pattern above, executing every 15 minutes and only looking at events that have been indexed in the last 15 minutes.

Suppose a1 and a2 have the following ingestion and actual event times:

a1 raw event time: 10:09:03, a1 ingestion time: 10:15:03
a2 raw event time: 10:10:00, a2 ingestion time: 10:13:00

And the search executes at 10:00, 10:15 and 10:30.

In the case above event a1 has a raw event time (the time the event actually occurred on the system) of 10:09:03 and an ingestion time (the time in which the event was ingested by Splunk) of 10:15:03 and a2 has a raw event time (the time the event actually occurred on the system) of 10:10:00 and an ingestion time (the time in which the event was ingested by Splunk) of 10:13:00.

If the ingestion delay were the same for both events the design pattern recommended above would suffice to account for the delay. However, given the ingestion delay varies between the two events the search would fail to pick up a case which it should. Put generally, because the ingestion delay may vary between the two events, it’s still possible, even though both events will still be considered by a search at some point, that, if the time they are generated and indexed falls close to the time a search will be kicked off, that they are considered by different search execution and therefor correlation does not occur.

You will have to account for this type of variance in ingestion delays for searches which involve more than a single event.



## Calculate ingestion delay

By default, Microsoft Sentinel scheduled alert rules are configured to have a 5-minute look-back period. However, each data source may have its own, individual ingestion delay. When joining multiple data types, you must understand the different delays for each data type in order to configure the look-back period correctly.

The **Workspace Usage Report**, provided in Microsoft Sentinel out-of-the-box, includes a dashboard that shows latency and delays for the different data types flowing into your workspace.

For example:

:::image type="content" source="media/ingestion-delay/end-to-end-latency.png" alt-text="Screenshot of the Workspace Usage Report showing End to End Latency by table":::


## Next steps

For more information, see:

- [Create custom analytics rules to detect threats](detect-threats-custom.md)
- [Customize alert details in Azure Sentinel](customize-alert-details.md)
- [Manage template versions for your scheduled analytics rules in Azure Sentinel](manage-analytics-rule-templates.md)
- [Use the health monitoring workbook](monitor-data-connector-health.md)
- [Log data ingestion time in Azure Monitor](../azure-monitor/logs/data-ingestion-time.md)
