const string userName = "system";
if (eventRequests == null) return;

// Remove old subscriptions
var oldEventKeys = _context.PbiReportEventTypes
    .Where(e => e.ReportKey == reportKey)
    .Select(e => e.ReportEventKey)
    .ToList(); // Materialize to avoid IQueryable vs IEnumerable ambiguity

var oldSubscriptions = _context.PbiReportSubscription
    .Where(s => oldEventKeys.Contains(s.ReportEventKey))
    .ToList(); // Optionally materialize if you plan to remove them

_context.PbiReportSubscription.RemoveRange(oldSubscriptions);
