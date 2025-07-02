{
  "reportKey": 24,
  "reportName": "prefix_Sales",
  "reportDescription": "prefix_Quarterly sales breakdown",
  "commentary": "prefix_Reviewed by finance team",
  "workSpaceKey": 3,
  "approvedBy": "prefix_john.doe@company.com",
  "reportStatus": "prefix_Approved",
  "reportStatusName": "prefix_Final Approval",
  "reportDatasets": [101, 102],
  "eventTypes": [
    {
      "eventTypeKey": 1,
      "nodes": [
        {
          "eventNodeId": "prefix_201",
          "eventNodeName": "prefix_North Region"
        }
      ],
      "subscription": {
        "reportDeliveryModeKey": 1,
        "reportDeliveryFormatKey": 2,
        "fileShareLocation": "prefix_\\\\fileshare\\reports\\q2",
        "emailTo": "prefix_sales@company.com",
        "emailCC": "prefix_manager@company.com",
        "emailSubject": "prefix_Q2 Sales Report"
      }
    },
    {
      "eventTypeKey": 2,
      "nodes": [
        {
          "eventNodeId": "prefix_202",
          "eventNodeName": "prefix_South Region"
        }
      ],
      "subscription": {
        "reportDeliveryModeKey": 2,
        "reportDeliveryFormatKey": 1,
        "fileShareLocation": null,
        "emailTo": "prefix_south.sales@company.com",
        "emailCC": null,
        "emailSubject": "prefix_South Region Q2 Report"
      }
    }
  ]
}




private async Task AddReportEventsAndSubscriptionsAsync(int reportKey, IEnumerable<PbiReportEventRequest> eventRequests)
{
    const string userName = "system";
    if (eventRequests == null || !eventRequests.Any()) return;

    var now = DateTime.UtcNow;

    var existingEventTypes = await GetExistingEventTypesAsync(reportKey);
    var (newEvents, updatedEvents, deletedEvents, eventKeyMap) = UpsertEventTypes(eventRequests, existingEventTypes, reportKey, userName, now);

    _context.PbiReportEventTypes.AddRange(newEvents);
    _context.PbiReportEventTypes.UpdateRange(updatedEvents);
    _context.PbiReportEventTypes.RemoveRange(deletedEvents);

    await SaveChangesSafelyAsync("events");

    var existingSubscriptions = await GetExistingSubscriptionsAsync(reportKey);
    var (newSubs, updatedSubs, deletedSubs) = UpsertSubscriptions(eventRequests, existingSubscriptions, eventKeyMap, reportKey, userName, now);

    _context.PbiReportSubscription.AddRange(newSubs);
    _context.PbiReportSubscription.UpdateRange(updatedSubs);
    _context.PbiReportSubscription.RemoveRange(deletedSubs);

    await SaveChangesSafelyAsync("subscriptions");
}

private Task<List<PbiReportEventTypes>> GetExistingEventTypesAsync(int reportKey)
{
    return _context.PbiReportEventTypes
        .Where(e => e.ReportKey == reportKey)
        .ToListAsync();
}

private (List<PbiReportEventTypes> newEvents,
         List<PbiReportEventTypes> updatedEvents,
         List<PbiReportEventTypes> deletedEvents,
         Dictionary<(int, int), int> eventKeyMap)
UpsertEventTypes(IEnumerable<PbiReportEventRequest> requests,
                 List<PbiReportEventTypes> existing,
                 int reportKey,
                 string userName,
                 DateTime now)
{
    var seenKeys = new HashSet<(int, int)>();
    var newList = new List<PbiReportEventTypes>();
    var updatedList = new List<PbiReportEventTypes>();

    foreach (var req in requests)
    {
        var key = (reportKey, req.EventTypeKey);
        seenKeys.Add(key);

        var match = existing.FirstOrDefault(e => e.ReportKey == reportKey && e.EventTypeKey == req.EventTypeKey);

        var newNode = req.Nodes?.FirstOrDefault();

        if (match != null)
        {
            bool updated = false;
            if (match.NodeId != newNode?.EventNodeId)
            {
                match.NodeId = newNode?.EventNodeId;
                updated = true;
            }

            if (match.EventNodeName != newNode?.EventNodeName)
            {
                match.EventNodeName = newNode?.EventNodeName;
                updated = true;
            }

            if (updated)
                updatedList.Add(match);
        }
        else
        {
            newList.Add(new PbiReportEventTypes
            {
                ReportKey = reportKey,
                EventTypeKey = req.EventTypeKey,
                NodeId = newNode?.EventNodeId,
                EventNodeName = newNode?.EventNodeName,
                CreatedAt = now,
                CreatedBy = userName
            });
        }
    }

    var toDelete = existing
        .Where(e => !seenKeys.Contains((e.ReportKey, e.EventTypeKey)))
        .ToList();

    var map = existing.Concat(newList)
        .ToDictionary(e => (e.ReportKey, e.EventTypeKey), e => e.ReportEventKey);

    return (newList, updatedList, toDelete, map);
}

private Task<List<PbiReportSubscription>> GetExistingSubscriptionsAsync(int reportKey)
{
    return _context.PbiReportSubscription
        .Where(s => s.ReportKey == reportKey)
        .ToListAsync();
}

private (List<PbiReportSubscription> newSubs,
         List<PbiReportSubscription> updatedSubs,
         List<PbiReportSubscription> deletedSubs)
UpsertSubscriptions(IEnumerable<PbiReportEventRequest> requests,
                    List<PbiReportSubscription> existing,
                    Dictionary<(int, int), int> eventKeyMap,
                    int reportKey,
                    string userName,
                    DateTime now)
{
    var seenSubKeys = new HashSet<int>();
    var newList = new List<PbiReportSubscription>();
    var updatedList = new List<PbiReportSubscription>();

    foreach (var req in requests)
    {
        if (req.Subscription == null)
            continue;

        var key = (reportKey, req.EventTypeKey);
        if (!eventKeyMap.TryGetValue(key, out int reportEventKey))
            continue;

        seenSubKeys.Add(reportEventKey);

        var existingSub = existing.FirstOrDefault(s => s.ReportEventKey == reportEventKey);
        if (existingSub != null)
        {
            bool updated = false;

            if (existingSub.ReportDeliveryModeKey != req.Subscription.ReportDeliveryModeKey)
            {
                existingSub.ReportDeliveryModeKey = req.Subscription.ReportDeliveryModeKey;
                updated = true;
            }

            if (existingSub.ReportDeliveryFormatKey != req.Subscription.ReportDeliveryFormatKey)
            {
                existingSub.ReportDeliveryFormatKey = req.Subscription.ReportDeliveryFormatKey;
                updated = true;
            }

            if (existingSub.FileShareLocation != req.Subscription.FileShareLocation)
            {
                existingSub.FileShareLocation = req.Subscription.FileShareLocation;
                updated = true;
            }

            if (existingSub.EmailTo != req.Subscription.EmailTo)
            {
                existingSub.EmailTo = req.Subscription.EmailTo;
                updated = true;
            }

            if (existingSub.EmailCC != req.Subscription.EmailCC)
            {
                existingSub.EmailCC = req.Subscription.EmailCC;
                updated = true;
            }

            if (existingSub.EmailSubject != req.Subscription.EmailSubject)
            {
                existingSub.EmailSubject = req.Subscription.EmailSubject;
                updated = true;
            }

            if (updated)
            {
                existingSub.IsActive = true;
                updatedList.Add(existingSub);
            }
        }
        else
        {
            newList.Add(new PbiReportSubscription
            {
                ReportKey = reportKey,
                ReportEventKey = reportEventKey,
                ReportDeliveryModeKey = req.Subscription.ReportDeliveryModeKey,
                ReportDeliveryFormatKey = req.Subscription.ReportDeliveryFormatKey,
                FileShareLocation = req.Subscription.FileShareLocation,
                EmailTo = req.Subscription.EmailTo,
                EmailCC = req.Subscription.EmailCC,
                EmailSubject = req.Subscription.EmailSubject,
                IsActive = true,
                CreatedAt = now,
                CreatedBy = userName
            });
        }
    }

    var toDelete = existing
        .Where(s => !seenSubKeys.Contains(s.ReportEventKey))
        .ToList();

    return (newList, updatedList, toDelete);
}

private async Task SaveChangesSafelyAsync(string context)
{
    try
    {
        await _context.SaveChangesAsync();
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error saving changes for {context}: {ex}");
        throw;
    }
}
