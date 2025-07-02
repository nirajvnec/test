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

    // Step 1: Process Events
    var dbEventEntities = await _context.PbiReportEventTypes
        .Where(e => e.ReportKey == reportKey)
        .ToListAsync();

    var incomingEventEntities = MapToEventEntities(reportKey, eventRequests, userName);
    await SyncEventTypesAsync(dbEventEntities, incomingEventEntities);

    // Step 2: Process Subscriptions
    var dbSubscriptions = await _context.PbiReportSubscription
        .Where(s => s.ReportKey == reportKey)
        .ToListAsync();

    var incomingSubscriptions = MapToSubscriptionEntities(reportKey, incomingEventEntities, eventRequests, userName);
    await SyncSubscriptionsAsync(dbSubscriptions, incomingSubscriptions);
}

private List<PbiReportEventTypes> MapToEventEntities(int reportKey, IEnumerable<PbiReportEventRequest> requests, string userName)
{
    return requests.Select(req => new PbiReportEventTypes
    {
        ReportEventKey = req.ReportEventKey, // 0 if new
        ReportKey = reportKey,
        EventTypeKey = req.EventTypeKey,
        EventNodeId = req.Nodes?.FirstOrDefault()?.EventNodeId,
        EventNodeName = req.Nodes?.FirstOrDefault()?.EventNodeName,
        CreatedAt = DateTime.UtcNow,
        CreatedBy = userName
    }).ToList();
}

private List<PbiReportSubscription> MapToSubscriptionEntities(
    int reportKey,
    List<PbiReportEventTypes> eventEntities,
    IEnumerable<PbiReportEventRequest> requests,
    string userName)
{
    var subscriptions = new List<PbiReportSubscription>();

    foreach (var req in requests)
    {
        if (req.Subscription != null)
        {
            var eventEntity = eventEntities.FirstOrDefault(e =>
                e.EventTypeKey == req.EventTypeKey &&
                e.EventNodeId == req.Nodes?.FirstOrDefault()?.EventNodeId);

            if (eventEntity == null) continue;

            subscriptions.Add(new PbiReportSubscription
            {
                ReportKey = reportKey,
                ReportEventKey = eventEntity.ReportEventKey,
                ReportDeliveryModeKey = req.Subscription.ReportDeliveryModeKey,
                ReportDeliveryFormatKey = req.Subscription.ReportDeliveryFormatKey,
                FileShareLocation = req.Subscription.FileShareLocation,
                EmailTo = req.Subscription.EmailTo,
                EmailCC = req.Subscription.EmailCC,
                EmailSubject = req.Subscription.EmailSubject,
                IsActive = true,
                CreatedAt = DateTime.UtcNow,
                CreatedBy = userName
            });
        }
    }

    return subscriptions;
}

private async Task SyncEventTypesAsync(List<PbiReportEventTypes> dbEvents, List<PbiReportEventTypes> incoming)
{
    var toAdd = incoming.Where(i => i.ReportEventKey == 0).ToList();

    var toUpdate = dbEvents.Where(db => incoming.Any(i => i.ReportEventKey == db.ReportEventKey &&
        (i.EventTypeKey != db.EventTypeKey ||
         i.EventNodeId != db.EventNodeId ||
         i.EventNodeName != db.EventNodeName))).ToList();

    var incomingKeys = incoming.Select(i => i.ReportEventKey).ToList();
    var toDelete = dbEvents.Where(db => !incomingKeys.Contains(db.ReportEventKey)).ToList();

    _context.PbiReportEventTypes.AddRange(toAdd);
    _context.PbiReportEventTypes.UpdateRange(toUpdate);
    _context.PbiReportEventTypes.RemoveRange(toDelete);

    try
    {
        await _context.SaveChangesAsync();
    }
    catch (Exception ex)
    {
        // Log or handle the exception as needed
        Console.WriteLine($"Error saving event types: {ex.Message}");
        throw;
    }
}

private async Task SyncSubscriptionsAsync(List<PbiReportSubscription> dbSubs, List<PbiReportSubscription> incoming)
{
    var toAdd = incoming.Where(i => !dbSubs.Any(d => d.ReportEventKey == i.ReportEventKey)).ToList();

    var toUpdate = dbSubs.Where(d => incoming.Any(i => i.ReportEventKey == d.ReportEventKey &&
        (i.EmailTo != d.EmailTo ||
         i.EmailCC != d.EmailCC ||
         i.EmailSubject != d.EmailSubject ||
         i.FileShareLocation != d.FileShareLocation))).ToList();

    var incomingKeys = incoming.Select(i => i.ReportEventKey).ToList();
    var toDelete = dbSubs.Where(d => !incomingKeys.Contains(d.ReportEventKey)).ToList();

    _context.PbiReportSubscription.AddRange(toAdd);
    _context.PbiReportSubscription.UpdateRange(toUpdate);
    _context.PbiReportSubscription.RemoveRange(toDelete);

    try
    {
        await _context.SaveChangesAsync();
    }
    catch (Exception ex)
    {
        // Log or handle the exception as needed
        Console.WriteLine($"Error saving subscriptions: {ex.Message}");
        throw;
    }
}

