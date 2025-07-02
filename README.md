public async Task<int> SaveAsync(PbiReportRequestV2 request)
{
    using var transaction = await _context.Database.BeginTransactionAsync();
    try
    {
        var report = await UpsertReportAsync(request);
        await AddReportDatasetsAsync(report.ReportKey, request.ReportDatasets);
        await AddReportEventsAndSubscriptionsAsync(report.ReportKey, request.EventTypes);

        await transaction.CommitAsync();
        return report.ReportKey;
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}

private async Task<PbiReport> UpsertReportAsync(PbiReportRequestV2 request)
{
    const string userName = "system";
    PbiReport report;

    if (request.ReportKey.HasValue && request.ReportKey.Value > 0)
    {
        report = await _context.PbiReports.FindAsync(request.ReportKey.Value)
                 ?? throw new Exception("Report not found.");

        report.ReportName = request.ReportName;
        report.ReportDescription = request.ReportDescription;
        report.ModifiedAt = DateTime.UtcNow;
        report.ModifiedBy = userName;
        report.Commentary = request.Commentary;
        report.WorkSpaceKey = request.WorkSpaceKey;
        report.ApprovedBy = request.ApprovedBy;
        report.ReportStatus = request.ReportStatus;
        report.ReportStatusName = request.ReportStatusName;
    }
    else
    {
        report = new PbiReport
        {
            ReportName = request.ReportName,
            ReportDescription = request.ReportDescription,
            CreatedAt = DateTime.UtcNow,
            UploadedBy = userName,
            Commentary = request.Commentary,
            WorkSpaceKey = request.WorkSpaceKey,
            ApprovedBy = request.ApprovedBy,
            ReportStatus = request.ReportStatus,
            ReportStatusName = request.ReportStatusName
        };

        _context.PbiReports.Add(report);
    }

    await _context.SaveChangesAsync();
    return report;
}

private async Task AddReportDatasetsAsync(int reportKey, int[]? datasetKeys)
{
    if (datasetKeys == null || datasetKeys.Length == 0)
        return;

    // Remove existing datasets
    var existing = _context.PbiReportDatasets.Where(d => d.ReportKey == reportKey);
    _context.PbiReportDatasets.RemoveRange(existing);

    // Add new ones
    var datasets = datasetKeys.Select(key => new PbiReportDatasets
    {
        ReportKey = reportKey,
        DatasetKey = key
    });

    _context.PbiReportDatasets.AddRange(datasets);
    await _context.SaveChangesAsync();
}

private async Task AddReportEventsAndSubscriptionsAsync(int reportKey, IEnumerable<PbiReportEventRequest>? eventRequests)
{
    const string userName = "system";
    if (eventRequests == null) return;

    // Remove old subscriptions
    var oldEventKeys = _context.PbiReportEventTypes
        .Where(e => e.ReportKey == reportKey)
        .Select(e => e.ReportEventKey);

    var oldSubscriptions = _context.PbiReportSubscriptions
        .Where(s => oldEventKeys.Contains(s.ReportEventKey));
    _context.PbiReportSubscriptions.RemoveRange(oldSubscriptions);

    // Remove old events
    var oldEvents = _context.PbiReportEventTypes
        .Where(e => e.ReportKey == reportKey);
    _context.PbiReportEventTypes.RemoveRange(oldEvents);

    await _context.SaveChangesAsync();

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
        await _context.SaveChangesAsync();

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

            _context.PbiReportSubscriptions.Add(subscription);
            await _context.SaveChangesAsync();
        }
    }
}
