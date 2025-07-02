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
