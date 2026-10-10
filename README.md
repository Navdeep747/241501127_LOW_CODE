# Employee Leave Management (Zoho Creator)

## 1. Project Description
A Zoho Creator app that lets employees apply for leave online, lets managers approve or reject requests, and tracks leave balances automatically. An AI field helps validate and summarize requests so approvals are faster and more consistent.

## 2. Problem Statements
- Leave requests are handled by email, paper or chat, so they get lost or delayed.
- Leave balances are tracked manually in spreadsheets, causing errors.
- Managers cannot easily see team availability or overlapping leaves.
- No audit trail of who approved what and when.
- Employees get no instant status updates.
- Reasons for leave are free text and slow for managers to review.

## 3. Forms

### 3.1 Employee (master)
| Field | Type | Notes |
|---|---|---|
| Employee_ID | Auto Number | Unique |
| Name | Name | Required |
| Email | Email | Unique |
| Department | Dropdown | |
| Manager | Lookup (Employee) | |
| Joining_Date | Date | |

### 3.2 Leave_Type (master)
| Field | Type | Notes |
|---|---|---|
| Type_Name | Single Line | Casual, Sick, Earned, etc. |
| Annual_Quota | Number | Days per year |

### 3.3 Leave_Balance
| Field | Type | Notes |
|---|---|---|
| Employee | Lookup (Employee) | |
| Leave_Type | Lookup (Leave_Type) | |
| Total_Days | Decimal | |
| Used_Days | Decimal | |
| Available_Days | Formula | Total_Days - Used_Days |

### 3.4 Leave_Request
Basic fields and AI fields are listed in section 4.

## 4. Fields

### 4.1 Basic Fields (Leave_Request)
| Field | Type | Notes |
|---|---|---|
| Employee | Lookup (Employee) | Required |
| Leave_Type | Lookup (Leave_Type) | Required |
| From_Date | Date | Required |
| To_Date | Date | Required, >= From_Date |
| Total_Days | Decimal | Auto-calculated |
| Reason | Multi Line | Required |
| Status | Dropdown | Pending / Approved / Rejected / Cancelled |
| Manager_Comments | Multi Line | Filled on decision |
| Applied_On | Date-Time | Auto (`zoho.currenttime`) |

### 4.2 AI Fields
| Field | Type | Purpose |
|---|---|---|
| AI_Reason_Summary | AI field (Text) | One-line summary of the reason for the manager |
| AI_Category | AI field (Classification) | Classifies reason: Medical, Personal, Family, Travel, Other |
| AI_Sentiment_Urgency | AI field (Classification) | Flags Urgent / Normal |
| AI_Recommendation | AI field (Text) | Suggests approve/reject based on balance, overlaps and reason (advisory only; manager decides) |

AI fields use Reason, Total_Days and balance data as input. They are advisory and never auto-approve.

## 5. Deluge Scripts

### 5.1 Calculate Total Days (On User Input of To_Date)
```deluge
if(input.From_Date != null && input.To_Date != null)
{
    input.Total_Days = input.From_Date.daysBetween(input.To_Date) + 1;
}
```

### 5.2 Validate Request (On Validate)
```deluge
if(input.To_Date < input.From_Date)
{
    alert "To Date cannot be before From Date";
    cancel submit;
}
bal = Leave_Balance[Employee == input.Employee && Leave_Type == input.Leave_Type];
if(bal.Available_Days < input.Total_Days)
{
    alert "Insufficient leave balance";
    cancel submit;
}
overlap = Leave_Request[Employee == input.Employee && Status != "Rejected" && Status != "Cancelled"
          && From_Date <= input.To_Date && To_Date >= input.From_Date];
if(overlap.count() > 0)
{
    alert "You already have leave in this period";
    cancel submit;
}
```

### 5.3 Notify Manager (On Success of Leave_Request)
```deluge
mgr = Employee[ID == input.Employee.Manager];
sendmail
[
    from: zoho.adminuserid
    to: mgr.Email
    subject: "Leave request from " + input.Employee.Name
    message: "Dates: " + input.From_Date + " to " + input.To_Date +
             "<br>Days: " + input.Total_Days +
             "<br>Summary: " + input.AI_Reason_Summary
]
```

### 5.4 Approve / Reject (Custom Button or Workflow on Edit)
```deluge
if(input.Status == "Approved")
{
    bal = Leave_Balance[Employee == input.Employee && Leave_Type == input.Leave_Type];
    bal.Used_Days = bal.Used_Days + input.Total_Days;
}
sendmail
[
    from: zoho.adminuserid
    to: input.Employee.Email
    subject: "Leave " + input.Status
    message: "Your leave from " + input.From_Date + " to " + input.To_Date +
             " is " + input.Status + ".<br>Comments: " + input.Manager_Comments
]
```

### 5.5 Restore Balance on Cancellation
```deluge
if(input.Status == "Cancelled" && old.Status == "Approved")
{
    bal = Leave_Balance[Employee == input.Employee && Leave_Type == input.Leave_Type];
    bal.Used_Days = bal.Used_Days - input.Total_Days;
}
```

### 5.6 Yearly Balance Reset (Schedule: Jan 1)
```deluge
for each emp in Employee
{
    for each lt in Leave_Type
    {
        rec = Leave_Balance[Employee == emp.ID && Leave_Type == lt.ID];
        if(rec.count() == 0)
        {
            insert into Leave_Balance
            [
                Employee = emp.ID
                Leave_Type = lt.ID
                Total_Days = lt.Annual_Quota
                Used_Days = 0
            ];
        }
        else
        {
            rec.Total_Days = lt.Annual_Quota;
            rec.Used_Days = 0;
        }
    }
}
```

## 6. Workflows
| # | Workflow | Form | Trigger | Action |
|---|---|---|---|---|
| 1 | Auto-calculate days | Leave_Request | On user input (From/To date) | Script 5.1 |
| 2 | Validate request | Leave_Request | On validate | Script 5.2 |
| 3 | Set default status | Leave_Request | On create | Status = Pending |
| 4 | Notify manager | Leave_Request | On create success | Script 5.3 |
| 5 | Decision handling | Leave_Request | On edit (Status changed) | Script 5.4 |
| 6 | Cancellation | Leave_Request | On edit (Status = Cancelled) | Script 5.5 |
| 7 | Yearly reset | Leave_Balance | Schedule (yearly, Jan 1) | Script 5.6 |
| 8 | Pending reminder | Leave_Request | Schedule (daily) | Email manager for requests Pending > 2 days |

## 7. Flow Summary
Employee applies, then the system validates, AI fields summarize and classify, the manager is notified, the manager approves or rejects, the balance updates and the employee is notified.

## 8. Setup Steps
1. Create the Employee, Leave_Type, Leave_Balance and Leave_Request forms.
2. Add the basic fields, then the AI fields on Leave_Request.
3. Add the Deluge scripts through the workflows listed above.
4. Create pages and reports: My Leaves, Pending Approvals, Team Calendar, Balance Report.
5. Set permissions: Employee (own records), Manager (team), HR/Admin (all).
