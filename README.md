private async Task AddReportEventsAndSubscriptionsAsync(
    int reportKey,
    List<PbiReportEventRequestV2> eventRequests)
{
    const string userName = "system";
    if (eventRequests == null || eventRequests.Count == 0)
        return;

    // Remove old event types for this report
    var oldEventKeys = await _context.PbiReportEventTypes
        .Where(e => e.ReportKey == reportKey)
        .Select(e => e.ReportEventKey)
        .ToListAsync();

    // Remove old subscriptions related to the old event keys
    var oldSubscriptions = await _context.PbiReportSubscription
        .Where(s => oldEventKeys.Contains(s.ReportEventKey))
        .ToListAsync();

    _context.PbiReportSubscription.RemoveRange(oldSubscriptions);
    _context.PbiReportEventTypes.RemoveRange(
        _context.PbiReportEventTypes.Where(e => e.ReportKey == reportKey));

    // Add new event types and subscriptions
    foreach (var request in eventRequests)
    {
        var eventType = new PbiReportEventTypes
        {
            ReportKey = reportKey,
            EventTypeKey = request.EventTypeKey,
            CreatedBy = userName,
            CreatedAt = DateTime.UtcNow
        };

        _context.PbiReportEventTypes.Add(eventType);
        await _context.SaveChangesAsync(); // Save to generate ReportEventKey

        // Add subscription if present in request
        if (request.Subscription != null)
        {
            var subscription = new PbiReportSubscription
            {
                ReportKey = reportKey,
                ReportEventKey = eventType.ReportEventKey,
                ReportDeliveryModeKey = request.Subscription.ReportDeliveryModeKey,
                ReportDeliveryFormatKey = request.Subscription.ReportDeliveryFormatKey,
                FileShareLocation = request.Subscription.FileShareLocation,
                EmailTo = request.Subscription.EmailTo,
                EmailCc = request.Subscription.EmailCc
            };

            _context.PbiReportSubscription.Add(subscription);
            await _context.SaveChangesAsync();
        }
    }
}
