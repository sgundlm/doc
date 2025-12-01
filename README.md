# AWS Cross-Cloud Interconnect (CCI) Runbook

This runbook guides the Operations and SRE teams in handling incidents originating from the AWS side of the Cross-Cloud Interconnect (CCI). It covers incident detection, alerting, and step-by-step triage procedures.

## 1. Incident Detection & Alerting

We utilize AWS CloudWatch and EventBridge to detect connectivity issues and performance degradation.

### 1.1. CloudWatch Alarms

The following CloudWatch alarms should be configured for the Direct Connect Virtual Interface (VIF) or VPN Connection.

| Metric | Namespace | Statistic | Threshold | Description |
| :--- | :--- | :--- | :--- | :--- |
| **ConnectionState** | AWS/DX | Minimum | `< 1` (for 1 min) | Triggers when the Direct Connect connection goes down (0 = down, 1 = up). |
| **VirtualInterfaceState** | AWS/DX | Minimum | `< 1` (for 1 min) | Triggers when the VIF goes down. |
| **ConnectionBps** | AWS/DX | Average | `< 1000` (Low) | Triggers if traffic drops to near zero, indicating potential routing or application issues. |
| **PacketDropCount** | AWS/DX | Sum | `> 100` (for 5 min) | High packet drops indicating congestion or physical layer issues. |
| **TunnelState** | AWS/VPN | Minimum | `< 1` (for 1 min) | (If VPN used) Triggers when the VPN tunnel is down. |

### 1.2. EventBridge Rules

Configure EventBridge rules to capture state changes and notify via SNS/PagerDuty.

*   **Rule Name**: `Capture-DX-State-Change`
*   **Event Pattern**:
    ```json
    {
      "source": ["aws.directconnect"],
      "detail-type": ["Direct Connect Connection State Change", "Direct Connect Virtual Interface State Change"]
    }
    ```
*   **Target**: SNS Topic (e.g., `SRE-Alerts`) -> PagerDuty/Slack.

## 2. Triage Procedure

Follow these steps when an alert is triggered or connectivity issues are reported.

### Step 1: Validate Connection Status

**Objective**: Determine if the physical connection or logical interface is down.

1.  **Check AWS Console**:
    *   Navigate to **Direct Connect** > **Connections**.
    *   Verify the status is `available`.
    *   Navigate to **Virtual Interfaces**.
    *   Verify the status is `available`.
2.  **Check VPN (if applicable)**:
    *   Navigate to **VPC** > **Site-to-Site VPN Connections**.
    *   Check `Tunnel Details`. Both tunnels should be `UP`.
3.  **CLI Command**:
    ```bash
    aws directconnect describe-connections --connection-id <dx-con-id>
    aws directconnect describe-virtual-interfaces --connection-id <dx-con-id>
    ```

**Action**:
*   If **Connection is DOWN**: Check for AWS maintenance notifications (PHD). Contact AWS Support if no maintenance is scheduled.
*   If **VIF is DOWN**: Check BGP session status. Ensure the GCP side is advertising routes.

### Step 2: Review CloudWatch Metrics

**Objective**: Identify performance degradation (packet loss, latency, throughput).

1.  Navigate to **CloudWatch** > **Metrics** > **All metrics** > **DX**.
2.  Select the relevant Connection or VIF.
3.  Analyze the following metrics over the last 1 hour:
    *   **ConnectionBps** (Ingress/Egress): Is traffic flowing? Is it saturated (hitting bandwidth limits)?
    *   **ConnectionPps**: Packet rate. Sudden spikes?
    *   **PacketDropCount**: Are we dropping packets?
        *   *Ingress Drops*: Usually means AWS can't process fast enough or policing.
        *   *Egress Drops*: Usually means the receiving end (GCP) or the link is congested.

### Step 3: Validate GCP Side (Cross-Cloud)

Since this is a CCI, the issue might be on the Google Cloud side.

1.  Check GCP **Cloud Interconnect** status in the Google Cloud Console.
2.  Verify **BGP Session** status on the Cloud Router.
3.  Check for **Dropped Packets** metrics in GCP Cloud Monitoring.

### Step 4: Escalation & Support

If the issue persists and cannot be resolved, initiate the formal escalation process.

#### 4.1. Escalation Path (L1 - L3)

| Level | Role | Responsibilities | SLA (Response/Resolution) |
| :--- | :--- | :--- | :--- |
| **L1** | **Cloud Operations / NOC** | Initial triage, validate alerts, check dashboards. Basic troubleshooting (Step 1-3). | 15 min / 1 hour |
| **L2** | **SRE / DevOps Team** | Deep dive analysis, log review, configuration validation. Engage AWS/GCP support if needed. | 30 min / 4 hours |
| **L3** | **Network Architecture / Principal Engineers** | Complex routing issues, architectural flaws, critical outages affecting multiple services. | 1 hour / 8 hours |

#### 4.2. Critical Incident Response Contacts

For **Severity 1 (Critical)** incidents (Complete outage of CCI):

*   **Incident Commander**: On-Call Manager (`+1-555-0100`)
*   **Bridge Line**: `https://meet.google.com/abc-defg-hij` (or internal Zoom link)
*   **Slack Channel**: `#incident-war-room`

#### 4.3. Key Stakeholders & Support Teams

| Team | Distribution List / Email | Assignment Group (ServiceNow/Jira) | Key Contacts |
| :--- | :--- | :--- | :--- |
| **AWS Operations** | `aws-ops@example.com` | `AWS_Cloud_Triage` | Jane Doe (Lead) |
| **AMS (Managed Services)** | `ams-support@example.com` | `AMS_Escalation` | AMS Account Manager |
| **Cloud Operations** | `cloud-ops@example.com` | `General_Ops_Support` | John Smith |
| **Network Engineering** | `net-eng@example.com` | `Network_Engineering` | Sarah Jones (Architect) |

#### 4.4. AMS Escalation Process

If the issue lies within the AWS Managed Services (AMS) scope:

1.  **Log Incident**: Create a case in AMS Console (Severity: Production System Down).
2.  **Escalate**: If no response within SLA, contact the **AMS Incident Manager**.
3.  **Playbook**: Refer to the [AWS Incident Manager Playbook](https://console.aws.amazon.com/systems-manager/incidents/home) (Replace with actual link).

## 3. Resources & Links

*   **Monitoring Dashboards**:
    *   [AWS CloudWatch Dashboard](https://console.aws.amazon.com/cloudwatch/home#dashboards:) (Link to specific CCI Dashboard)
    *   [GCP Monitoring Dashboard](https://console.cloud.google.com/monitoring/dashboards)
*   **Automation Scripts**:
    *   [Connectivity Test Script](https://github.com/org/repo/blob/main/scripts/test_connectivity.sh)
    *   [Log Collection Script](https://github.com/org/repo/blob/main/scripts/collect_logs.sh)
*   **Reference Documentation**:
    *   [AMS Monitoring Guide](https://docs.aws.amazon.com/managedservices/latest/userguide/monitoring.html)

## 4. ITIL Alignment & Governance

This runbook aligns with the **ITIL Incident Management** process:

*   **Detection**: Automated via CloudWatch/EventBridge.
*   **Classification**: Categorized as `Network/Connectivity`.
*   **Diagnosis**: Triage steps 1-3.
*   **Resolution**: Restoration of service or workaround.
*   **Closure**: Post-Incident Review (PIR) and Root Cause Analysis (RCA).

## 5. Useful Commands

*   **Traceroute (TCP)**: `traceroute -T -p <port> <dest-ip>`
*   **MTR**: `mtr -r -c 100 <dest-ip>`