{
  "reportKey": null,
  "reportName": "Sales",
  "reportDescription": "Quarterly sales breakdown",
  "commentary": "Reviewed by finance team",
  "workSpaceKey": 3,
  "approvedBy": "john.doe@company.com",
  "reportStatus": "Approved",
  "reportStatusName": "Final Approval",
  "reportDatasets": [101, 102],
  "eventTypes": [
    {
      "eventTypeKey": 1,
      "nodes": [
        {
          "eventNodeId": "201",
          "eventNodeName": "North Region"
        }
      ],
      "subscription": {
        "reportDeliveryModeKey": 1,
        "reportDeliveryFormatKey": 2,
        "fileShareLocation": "\\\\fileshare\\reports\\q2",
        "emailTo": "sales@company.com",
        "emailCC": "manager@company.com",
        "emailSubject": "Q2 Sales Report"
      }
    },
    {
      "eventTypeKey": 2,
      "nodes": [
        {
          "eventNodeId": "202",
          "eventNodeName": "South Region"
        }
      ],
      "subscription": {
        "reportDeliveryModeKey": 2,
        "reportDeliveryFormatKey": 1,
        "fileShareLocation": null,
        "emailTo": "south.sales@company.com",
        "emailCC": null,
        "emailSubject": "South Region Q2 Report"
      }
    }
  ]
}
