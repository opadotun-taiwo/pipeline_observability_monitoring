# pipeline_observability_monitoring
Data Observability & Monitoring Pipeline (Airflow + ClickHouse + Teams)
Overview

This project implements a data observability and alerting system using Apache Airflow. It extracts daily e-commerce metrics from ClickHouse, delivers summarized reports to Microsoft Teams, and provides real-time failure notifications for pipeline monitoring.

The system is designed to eliminate silent failures and improve operational visibility for both technical and business stakeholders.

🏗️ System Architecture
                +---------------------+
                |   ClickHouse DB     |
                +----------+----------+
                           |
                           v
                +---------------------+
                |   Airflow DAG       |
                |  (PythonOperator)   |
                +----------+----------+
                           |
          +----------------+----------------+
          |                                 |
          v                                 v
+---------------------+        +--------------------------+
| extract_summary()   |        | task_fail_alert()        |
| (Data Aggregation)  |        | (Failure Callback)       |
+----------+----------+        +------------+-------------+
           |                                |
           v                                v
+---------------------+        +--------------------------+
| send_to_teams()     |------->| Microsoft Teams Webhook  |
| (Success Report)    |        | (Alerts & Metrics)       |
+---------------------+        +--------------------------+
⚙️ Tech Stack
Layer	Technology
Orchestration	Apache Airflow
Language	Python
Database	ClickHouse
Data Processing	Pandas
Alerting	Microsoft Teams Webhook
Config Management	Airflow Variables
📅 DAG Configuration
dag_id='ecom_order_summary'
schedule_interval='0 9 * * *'  # Daily at 09:00
catchup=False
Task Flow
extract_data >> send_to_teams
Default Args
default_args = {
    'owner': 'Opadotun Taiwo',
    'depends_on_past': False,
    'on_failure_callback': task_fail_alert
}
🔍 Data Extraction Logic
Function: extract_summary(config: dict)
Responsibilities:

Establish connection to ClickHouse

Execute aggregation query for previous day

Return structured output via XCom

Query
SELECT
    count(MainOrderId) AS Total_orders,
    sum(CASE WHEN status = 'order_delivered' THEN 1 ELSE 0 END) AS Total_fulfilled_orders,
    sum(CASE WHEN status = 'order_cancelled' THEN 1 ELSE 0 END) AS Total_cancelled_orders,
    sum(final_amount) AS Total_order_amount,
    count(DISTINCT customerId) AS Total_Customers
FROM order_line
WHERE toDate(orderDate) = toDate('{yesterday}')
Output Schema
Column	Type
total_orders	int
fulfilled_orders	int
cancelled_orders	int
total_order_amount	float
total_customers	int
📤 Teams Notification Logic
Function: send_to_teams(**context)
Responsibilities:

Pull extracted data via XCom

Format response payload

Send HTTP POST request to Teams webhook

Key Implementation Details

Uses requests.post() with JSON payload

Formats currency using Nigerian Naira (₦)

Converts Pandas DataFrame to structured message card

Payload Format (Simplified)
{
  "title": "Ecommerce Order Summary – YYYY-MM-DD",
  "sections": [
    {
      "facts": [
        {"name": "Total Orders", "value": 100},
        {"name": "Total Revenue", "value": "₦1,000,000"}
      ]
    }
  ]
}
🚨 Failure Alerting System
Function: task_fail_alert(context)
Trigger:

Automatically invoked via Airflow on_failure_callback

Responsibilities:

Capture task failure metadata

Send alert to Teams with debugging context

Includes:

Task ID

DAG ID

Execution date

Direct log URL

Example Payload
{
  "title": "Airflow Task Failed",
  "sections": [
    {
      "activityTitle": "Task failed",
      "facts": [
        {"name": "Logical Date", "value": "2026-01-01"},
        {"name": "Log URL", "value": "http://airflow/log"}
      ]
    }
  ]
}
🔐 Configuration

All sensitive credentials are managed via Airflow Variables:

Variable Name	Description
clickhouse_host	ClickHouse host
ecomm_port	ClickHouse port
ecomm_share_username	DB username
ecomm_password	DB password
ecomm_share_db	Database name
teams_webhook_secret	Teams webhook URL
🧪 Execution Flow

DAG triggers at scheduled time

extract_data:

Connects to ClickHouse

Executes aggregation query

Pushes result to XCom

send_to_teams:

Pulls XCom data

Sends formatted summary to Teams

If any task fails:

task_fail_alert sends failure notification

⚠️ Error Handling
ClickHouse Connection
except Exception as e:
    raise AirflowException(...)
Teams Delivery
if response.status_code != 200:
    print("Message failed to deliver")
DAG-Level Monitoring

Centralized failure handling via callback

Ensures all failures trigger alerts

📦 Installation
pip install clickhouse-connect pandas requests apache-airflow
🚀 Deployment

Place DAG file in Airflow dags/ directory

Place utility scripts in utils/ directory

Configure Airflow Variables

Enable DAG in Airflow UI

📈 Extensibility

Add additional metrics to SQL query

Extend alerting to Slack/Email

Integrate data quality checks

Add retry logic and SLA monitoring

Persist historical summaries to warehouse
