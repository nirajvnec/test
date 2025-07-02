private async Task AddReportEventsAndSubscriptionsAsync(int reportKey, IEnumerable<PbiReportEventRequest> eventRequests)
{
    const string userName = "system";

    if (eventRequests == null || !eventRequests.Any())
        return;

    // ✔ Materialize old event keys to avoid open DataReader issue
    var oldEventKeys = _context.PbiReportEventTypes
        .Where(e => e.ReportKey == reportKey)
        .Select(e => e.ReportEventKey)
        .ToList(); // ✔

    var oldSubscriptions = _context.PbiReportSubscription
        .Where(s => oldEventKeys.Contains(s.ReportEventKey))
        .ToList(); // ✔
    
    _context.PbiReportSubscription.RemoveRange(oldSubscriptions);

    var oldEvents = _context.PbiReportEventTypes
        .Where(e => e.ReportKey == reportKey)
        .ToList(); // ✔

    _context.PbiReportEventTypes.RemoveRange(oldEvents);

    try
    {
        await _context.SaveChangesAsync(); // ✔ Safe now
    }
    catch (Exception e)
    {
        Console.WriteLine(e);
        throw;
    }

    // Add new events and subscriptions
    foreach (var eventReq in eventRequests)
    {
        var eventEntity = new PbiReportEventTypes
        {
            ReportKey = reportKey,
            EventTypeKey = eventReq.EventTypeKey,
            NodeId = eventReq.Nodes?.FirstOrDefault()?.EventNodeId,
            EventNodeName = eventReq.Nodes?.FirstOrDefault()?.EventNodeName,
            CreatedAt = DateTime.UtcNow,
            CreatedBy = userName
        };

        _context.PbiReportEventTypes.Add(eventEntity);

        try
        {
            await _context.SaveChangesAsync(); // ✔ Save each eventEntity to get ReportEventKey for FK
        }
        catch (Exception e)
        {
            Console.WriteLine(e);
            throw;
        }

        if (eventReq.Subscription != null)
        {
            var subscription = new PbiReportSubscription
            {
                ReportKey = reportKey,
                ReportEventKey = eventEntity.ReportEventKey,
                ReportDeliveryModeKey = eventReq.Subscription.ReportDeliveryModeKey,
                ReportDeliveryFormatKey = eventReq.Subscription.ReportDeliveryFormatKey,
                FileShareLocation = eventReq.Subscription.FileShareLocation,
                EmailTo = eventReq.Subscription.EmailTo,
                EmailCC = eventReq.Subscription.EmailCC,
                EmailSubject = eventReq.Subscription.EmailSubject,
                IsActive = true,
                CreatedAt = DateTime.UtcNow,
                CreatedBy = userName
            };

            _context.PbiReportSubscription.Add(subscription);
            await _context.SaveChangesAsync(); // ✔ Save each subscription
        }
    }
}
