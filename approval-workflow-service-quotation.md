# Approval Workflow for Service Quotation

*Published: April 23, 2023 | SAP Community*

**SAP Managed Tags:** SAP S/4HANA

---

One of the key features of workflow is to enable an organization to automate and streamline their processes, however, sometimes it's too rigid and does not meet specific business needs. In this blog, we will explore how to create a flexible workflow for Service Quotation that can adapt to changing business requirements.

---

## Introduction

The flexible workflow is a general function that allows you to define condition-based workflows for approval processes. The functionality is available with CE2302. The flexible workflow for a Service Quotation allows you to define Single Stage Approval Processes for Service Quotation according to your requirements. Approvers can then approve or reject their work items in the app **My Inbox**.

The functionality supported by the approval process includes:

- Sending quote for Internal Approval before it's released to customer
- Approval Withdrawal
- Sending of quote for Internal Approval based on preconfigured condition (e.g., based on service margin below or above a certain percentage value)
- Email Notification
- Search and Filter the quotation in the Manage Service Quotation Application based on Approval Statuses

---

## Prerequisites

| Item | Value |
|------|-------|
| Workflow Scenario ID | `WS04300018` |
| Status Profile | `SERVQTWF` |
| Responsibility Rule | `SRV_QTAN_AGENT` |

Before setting up the workflow for the first time, complete the following steps:

1. **Activate the workflow `WS04300018`** (tcode `SWF_PROCESS_ADMIN`)
2. **Maintain Status Profile `SERVQTWF`** to one of the Service Quotation Transaction Types
3. **Optimize user status for queries** — the following three statuses need to be mapped to the correct user status:
   - `Q071`
   - `Q072`
   - `Q073`
4. **Ensure Edit runtime environment is green** (tcode: `SWU3`)

Once the above steps are completed, the status dropdown will display the additional Approval Workflow statuses:

- Sent for Approval
- Approval Released
- Approval Rejected

The button **Send for Approval** is enabled if the status of the quotation is `Open` or `Under Review`, and the new assignment block **Approval Details** becomes visible.

---

## Agent Determination

### Create the Custom Responsibility Rule ID

1. Navigate to the **Custom Logic** app and click on **Create**
2. Choose **Responsibility Rule** as the Business Context and click on **Step 2**
3. Maintain the BAdI filters by clicking **Add** and adding the necessary filters
4. Set the BAdI Filter Value to `YY1_AGENT_DET_QTAN`. Click **Save** and proceed to **Step 3**
5. Keep the implementation ID and the BAdI Filter Value name the same. Note that when you enter the description, the implementation ID will be determined based on the description provided. You can overwrite this ID to match the BAdI filter
6. Click on **Review** and then **Create**. The status will show as **Creating**
7. After some time, the **Publish** button will be enabled. Click on **Publish** — status changes to **Publishing** and later to **Published**

### Using the Responsibility Rule BAdI

With the **Manage Responsibility Rules** app, customers can create custom/copy standard responsibility rules delivered by SAP to determine agents responsible for workflows and business scenarios.

1. Log on to SAP Fiori Launchpad as an administrator (`SAP_BR_ADMINISTRATOR` role)
2. Click on tile **Manage Responsibility Rules** under group **Extensibility for Responsibility Management**
3. Go to the **Custom** tab and click on **Create**
4. Enter a rule ID and description (Prefix is populated automatically: For Cloud, `YY1_` prefix is populated). BAdI filter gets populated automatically based on the ID assigned
5. Click on the **Create** button for Parameters (optional — for this scenario, leave blank)
6. Save the Rule

### Using the Manage Responsibility Contexts Application

1. Log on to SAP Fiori Launchpad as an administrator (`SAP_BR_ADMINISTRATOR` role)
2. Open app **Manage Responsibility Contexts**
3. You will see standard contexts delivered by SAP and extensions under separate tabs. Context type `SITU` is for Situations and `SWF` is for Workflow
4. Select standard responsibility context ID `WS04300018` and choose **Extend**

   > **Note:** The Extend button is only available in an extensibility dev system/client. It is not available if a standard responsibility context has already been extended — a standard context can be extended only once. To make changes to an existing extension, edit the context extension.

5. From the **Agent Rule** tab, choose **Create**
6. Select the value help for **Data source mapping** and map the correct CDS field/value from the value help
7. Choose **Apply** and **Save** — the Agent rule is added and Responsibility Context Extension is saved successfully

---

## Custom Fields and Logic App

In this example, we determine the Agent Responsible for processing the Approval Workflow related to Service Quotation assigned to Sales Org `1710`.

1. Go to App **Custom Logic**
2. Click on **Create** (top right corner) and choose Business Context as **Service Header**

   New preconditions can be defined using custom BAdI `SWF_WORKFLOW_CONDITION_DEF`.

3. Assign the Scenario ID: `WS04300018`
4. Provide the implementation description (implementation ID gets populated automatically)
5. Click on **Review** and **Create**
6. Open the code editor and write the logic to define the new precondition. Example — defining the condition Sales Org with parameter value equals `1710`:

```abap
* condition id - value to be changed
data : ls_condition like line of ct_condition.
data : ls_parameter like line of ct_parameter.

ls_condition-id = 1.
ls_condition-subject = 'Sales Org'.
ls_condition-type    = if_swf_flex_ifs_condition_def=>cs_condtype-start_step.
append ls_condition to ct_condition.

ls_parameter-id = 1.
ls_parameter-name = 'Sales Org'.
ls_parameter-xsd_type = if_swf_flex_ifs_condition_def=>cs_xstype-string.
ls_parameter-mandatory = abap_false.
append ls_parameter to ct_parameter.
```

7. Define the step condition to be evaluated using BAdI `SWF_WORKFLOW_CONDITION_EVAL`. Repeat the same steps to create the BAdI implementation, but this time in BAdI description select **"Value evaluation of addition conditions for scenarios"**

   In this BAdI implementation, read the value of the condition defined (Sales Org = 1710) and check if the Service Quotation Sales Org ID matches. If true, pass `CV_IS_TRUE = 'X'`:

```abap
DATA: lv_org TYPE VKORG.

SELECT SINGLE * INTO @DATA(lS_service)
  FROM I_ServiceDocumentEnhcd
  WHERE ServiceDocument = @is_sap_object_node_type-sont_key_part_1.

IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE cx_ble_runtime_error
    EXPORTING
      previous = NEW cx_swf_flex_lra(
        textid = cx_swf_flex_lra=>object_instance_not_found ).
ENDIF.

READ TABLE it_parameter_value INTO DATA(ls_param_value)
  WITH KEY name = 'Sales Org'.

IF sy-subrc = 0.
  lv_org = ls_param_value-value.
ENDIF.

IF lS_service-SALESORGANIZATION = ls_param_value-value.
  cv_is_true = abap_true.
ELSE.
  cv_is_true = abap_false.
ENDIF.
```

With all the steps above, you will be able to activate the workflow for Service Quotation with Sales Org = `1710`.

---

## Flexible Workflow — Email Templates

| Use Case | Template | Description |
|----------|----------|-------------|
| Workflow End | `CRMS4_QTAN_APPR_EMAIL_TMPL` | At the end of the workflow, the initiator is informed about the Approval |
| Workflow End | `CRMS4_QTAN_RJCT_EMAIL_TMPL` | At the end of the workflow, the initiator is informed about the Rejection |
| Step Deadline | `CRMS4_QTAN_DLINE_EMAIL_TMPL` | When a deadline is reached, the step is marked as Overdue |

Copy the template in App: **Maintain Email Templates**.

---

## Manage Workflow App for Service Quotation

By default, when the system is set up for the first time, you only have **Predelivered Workflow Content**. The predelivered content is triggered when none of the process definitions' start conditions apply. For example, if the scenario is set up to trigger the workflow when Sales Org is `1710`, but the quotation does not belong to Sales Org `1710`, the default predelivered workflow is triggered which will initiate auto approval.

### Create a Custom Process Definition

1. Go to App **Manage Workflow for Service Quotation**

   The app includes the following key features:
   - View a list of available approval workflows
   - Display approval workflow details
   - Edit workflow details (Name, Start/End dates, Validity dates)
   - Define order in which workflow start conditions will be evaluated
   - Activate and deactivate workflows
   - Make a copy of an existing approval workflow
   - Delete a draft of an existing workflow
   - Set start conditions for workflows

2. Click on **Create**
3. Provide a meaningful description and start condition. The only standard start condition delivered is **Service Quotation Type**. Users can configure additional custom conditions (e.g., Sales Org or any other custom fields)
4. Define the **steps** — provide a step name and choose step type:
   - **Release of Service Quotation** — for agent-determined approval
   - **Automatic Release of Service Quotation** — for auto-approval scenarios

   Assignment By can be **Role** or **User**. If you choose User, the agent is picked from F4 help. For Role-based, the agent is determined via BAdI.

5. Define the **deadline** — the action when a certain threshold is crossed (mark step as overdue in MyInbox, or notify via email)
6. Configure **exception handling** — similar to deadline, but triggers actions that could be processed further in a workflow. With this release, the option is to cancel the workflow
7. Configure **email notification** — choose the **Initiator of the Workflow** to be notified via email when the workflow is approved or rejected

   > **Note:** The workflow initiator is the person who clicks the **Send for Approval** button.

8. **Activate** the newly created process definition

---

## Testing

1. Navigate to app **Manage Service Quotation**
2. Select a quotation in status **Open** or **Under Review**
3. Click on **Send for Approval** — this triggers the workflow to the configured agent
4. The agent checks the triggered workflow in app **My Inbox** and can take action: **Approve** or **Reject**

   > MyInbox app includes features like attachments, comments, and open tasks to help the agent make decisive decisions. Comments added in the notes section of the service quotation are visible to the agent in MyInbox.

---

*Though this blog has not covered authorization and gateway settings (beyond its scope), feel free to drop your queries/comments if you are stuck or facing technical challenges.*
