# Prompt 5: Integrate Frontend with Backend (End-to-End AWS)

Connect the React frontend to the FastAPI backend. Wire up WebSocket for live progress, auth for all routes, history display, and AWS account management.

## What to build

### API Integration

- When the user clicks "Run Analysis", send `POST /api/analyze` with AWS account ID, region, and JWT in the `Authorization` header:
  ```json
  {
    "account_id": "123456789012",
    "region": "us-east-1"
  }
  ```
- On the backend, validate the JWT on all protected endpoints (`/api/analyze`, `/api/history`, `/api/accounts`, `/api/regions`) using a FastAPI dependency.
- Return `analysis_id` immediately so frontend can connect to WebSocket.

### WebSocket Progress

- After triggering analysis, connect to `ws://localhost:8000/ws/progress/{analysis_id}` from React.
- Display each progress message in the `ProgressTracker` component as an animated step list with timestamps.
- Show resource counts: "Scanned 23 EC2 instances, 5 RDS databases, 8 S3 buckets..."
- Handle WebSocket disconnections gracefully with retry logic.

### AWS Account Management

- Store AWS credentials securely (never store access keys in localStorage).
- Implement basic account selection — users can specify different AWS account IDs.
- Show account-specific history and analyses.

### History + Reports

- History page fetches from `GET /api/history?page=1&limit=10` with JWT.
- Include pagination controls.
- Add date range filter, region filter, account filter.
- Clicking a past analysis opens the Report page with full details from the database.
- Show analysis metadata: duration, timestamp, resources scanned count.

### Report Display & Actions

- **Summary Card** at the top:
  - AWS Account ID, Region
  - Analysis timestamp and duration
  - Total resources scanned
  - Issues found (by severity)
  - Total estimated monthly savings (highlight savings amount)
  
- **Quick Wins Section** (top 3 actions):
  - Most impactful recommendations first
  - Estimated savings amount
  - AWS CLI copy button
  
- **Issues List**:
  - Filter by severity (High/Medium/Low)
  - Sort by estimated savings (highest first)
  - Each issue shows:
    - Resource ID and type (with icon)
    - Issue category (over-provisioned/unused/misconfigured/etc)
    - Severity badge (High=Red, Medium=Yellow, Low=Green)
    - Description and explanation
    - Estimated monthly savings
    - Fix command(s) in copyable code blocks
    - "Learn more" links to AWS documentation
  
- **Export Options**:
  - Download as PDF (summary + recommendations)
  - Download as JSON (full analysis data)
  - Share analysis link (optional)

### Error Handling

- Display user-friendly error messages if:
  - AWS credentials are invalid/expired
  - Region doesn't exist
  - Analysis times out
  - WebSocket connection fails
- Provide recovery steps (e.g., "Check your AWS credentials and try again")

### Performance & UX

- Debounce region/account selection to prevent accidental re-triggers
- Show estimated analysis time: "This will scan approximately 50-100 resources (~2-3 minutes)"
- Add a skeleton loader for history list while fetching
- Cache recent analyses for faster navigation
- Add "Refresh" button on report to re-run analysis on same account/region

### Final Testing Checklist

- [ ] Signup with new email
- [ ] Login with credentials
- [ ] View dashboard
- [ ] Select account and region
- [ ] Click "Run Analysis"
- [ ] Watch live progress updates via WebSocket
- [ ] View generated report with issues and recommendations
- [ ] Copy fix commands
- [ ] Check history page
- [ ] Filter history by account/region/date
- [ ] Open past analysis from history
- [ ] Export analysis as PDF/JSON
- [ ] Logout and login again
- [ ] Verify past analyses persist
- [ ] Test error scenarios (invalid account, network issues)

### Full Request Flow

```
User (Signup/Login)
  ↓ JWT stored in localStorage
User selects account & region, clicks "Run Analysis"
  ↓ POST /api/analyze {account_id, region} with JWT
Backend validates JWT
  ↓ Calls aws_scanner.py (Boto3 API calls)
  ↓ Emits WebSocket progress for each AWS service
  ↓ Calls ai_analyzer.py (OpenAI analysis)
  ↓ Stores results in Amazon RDS
  ↓ Returns analysis_id
Frontend connects to WebSocket ws:///.../ws/progress/{analysis_id}
  ↓ Receives live progress messages
  ↓ Displays animated progress tracker
Analysis completes
  ↓ Frontend fetches complete report from /api/analyses/{analysis_id}
  ↓ Displays report with issues, recommendations, fixes
User can:
  - Copy fix commands
  - Export as PDF/JSON
  - View past analyses in history
  - Run new analysis
```

Refer to `Architecture.MD`, `RequestFlow.MD`, and `AWS_MIGRATION_GUIDE.md`. This covers the full end-to-end flow — steps ① through ⑦.
