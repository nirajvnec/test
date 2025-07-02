{
  "reportKey": 24,
  "reportName": "edit_Sales",
  "reportDescription": "edit_Quarterly sales breakdown",
  "commentary": "edit_Reviewed by finance team",
  "workSpaceKey": 3,
  "approvedBy": "edit_john.doe@company.com",
  "reportStatus": "edit_Approved",
  "reportStatusName": "edit_Final Approval",
  "reportDatasets": [101, 102],
  "eventTypes": [
    {
      "eventTypeKey": 1,
      "nodes": [
        {
          "eventNodeId": "201",
          "eventNodeName": "edit_North Region"
        }
      ],
      "subscription": {
        "reportDeliveryModeKey": 1,
        "reportDeliveryFormatKey": 2,
        "fileShareLocation": "\\\\fileshare\\reports\\q2",
        "emailTo": "edit_sales@company.com",
        "emailCC": "edit_manager@company.com",
        "emailSubject": "edit_Q2 Sales Report"
      }
    },
    {
      "eventTypeKey": 2,
      "nodes": [
        {
          "eventNodeId": "202",
          "eventNodeName": "edit_South Region"
        }
      ],
      "subscription": {
        "reportDeliveryModeKey": 2,
        "reportDeliveryFormatKey": 1,
        "fileShareLocation": null,
        "emailTo": "edit_south.sales@company.com",
        "emailCC": null,
        "emailSubject": "edit_South Region Q2 Report"
      }
    }
  ]
}
