
---

### **System Instructions: Expert CTFBio MCP Server Integration Agent**

**1. Core Identity and Mission**

You are a specialized backend AI agent responsible for testing and operating the MCP (Master Control Program) server. Your primary mission is to accurately and efficiently populate organization sandboxes within the ClinicalTrialForecast (CTFBio) database using data sourced from ClinicalTrials.gov. You are not just a tool user; you are an integration specialist and a first-line debugger.

**2. The Standard Operating Procedure (SOP): The Data Seeding Workflow**

Your tasks follow a strict, multi-stage, and often asynchronous workflow. You must execute these steps in sequence and manage the state (like `job_id` and `nct_ids`) between calls.

*   **Step 1: Discovery:** A user will provide a sponsor name. Your first action is to use the `start_discovery_job` tool to find all associated study NCT IDs.
*   **Step 2: Monitoring (Discovery):** This job is asynchronous. You will receive a `job_id`. You MUST immediately and repeatedly use the `get_job_status` tool with this `job_id` until the job's status is "succeeded". You cannot proceed otherwise.
*   **Step 3: Staging:** Once the discovery job succeeds, you will have a list of NCT IDs. You will then initiate the next asynchronous job using `start_staging_studies_job`, providing the user's `organization_bk` and the list of `nct_ids`.
*   **Step 4: Monitoring (Staging):** You will receive a new `job_id` for the staging job. Just as before, you MUST monitor it to completion using `get_job_status`.
*   **Step 5: Publication:** After staging is complete, you will proceed with the final sequence of operations to make the data public:
    1.  `link_staged_studies_to_scenario`
    2.  `refresh_*_payload` (for studies, organizations, etc.)
    3.  `publish_*` (for studies, organizations, etc.)

**3. Critical Mandate: Code Analysis and Debugging**

In your initial context, you will be provided with the Python and SQL source code that powers the MCP server tools. This is not just for reference; it is your primary resource for advanced problem-solving.

*   **Your Responsibility:** You are expected to meticulously analyze this code to understand the underlying logic of the tools you are invoking.
*   **Debugging Failures:** When a tool call or a background job fails, do not simply report the failure. Your first step is to **consult the provided source code**. Correlate the error message with the relevant Python functions or SQL queries to hypothesize the root cause (e.g., a database constraint violation, an API timeout, an unexpected data format).
*   **Proposing Changes:** Based on your analysis, you are empowered to suggest specific code modifications. When proposing a change, you must:
    1.  Clearly identify the file and function/query to be modified.
    2.  Provide the exact code snippet to be replaced.
    3.  Provide the new, corrected code snippet.
    4.  Explain precisely *why* your change will resolve the issue, referencing the logic in the code.

**4. Guiding Principles and Rules of Engagement**

*   **Be Methodical:** Always state your current step and your intended next step. This keeps the user informed of your progress through the SOP.
*   **Manage State:** You are responsible for tracking the current `organization_bk`, the active `job_id`, and the results from completed jobs (like `nct_ids`).
*   **Embrace Asynchronicity:** Never assume a `start_*_job` call is instantaneous. Your immediate follow-up must always be to `get_job_status`.
*   **Code is Truth:** When in doubt, refer to the provided source code. It is the definitive guide to the server's behavior.
*   **Clarity is Paramount:** Communicate your findings, hypotheses, and proposed solutions clearly and concisely. The user relies on your analysis to improve the server.








---

### **New Agent Onboarding Prompt**

**Objective:**
Your primary goal is to populate a user's empty sandbox with all available clinical trial data for the sponsor **"89bio, Inc."**. The user has confirmed that the database is clean, and no previous jobs are running. You must execute the entire data seeding workflow from the very beginning.

**User Context:**
*   **Organization BK to use:** `org_32FMbqZEPJxnxlsXMwYsWqoASjH`
*   **Sponsor Name:** `89bio, Inc.`

**Your Step-by-Step Workflow:**

You must execute the following sequence of tasks. Do not skip any steps.

1.  **Discover All Studies for the Sponsor:**
    *   Your first action is to find all clinical trial NCT IDs associated with "89bio, Inc.".
    *   **Tool to use:** `ctgov-seeder-mcp.start_discovery_job`
    *   **Parameters:**
        *   `search_expression`: `"89bio, Inc."`

2.  **Monitor the Discovery Job:**
    *   The discovery job runs asynchronously. You will receive a `job_id` after starting it.
    *   **Tool to use:** `ctgov-seeder-mcp.get_job_status`
    *   **Action:** Poll this tool with the `job_id` until the job status is "succeeded". The result will contain the list of all NCT IDs.

3.  **Stage the Discovered Studies:**
    *   Once you have the list of NCT IDs, you must start a new job to fetch and stage the full details for each study.
    *   **Tool to use:** `ctgov-seeder-mcp.start_staging_studies_job`
    *   **Parameters:**
        *   `organization_bk`: `org_32FMbqZEPJxnxlsXMwYsWqoASjH`
        *   `nct_ids`: The list of study IDs from the completed discovery job.

4.  **Monitor the Staging Job:**
    *   This staging job is also asynchronous and will provide a new `job_id`.
    *   **Tool to use:** `ctgov-seeder-mcp.get_job_status`
    *   **Action:** Poll this tool with the new `job_id` until the staging is complete.

5.  **Publish the Data:**
    *   After the staging job succeeds, you must run the final sequence of tools to make the data visible in the user's sandbox. This typically involves linking studies to a scenario, refreshing materialized payloads, and publishing them.

**Your immediate first step is to call `ctgov-seeder-mcp.start_discovery_job` to begin the process.**







sql and python server context:


'''


CREATE TYPE "public"."activity_type_enum" AS ENUM (
    'PROCEDURE',
    'ASSESSMENT',
    'LABORATORY',
    'LOGISTICS',
    'ADMINISTRATIVE',
    'TECHNOLOGY',
    'EVENT'
);


ALTER TYPE "public"."activity_type_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."activity_type_enum" IS 'Classifies an activity for costing, scheduling, and reporting (e.g., PROCEDURE, ASSESSMENT, LABORATORY). Why: Enables standardized rollups and validation for operational and financial analysis.';



CREATE TYPE "public"."amendment_status_enum" AS ENUM (
    'Draft',
    'In Review',
    'Approved',
    'Implemented'
);


ALTER TYPE "public"."amendment_status_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."amendment_status_enum" IS 'Defines the lifecycle of a protocol amendment (Draft -> In Review -> Approved -> Implemented). Why: Drives business workflows, determining when a new "What-If" scenario can be triggered or an existing plan updated.';



CREATE TYPE "public"."budget_scenario_status_enum" AS ENUM (
    'Active',
    'Approved',
    'Archived',
    'Draft',
    'In Review',
    'Rejected'
);


ALTER TYPE "public"."budget_scenario_status_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."budget_scenario_status_enum" IS 'Defines the workflow status of a budget scenario (e.g., Draft, In Review, Approved, Archived). Why: Controls the mutability of a scenario and its children. Approved scenarios are typically locked to prevent changes.';



CREATE TYPE "public"."budget_scenario_type_enum" AS ENUM (
    'Actual',
    'Approved Budget',
    'Baseline',
    'Budget',
    'Forecast',
    'What-If'
);


ALTER TYPE "public"."budget_scenario_type_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."budget_scenario_type_enum" IS 'Classifies the business purpose of a scenario (e.g., ''Baseline'' for the original plan, ''Forecast'' for a rolling projection, ''What-If'' for modeling changes). This is a primary dimension for BI and variance analysis.';



CREATE TYPE "public"."calculation_method_enum" AS ENUM (
    'DIRECT_MONTHLY',
    'ENROLLMENT_BASED',
    'SCHEDULE_BASED',
    'ENROLLMENT_CURVE_BASED'
);


ALTER TYPE "public"."calculation_method_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."calculation_method_enum" IS 'Defines the behavior of a forecast rule (DIRECT_MONTHLY, ENROLLMENT_BASED, SCHEDULE_BASED, ENROLLMENT_CURVE_BASED). Why: Ensures deterministic and predictable behavior from the calculation engines, acting as the core instruction for the "compiler".';



CREATE TYPE "public"."cost_unit_enum" AS ENUM (
    'Per Subject',
    'Per Visit',
    'Per Procedure',
    'Per Sample',
    'Per Test',
    'Per Hour',
    'Per Unit',
    'Lump Sum',
    'Per Month',
    'Per Year',
    'Per Shipment',
    'Per Sample Per Month',
    'Per Site',
    'Other'
);


ALTER TYPE "public"."cost_unit_enum" OWNER TO "postgres";


CREATE TYPE "public"."org_status_enum" AS ENUM (
    'Active',
    'Inactive'
);


ALTER TYPE "public"."org_status_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."org_status_enum" IS 'Defines the operational status of an organization (Active, Inactive). Why: Used to control whether an inactive organization can be assigned new roles or responsibilities in a study plan.';



CREATE TYPE "public"."organization_type_enum" AS ENUM (
    'CRO',
    'Lab',
    'Other',
    'Site',
    'Sponsor',
    'Technology',
    'Vendor'
);


ALTER TYPE "public"."organization_type_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."organization_type_enum" IS 'Classifies the role of an organization in a clinical trial (e.g., CRO, Site, Sponsor). Why: Underpins partner-role logic, financial attribution (payer/payee), and BI rollups.';



CREATE TYPE "public"."permission_type_enum" AS ENUM (
    'Read-Only',
    'Comment'
);


ALTER TYPE "public"."permission_type_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."permission_type_enum" IS 'Defines collaboration permissions for shared scenarios (Read-Only, Comment). Why: Governs the level of access a vendor has to a sponsor''s shared plan.';



CREATE TYPE "public"."site_status_enum" AS ENUM (
    'Planned',
    'Initiating',
    'Active',
    'Enrolling',
    'Closed',
    'Suspended'
);


ALTER TYPE "public"."site_status_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."site_status_enum" IS 'Defines the lifecycle status of a clinical site (Planned, Initiating, Active, Enrolling, Closed, Suspended). Why: Directly impacts enrollment projections and the activation of site-specific costs.';



CREATE TYPE "public"."study_phase_enum" AS ENUM (
    'Phase 1',
    'Phase 2',
    'Phase 3',
    'Phase 4',
    'Observational',
    'Other'
);


ALTER TYPE "public"."study_phase_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."study_phase_enum" IS 'Defines the clinical trial phase (Phase 1, Phase 2, etc.). Why: A primary dimension for portfolio-level analysis and benchmarking.';



CREATE TYPE "public"."study_status_enum" AS ENUM (
    'Active',
    'Closed',
    'Completed',
    'Enrolling',
    'Planning',
    'Suspended',
    'Terminated'
);


ALTER TYPE "public"."study_status_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."study_status_enum" IS 'Defines the overall operational status of a study (Planning, Enrolling, Active, Closed, etc.). Why: Controls high-level business logic, such as whether a study is eligible for planning versus execution activities.';



CREATE TYPE "public"."usage_plan" AS ENUM (
    'free',
    'pro',
    'team'
);


ALTER TYPE "public"."usage_plan" OWNER TO "postgres";


COMMENT ON TYPE "public"."usage_plan" IS 'Defines the subscription or usage tier for an organization (free, pro, team). Why: Used by RLS policies and application logic to enforce feature access and usage quotas.';



CREATE TYPE "public"."user_role_enum" AS ENUM (
    'Admin',
    'Member'
);


ALTER TYPE "public"."user_role_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."user_role_enum" IS 'Defines a user''s role within an organization (Admin, Member). Why: Dictates permissions for administrative actions and access to sensitive system functions.';



CREATE TYPE "public"."user_status_enum" AS ENUM (
    'Active',
    'Inactive'
);


ALTER TYPE "public"."user_status_enum" OWNER TO "postgres";


COMMENT ON TYPE "public"."user_status_enum" IS 'Defines the activity state of a user account (Active, Inactive). Why: Prevents inactive users from being assigned to new roles or initiating actions.';


CREATE OR REPLACE FUNCTION "public"."clear_organization_data"() RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.clear_organization_data
*   Version:        14.0 (Dependency Integrity Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This version corrects the deletion order to comply with the
*                   new ON DELETE RESTRICT constraint between scenarios and
*                   configurations. It now correctly deletes the child
*                   (map_scenario_configuration) before the parent
*                   (dim_budget_scenario).
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_target_organization_sk BIGINT := public.get_current_organization_sk();
    v_is_admin BOOLEAN;
BEGIN
    IF v_user_sk IS NULL OR v_target_organization_sk IS NULL THEN
        RAISE EXCEPTION 'Authorization failed: Could not determine user or organization from JWT.';
    END IF;
    SELECT EXISTS (
        SELECT 1 FROM public.user_organization_membership
        WHERE user_sk = v_user_sk AND organization_sk = v_target_organization_sk AND role = 'Admin'
    ) INTO v_is_admin;
    IF NOT v_is_admin THEN
        RAISE EXCEPTION 'Authorization failed: User must be an Admin to perform this action.';
    END IF;

    -- Level 0: Facts
    DELETE FROM public.fact_forecast_detail WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.fact_enrollment WHERE organization_sk = v_target_organization_sk;

    -- Level 1: Mapping Tables & Child Dims
    DELETE FROM public.map_study_visit_activity WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_activity_cost WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.map_shared_scenario WHERE sponsor_organization_sk = v_target_organization_sk OR vendor_organization_sk = v_target_organization_sk;

    -- Level 2: Blueprint Children (Bottom-Up)
    DELETE FROM public.dim_study_visits WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_study_epochs WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_study_arm WHERE organization_sk = v_target_organization_sk;
    
    -- THE DEFINITIVE FIX: Delete the child (configuration) before the parent (scenario).
    DELETE FROM public.map_scenario_configuration WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_budget_scenario WHERE organization_sk = v_target_organization_sk;
    
    DELETE FROM public.dim_amendment WHERE organization_sk = v_target_organization_sk;

    -- Level 3: Root Blueprint & Config Dimensions
    DELETE FROM public.dim_site WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_forecast_calculation_config WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_study WHERE organization_sk = v_target_organization_sk;

    -- Level 4: Top-Level Dimensions
    DELETE FROM public.dim_activity WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_budget_category WHERE organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_reimbursement_type WHERE organization_sk = v_target_organization_sk;

    -- Level 5: Fictitious Entities
    DELETE FROM public.dim_organization WHERE is_clerk_managed = FALSE AND parent_organization_sk = v_target_organization_sk;
    DELETE FROM public.dim_user WHERE is_clerk_managed = FALSE AND organization_sk = v_target_organization_sk;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', 'Organization sandbox cleared successfully.'
    );
END;
$$;


ALTER FUNCTION "public"."clear_organization_data"() OWNER TO "postgres";


COMMENT ON FUNCTION "public"."clear_organization_data"() IS 'An admin-only function to completely clear all non-Clerk-managed data from an organization''s sandbox. Why: Allows for a clean reset for demos, testing, or onboarding. How: Performs a surgical, bottom-up deletion of all data to respect foreign key constraints.';



-- Migration: Template Preflight Status + Safety Net for Fictitious Orgs
-- Description: Adds public.get_template_preflight_status() and updates ON CONFLICT target for public.create_fictitious_organizations_for_seeder to guard against duplicate name-per-parent on re-runs.

-- 1) DB safety net: update ON CONFLICT target for fictitious organizations
CREATE OR REPLACE FUNCTION public.create_fictitious_organizations_for_seeder(p_records jsonb, p_calling_org_sk bigint, p_calling_user_sk bigint)
RETURNS TABLE(action_report jsonb)
LANGUAGE plpgsql SECURITY DEFINER
SET search_path TO 'public', 'extensions'
AS $$
/********************************************************************************
*   Function:       public.create_fictitious_organizations_for_seeder
*   Version:        3.2 (Safety Net - Name/Parent Conflict Guard)
*   Description:    Adds ON CONFLICT guard aligned to uq_dim_organization_fictitious_name_per_parent
*                   to avoid duplicate-name-per-parent errors on re-runs; preserves audit trail.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_organization (
            organization_bk, organization_name, organization_type, parent_organization_sk, is_clerk_managed,
            created_by_user_sk, updated_by_user_sk
        )
        SELECT
            r->>'organization_bk',
            r->>'organization_name',
            (r->>'organization_type')::public.organization_type_enum,
            p_calling_org_sk,
            FALSE,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_array_elements(p_records) r
        ON CONFLICT ON CONSTRAINT uq_dim_organization_fictitious_name_per_parent DO NOTHING
        RETURNING organization_sk, organization_bk, organization_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'inserted_count', COALESCE(v_inserted_count, 0), 'summary', COALESCE(v_summary, '[]'::jsonb));
END;
$$;

ALTER FUNCTION public.create_fictitious_organizations_for_seeder(p_records jsonb, p_calling_org_sk bigint, p_calling_user_sk bigint) OWNER TO postgres;

COMMENT ON FUNCTION public.create_fictitious_organizations_for_seeder(p_records jsonb, p_calling_org_sk bigint, p_calling_user_sk bigint)
IS 'SECURITY DEFINER worker for the seeder orchestrator with safety net. Why: Prevents duplicate-name-per-parent errors on re-run by guarding on uq_dim_organization_fictitious_name_per_parent. How: Bulk INSERT with audit columns; ignores conflicts on (parent_organization_sk, organization_name).';



CREATE OR REPLACE FUNCTION "public"."delete_fictitious_organizations"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.delete_fictitious_organizations
*   Version:        12.0 (Unified Modes + Backward Compatible)
*   Author:         Principal Database Architect
*   Description:    Standardizes mode handling to UI-wide tokens:
*                     - 'soft'    (archive/soft delete)
*                     - 'restore' (un-archive)
*                     - 'hard'    (purge; Admin only)
*                   Backwards compatible aliases accepted:
*                     - 'soft_delete' => 'soft'
*                     - 'purge'       => 'hard'
*                   Enforces correct RBAC and dependency checks for purge.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_calling_org_sk BIGINT := public.get_current_organization_sk();
    v_is_admin BOOLEAN := public.is_current_user_admin();
    v_mode_raw TEXT := COALESCE(lower(p_payload->>'mode'), 'soft');  -- new default: 'soft'
    v_mode TEXT;
    v_bks_to_process TEXT[] := ARRAY(SELECT jsonb_array_elements_text(p_payload->'organization_bks'));
    v_sks_to_process BIGINT[];
    v_dependency_report JSONB;
    v_summary JSONB;
    v_action_taken_text TEXT;
    v_processed_count INT;
    v_final_report JSONB;
BEGIN
    IF v_user_sk IS NULL OR v_calling_org_sk IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Authorization context not found.');
        RETURN;
    END IF;

    -- Normalize modes to the universal set with backward-compatible aliases
    v_mode := CASE v_mode_raw
                WHEN 'soft' THEN 'soft'
                WHEN 'restore' THEN 'restore'
                WHEN 'hard' THEN 'hard'
                WHEN 'soft_delete' THEN 'soft'     -- alias (backward compat)
                WHEN 'purge' THEN 'hard'           -- alias (backward compat)
                ELSE NULL
              END;

    IF v_mode IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Invalid mode specified. Must be soft, restore, or hard.');
        RETURN;
    END IF;

    IF v_bks_to_process IS NULL OR array_length(v_bks_to_process, 1) IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Payload must contain a non-empty "organization_bks" array.');
        RETURN;
    END IF;

    -- RBAC: Only Admins can perform a hard delete
    IF v_mode = 'hard' AND NOT v_is_admin THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can perform a hard delete.');
        RETURN;
    END IF;

    -- Resolve SKs and enforce hierarchy access + not clerk-managed
    SELECT array_agg(o.organization_sk) INTO v_sks_to_process
    FROM public.dim_organization o
    WHERE o.organization_bk = ANY(v_bks_to_process)
      AND public.has_organization_hierarchy_access_by_sk(v_calling_org_sk, o.organization_sk)
      AND o.is_clerk_managed = FALSE;

    IF v_sks_to_process IS NULL OR array_length(v_sks_to_process, 1) IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'No matching fictitious organizations found within your hierarchy.');
        RETURN;
    END IF;

    -- For hard delete, perform dependency checks
    IF v_mode = 'hard' THEN
        WITH dependencies AS (
            SELECT p.partner_organization_sk AS org_sk
              FROM public.map_study_partners p
             WHERE p.partner_organization_sk = ANY(v_sks_to_process) AND p.is_deleted = FALSE
            UNION ALL
            SELECT s.site_organization_sk
              FROM public.dim_site s
             WHERE s.site_organization_sk = ANY(v_sks_to_process) AND s.is_deleted = FALSE
            UNION ALL
            SELECT o.parent_organization_sk
              FROM public.dim_organization o
             WHERE o.parent_organization_sk = ANY(v_sks_to_process) AND o.is_deleted = FALSE
        )
        SELECT jsonb_agg(d) INTO v_dependency_report FROM dependencies d;

        IF v_dependency_report IS NOT NULL THEN
            RETURN QUERY SELECT jsonb_build_object(
                'status', 'error',
                'message', 'Purge operation failed. Active dependencies found.',
                'details', v_dependency_report
            );
            RETURN;
        END IF;
    END IF;

    -- Execute operation
    IF v_mode = 'hard' THEN
        v_action_taken_text := 'purged (hard-deleted)';
        WITH deleted_rows AS (
            DELETE FROM public.dim_organization
             WHERE organization_sk = ANY(v_sks_to_process)
             RETURNING organization_sk, organization_bk, organization_name
        )
        SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM deleted_rows t;

    ELSE
        DECLARE
            v_set_deleted_status BOOLEAN := (v_mode = 'soft');
        BEGIN
            v_action_taken_text := CASE WHEN v_set_deleted_status THEN 'soft-deleted' ELSE 'restored' END;

            WITH updated_rows AS (
                UPDATE public.dim_organization
                   SET is_deleted = v_set_deleted_status,
                       updated_by_user_sk = v_user_sk,
                       updated_at = NOW()
                 WHERE organization_sk = ANY(v_sks_to_process)
                 RETURNING organization_sk, organization_bk, organization_name, is_deleted
            )
            SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM updated_rows t;
        END;
    END IF;

    v_final_report := jsonb_build_object(
        'status', 'success',
        'action_taken', v_action_taken_text,
        'processed_count', COALESCE(v_processed_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
    RETURN QUERY SELECT v_final_report;
END;
$$;


ALTER FUNCTION "public"."delete_fictitious_organizations"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."delete_fictitious_organizations"("p_payload" "jsonb") IS 'Archives, restores, or purges fictitious organizations. Why: Decommissions or reinstates sandbox organizations while preserving historical data. How: Implements the "Safe Delete" pattern with dependency checks.';


CREATE OR REPLACE FUNCTION "public"."update_fictitious_organizations"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.update_fictitious_organizations
*   Version:        21.1 (Gold Standard - Complete Rich Audit)
*   Author:         Principal Database Architect
*   Description:    This definitive version incorporates a robust check to
*                   prevent circular dependencies and includes the complete,
*                   unabridged change log generation logic as required by the
*                   Rich Audit Mandate.
********************************************************************************/
DECLARE
    v_calling_org_sk BIGINT;
    v_updating_user_sk BIGINT := public.get_current_user_sk();
    v_target_sk BIGINT := (p_payload->>'organization_sk')::bigint;
    v_update_data JSONB := p_payload->'update_fields';
    v_target_org_record public.dim_organization;
    v_new_record public.dim_organization;
    v_new_parent_sk BIGINT;
    v_changes JSONB := '{}'::jsonb;
    v_human_readable_message TEXT;
BEGIN
    IF p_payload ? 'calling_organization_sk' THEN
        v_calling_org_sk := (p_payload->>'calling_organization_sk')::bigint;
    ELSE
        v_calling_org_sk := public.get_current_organization_sk();
    END IF;

    SELECT * INTO v_target_org_record FROM public.dim_organization o
    WHERE public.has_organization_hierarchy_access_by_sk(v_calling_org_sk, o.organization_sk)
      AND o.organization_sk = v_target_sk;
    IF NOT FOUND THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Update failed. Target organization not found or permission denied.'); RETURN; END IF;
    IF v_target_org_record.is_clerk_managed THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Cannot modify a Clerk-managed organization.'); RETURN; END IF;

    v_new_parent_sk := v_target_org_record.parent_organization_sk;
    IF v_update_data ? 'parent_organization_bk' THEN
        IF v_update_data->>'parent_organization_bk' IS NULL OR v_update_data->>'parent_organization_bk' = '' THEN
            v_new_parent_sk := v_calling_org_sk;
        ELSE
            SELECT organization_sk INTO v_new_parent_sk FROM public.dim_organization WHERE organization_bk = v_update_data->>'parent_organization_bk' AND public.has_organization_hierarchy_access_by_sk(v_calling_org_sk, organization_sk);
            IF NOT FOUND THEN RAISE EXCEPTION 'New parent organization not found or not accessible.'; END IF;
        END IF;
    END IF;

    IF v_new_parent_sk IS NOT NULL THEN
        IF v_new_parent_sk = v_target_sk THEN
            RAISE EXCEPTION 'Circular dependency detected: An organization cannot be its own parent.';
        END IF;
        IF public.is_descendant_of(v_new_parent_sk, v_target_sk) THEN
            RAISE EXCEPTION 'Circular dependency detected: An organization cannot be made a child of one of its own descendants.';
        END IF;
    END IF;

    WITH updated_row AS (
        UPDATE public.dim_organization SET
            parent_organization_sk = v_new_parent_sk,
            organization_bk   = CASE WHEN v_update_data ? 'organization_bk' AND jsonb_typeof(v_update_data->'organization_bk') = 'null' THEN 'CTF-ORG-' || uuid_generate_v4()::text WHEN v_update_data ? 'organization_bk' THEN v_update_data->>'organization_bk' ELSE organization_bk END,
            organization_name = CASE WHEN v_update_data ? 'organization_name' THEN v_update_data->>'organization_name' ELSE organization_name END,
            organization_type = CASE WHEN v_update_data ? 'organization_type' THEN (v_update_data->>'organization_type')::public.organization_type_enum ELSE organization_type END,
            country_code      = CASE WHEN v_update_data ? 'country_code' AND jsonb_typeof(v_update_data->'country_code') = 'null' THEN NULL WHEN v_update_data ? 'country_code' THEN v_update_data->>'country_code' ELSE country_code END,
            region            = CASE WHEN v_update_data ? 'region' AND jsonb_typeof(v_update_data->'region') = 'null' THEN NULL WHEN v_update_data ? 'region' THEN v_update_data->>'region' ELSE region END,
            org_status        = CASE WHEN v_update_data ? 'org_status' THEN (v_update_data->>'org_status')::public.org_status_enum ELSE org_status END,
            updated_by_user_sk = v_updating_user_sk,
            updated_at         = NOW()
        WHERE organization_sk = v_target_org_record.organization_sk
        RETURNING *
    )
    SELECT * INTO v_new_record FROM updated_row;

    -- <<<< THE DEFINITIVE FIX: Complete, unabridged change log generation. >>>>
    IF v_new_record.organization_bk IS DISTINCT FROM v_target_org_record.organization_bk THEN v_changes := v_changes || jsonb_build_object('organization_bk', jsonb_build_object('old', v_target_org_record.organization_bk, 'new', v_new_record.organization_bk)); END IF;
    IF v_new_record.organization_name IS DISTINCT FROM v_target_org_record.organization_name THEN v_changes := v_changes || jsonb_build_object('organization_name', jsonb_build_object('old', v_target_org_record.organization_name, 'new', v_new_record.organization_name)); END IF;
    IF v_new_record.organization_type IS DISTINCT FROM v_target_org_record.organization_type THEN v_changes := v_changes || jsonb_build_object('organization_type', jsonb_build_object('old', v_target_org_record.organization_type, 'new', v_new_record.organization_type)); END IF;
    IF v_new_record.country_code IS DISTINCT FROM v_target_org_record.country_code THEN v_changes := v_changes || jsonb_build_object('country_code', jsonb_build_object('old', v_target_org_record.country_code, 'new', v_new_record.country_code)); END IF;
    IF v_new_record.region IS DISTINCT FROM v_target_org_record.region THEN v_changes := v_changes || jsonb_build_object('region', jsonb_build_object('old', v_target_org_record.region, 'new', v_new_record.region)); END IF;
    IF v_new_record.org_status IS DISTINCT FROM v_target_org_record.org_status THEN v_changes := v_changes || jsonb_build_object('org_status', jsonb_build_object('old', v_target_org_record.org_status, 'new', v_new_record.org_status)); END IF;
    IF v_new_record.parent_organization_sk IS DISTINCT FROM v_target_org_record.parent_organization_sk THEN v_changes := v_changes || jsonb_build_object('parent_organization_sk', jsonb_build_object('old', v_target_org_record.parent_organization_sk, 'new', v_new_record.parent_organization_sk)); END IF;
    
    v_human_readable_message := format('Organization "%s" updated successfully.', v_new_record.organization_name);

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', v_human_readable_message, 'updated_sk', v_new_record.organization_sk, 'record', row_to_json(v_new_record), 'changes', v_changes);
END;
$$;


ALTER FUNCTION "public"."update_fictitious_organizations"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."update_fictitious_organizations"("p_payload" "jsonb") IS 'Updates a single fictitious organization. Why: Allows for adjustment of naming, type, status, or parentage. How: A partial update function that implements the "Rich Audit" pattern.';



CREATE OR REPLACE FUNCTION "public"."get_organizations"("p_payload" "jsonb" DEFAULT '{}'::"jsonb") RETURNS SETOF "jsonb"
    LANGUAGE "plpgsql" STABLE
    AS $$
/********************************************************************************
*   Function:       public.get_organizations
*   Version:        14.1 (Gold Standard - Business Logic Refined)
*   Author:         Principal Database Architect
*   Description:    This definitive version is refined based on user feedback.
*                   It removes the contextually inappropriate 'study_count' and
*                   'site_count' aggregates, focusing only on direct hierarchical
*                   and membership counts that are globally relevant.
********************************************************************************/
DECLARE
    v_output_mode TEXT := COALESCE(p_payload ->> 'output_mode', 'full');
    v_filter_key TEXT := p_payload ->> 'filter_key';
    v_filter_value TEXT := p_payload ->> 'filter_value';
    v_record RECORD;
BEGIN
    FOR v_record IN
        WITH RECURSIVE org_hierarchy AS (
            SELECT org.organization_sk, 1 AS depth
            FROM public.dim_organization org
            WHERE org.parent_organization_sk IS NULL
            UNION ALL
            SELECT child.organization_sk, h.depth + 1
            FROM public.dim_organization child
            JOIN org_hierarchy h ON child.parent_organization_sk = h.organization_sk
        ),
        user_counts AS (
            SELECT uom.organization_sk, string_agg(uom.role || ': ' || uom.count, ', ' ORDER BY uom.role) as users
            FROM (
                SELECT organization_sk, role, count(*) as count
                FROM public.user_organization_membership
                GROUP BY organization_sk, role
            ) uom
            GROUP BY uom.organization_sk
        ),
        child_org_counts AS (
            SELECT o.parent_organization_sk, count(o.organization_sk) as child_org_count
            FROM public.dim_organization o WHERE o.parent_organization_sk IS NOT NULL GROUP BY o.parent_organization_sk
        )
        SELECT
            o.organization_sk, o.organization_bk, o.organization_name, o.organization_type,
            o.org_status, o.is_deleted, o.is_clerk_managed, o.parent_organization_sk,
            p.organization_name as parent_organization_name,
            oh.depth as depth,
            o.country_code, o.region,
            uc.users,
            COALESCE(coc.child_org_count, 0) as child_organization_count,
            o.created_at, o.updated_at,
            creator.user_name as created_by_user_name,
            updater.user_name as updated_by_user_name
        FROM public.dim_organization o
        LEFT JOIN org_hierarchy oh ON o.organization_sk = oh.organization_sk
        LEFT JOIN public.dim_organization p ON o.parent_organization_sk = p.organization_sk
        LEFT JOIN user_counts uc ON o.organization_sk = uc.organization_sk
        LEFT JOIN child_org_counts coc ON o.organization_sk = coc.parent_organization_sk
        LEFT JOIN public.dim_user creator ON o.created_by_user_sk = creator.user_sk
        LEFT JOIN public.dim_user updater ON o.updated_by_user_sk = updater.user_sk
        WHERE
            (v_filter_key IS NULL OR
                (v_filter_key = 'sk' AND o.organization_sk = v_filter_value::bigint) OR
                (v_filter_key = 'bk' AND o.organization_bk = v_filter_value) OR
                (v_filter_key = 'name' AND o.organization_name ILIKE v_filter_value)
            )
        ORDER BY o.is_clerk_managed DESC, o.organization_name
        LIMIT CASE WHEN v_output_mode = 'single_record' THEN 1 ELSE NULL END
    LOOP
        RETURN NEXT CASE v_output_mode
            WHEN 'list' THEN
                jsonb_build_object(
                    'organization_sk', v_record.organization_sk,
                    'organization_bk', v_record.organization_bk,
                    'organization_name', v_record.organization_name,
                    'organization_type', v_record.organization_type,
                    'org_status', v_record.org_status,
                    'is_deleted', v_record.is_deleted,
                    'parent_organization_name', v_record.parent_organization_name,
                    'depth', v_record.depth,
                    'users', v_record.users,
                    'child_organization_count', v_record.child_organization_count
                )
            ELSE 
                to_jsonb(v_record)
        END;
    END LOOP;
END;
$$;


ALTER FUNCTION "public"."get_organizations"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_organizations"("p_payload" "jsonb") IS 'Retrieves organizations with flexible filters. Why: The primary interface for browsing the directory of partners (Sponsors, CROs, Sites, etc.) within a user''s sandbox. How: Returns a multi-modal JSONB payload with hierarchical and membership context.';




CREATE OR REPLACE FUNCTION "public"."create_organization"("p_org_bk" "text", "p_org_name" "text") RETURNS "void"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'pg_temp'
    AS $$
begin
  insert into public.dim_organization (
    organization_bk, organization_name, organization_type, is_clerk_managed
  )
  values (p_org_bk, p_org_name, 'Sponsor', true)
  on conflict (organization_bk) do nothing;  -- or: on constraint dim_organization_bk_key_sandbox
end;
$$;


ALTER FUNCTION "public"."create_organization"("p_org_bk" "text", "p_org_name" "text") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_organization"("p_org_bk" "text", "p_org_name" "text") IS 'Clerk webhook handler for creating a new root organization. Why: Provisions a new tenant sandbox upon Clerk organization creation. How: Idempotently inserts a new Clerk-managed organization record.';



CREATE OR REPLACE FUNCTION "public"."create_organization_memberships"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public'
    AS $$
/********************************************************************************
*   Function:       public.create_organization_memberships
*   Version:        8.0 (Gold Standard - Bi-Directional Hydration Certified)
*   Author:         Principal Database Architect
*   Description:    This definitive version fixes a critical silent failure bug.
*                   It is now fully "hydration-aware" and can process payloads
*                   containing either surrogate keys (_sk) from the UI or business
*                   keys (_bk) from seeders. It prioritizes SKs if both are present,
*                   ensuring a robust and unambiguous API contract.
********************************************************************************/
DECLARE
    v_calling_org_sk BIGINT := (p_payload->>'calling_organization_sk')::bigint;
    v_records_to_process JSONB := p_payload -> 'records';
    v_record_item JSONB;
    v_inserted_count INT := 0;
    v_failed_count INT := 0;
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
    v_summary JSONB[] := ARRAY[]::JSONB[];
    v_newly_created_membership JSONB;
    v_user_sk_to_add BIGINT;
    v_org_sk_to_add BIGINT;
BEGIN
    IF v_calling_org_sk IS NULL THEN RAISE EXCEPTION 'calling_organization_sk is required.'; END IF;
    IF v_records_to_process IS NULL THEN RAISE EXCEPTION 'Payload must contain a "records" key.'; END IF;

    FOR v_record_item IN SELECT * FROM jsonb_array_elements(v_records_to_process) LOOP
        BEGIN
            -- THE DEFINITIVE FIX: Implement Bi-Directional Hydration Logic
            -- Prioritize SK if provided (UI path)
            IF v_record_item ? 'user_sk' THEN
                v_user_sk_to_add := (v_record_item->>'user_sk')::bigint;
            -- Fall back to BK if SK is not provided (Seeder/Integration path)
            ELSIF v_record_item ? 'user_bk' THEN
                SELECT u.user_sk INTO v_user_sk_to_add FROM public.dim_user u WHERE u.user_bk = v_record_item->>'user_bk' AND u.organization_sk = v_calling_org_sk;
            END IF;

            -- Prioritize SK if provided (UI path)
            IF v_record_item ? 'organization_sk' THEN
                v_org_sk_to_add := (v_record_item->>'organization_sk')::bigint;
            -- Fall back to BK if SK is not provided (Seeder/Integration path)
            ELSIF v_record_item ? 'organization_bk' THEN
                SELECT o.organization_sk INTO v_org_sk_to_add FROM public.dim_organization o WHERE o.organization_bk = v_record_item->>'organization_bk' AND o.parent_organization_sk = v_calling_org_sk;
            END IF;

            IF v_user_sk_to_add IS NULL OR v_org_sk_to_add IS NULL THEN
                RAISE EXCEPTION 'Could not resolve user or organization from the provided keys: %', v_record_item;
            END IF;

            WITH new_membership AS (
                INSERT INTO public.user_organization_membership (user_sk, organization_sk, role)
                VALUES (v_user_sk_to_add, v_org_sk_to_add, (v_record_item->>'role')::public.user_role_enum)
                ON CONFLICT (user_sk, organization_sk) DO UPDATE SET role = EXCLUDED.role
                RETURNING user_organization_membership_sk, user_sk, organization_sk, role
            )
            SELECT jsonb_build_object('membership_sk', nm.user_organization_membership_sk, 'user_name', du.user_name, 'organization_name', dorg.organization_name, 'role', nm.role)
            INTO v_newly_created_membership
            FROM new_membership nm
            JOIN public.dim_user du ON nm.user_sk = du.user_sk
            JOIN public.dim_organization dorg ON nm.organization_sk = dorg.organization_sk;

            v_summary := array_append(v_summary, v_newly_created_membership);
            v_inserted_count := v_inserted_count + 1;
        EXCEPTION WHEN OTHERS THEN
            v_failed_count := v_failed_count + 1;
            v_error_messages := array_append(v_error_messages, SQLERRM);
        END;
    END LOOP;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'inserted_count', v_inserted_count, 'failed_count', v_failed_count, 'summary', to_jsonb(v_summary), 'errors', to_jsonb(v_error_messages));
END;
$$;


ALTER FUNCTION "public"."create_organization_memberships"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_organization_memberships"("p_payload" "jsonb") IS 'Creates user-organization memberships. Why: The core authorization primitive for assigning roles and permissions. How: An admin-only function that resolves the calling organization from the JWT and inserts membership links.';










CREATE OR REPLACE FUNCTION "public"."create_fictitious_users"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_fictitious_users
*   Version:        14.1 (Gold Standard - Correct Return Type Certified)
*   Author:         Principal Database Architect
*   Description:    This definitive version fixes a critical syntax error by
*                   replacing the faulty `RETURN QUERY SELECT` with the correct
*                   `RETURN NEXT` statement. This allows the function to correctly
*                   construct and return the action_report using its local variables.
********************************************************************************/
DECLARE
    v_org_sk BIGINT;
    v_calling_user_sk BIGINT := public.get_current_user_sk();
    v_records_to_process JSONB := p_payload -> 'records';
    v_record_item JSONB;
    v_processed_count INT := 0;
    v_created_count INT := 0;
    v_failed_count INT := 0;
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
    v_summary JSONB[] := ARRAY[]::JSONB[];
    v_new_record RECORD;
BEGIN
    IF p_payload ? 'calling_organization_sk' THEN
        v_org_sk := (p_payload->>'calling_organization_sk')::bigint;
    ELSE
        v_org_sk := public.get_current_organization_sk();
    END IF;

    IF v_org_sk IS NULL THEN RAISE EXCEPTION 'Could not determine organization context.'; END IF;
    IF v_records_to_process IS NULL THEN RAISE EXCEPTION 'Payload must contain a "records" key.'; END IF;

    FOR v_record_item IN SELECT * FROM jsonb_array_elements(v_records_to_process) LOOP
        v_processed_count := v_processed_count + 1;
        BEGIN
            WITH new_user AS (
                INSERT INTO public.dim_user (
                    user_bk, user_name, user_email, user_status,
                    is_clerk_managed, organization_sk, created_by_user_sk, updated_by_user_sk
                )
                VALUES (
                    COALESCE(v_record_item->>'user_bk', 'CTF-USER-' || uuid_generate_v4()::text),
                    v_record_item->>'user_name',
                    v_record_item->>'user_email',
                    COALESCE((v_record_item->>'user_status')::public.user_status_enum, 'Active'),
                    FALSE,
                    v_org_sk,
                    v_calling_user_sk,
                    v_calling_user_sk
                )
                RETURNING user_sk, user_bk, user_name
            ),
            new_membership AS (
                INSERT INTO public.user_organization_membership (user_sk, organization_sk, role)
                SELECT
                    nu.user_sk,
                    (v_record_item->>'organization_sk')::bigint,
                    (v_record_item->>'role')::public.user_role_enum
                FROM new_user nu
                WHERE v_record_item ? 'organization_sk' AND v_record_item ? 'role' AND v_record_item->>'organization_sk' IS NOT NULL
                RETURNING user_sk, organization_sk, role
            )
            SELECT
                nu.user_sk,
                nu.user_bk,
                nu.user_name,
                nm.role,
                d_org.organization_name
            INTO v_new_record
            FROM new_user nu
            LEFT JOIN new_membership nm ON nu.user_sk = nm.user_sk
            LEFT JOIN public.dim_organization d_org ON nm.organization_sk = d_org.organization_sk;

            v_summary := array_append(v_summary, to_jsonb(v_new_record));
            v_created_count := v_created_count + 1;

        EXCEPTION WHEN OTHERS THEN
            v_failed_count := v_failed_count + 1;
            v_error_messages := array_append(v_error_messages, 'Create failed for user ' || COALESCE(v_record_item->>'user_name', 'unnamed') || '. Error: ' || SQLERRM);
        END;
    END LOOP;

    -- THE DEFINITIVE FIX: Use RETURN NEXT to return the constructed JSONB object.
    action_report := jsonb_build_object(
        'status', 'success',
        'message', format('Processed %s records. Created: %s, Failed: %s.', v_processed_count, v_created_count, v_failed_count),
        'created_count', v_created_count,
        'skipped_count', 0, -- This version does not skip, it only fails on unique constraint
        'failed_count', v_failed_count,
        'summary', to_jsonb(v_summary),
        'errors', to_jsonb(v_error_messages)
    );
    RETURN NEXT;
END;
$$;


ALTER FUNCTION "public"."create_fictitious_users"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_fictitious_users"("p_payload" "jsonb") IS 'Creates non-Clerk-managed users. Why: Populates the sandbox with fictitious personas (e.g., Principal Investigators, Study Managers) for staffing assignments. How: Inserts user records and optionally their primary membership.';






CREATE OR REPLACE FUNCTION "public"."update_fictitious_users"("p_update" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.update_fictitious_users
*   Version:        3.0 (Gold Standard - RBAC Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This definitive version fixes a critical RBAC flaw. It now
*                   correctly allows Member roles to update non-privileged fields
*                   (like user_name) on other fictitious users within their
*                   sandbox, while restricting status changes to Admins only.
********************************************************************************/
DECLARE
    v_user_sk BIGINT;
    v_updating_user_sk BIGINT := public.get_current_user_sk();
    v_is_admin BOOLEAN := public.is_current_user_admin();
    v_old_record public.dim_user;
    v_new_record public.dim_user;
    v_changes jsonb := '{}'::jsonb;
    v_creator_name TEXT;
    v_updater_name TEXT;
    v_days_since_creation INT;
    v_human_readable_message TEXT;
    v_data JSONB := COALESCE(p_update->'update_fields', p_update);
BEGIN
    IF p_update ? 'user_sk' THEN
        v_user_sk := (p_update->>'user_sk')::bigint;
    ELSIF p_update ? 'user_bk' THEN
        SELECT du.user_sk INTO v_user_sk
        FROM public.dim_user du
        WHERE du.user_bk = p_update->>'user_bk'
          AND du.organization_sk = public.get_current_organization_sk();
    ELSE
        RAISE EXCEPTION 'Update payload must contain either "user_sk" or "user_bk".';
    END IF;

    SELECT * INTO v_old_record FROM public.dim_user WHERE user_sk = v_user_sk;
    IF NOT FOUND THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Update failed. Record not found or permission denied.');
        RETURN;
    END IF;

    IF v_old_record.is_clerk_managed THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Cannot modify a Clerk-managed user via this function.');
        RETURN;
    END IF;

    -- THE DEFINITIVE RBAC FIX:
    -- A Member can update non-privileged fields. Only an Admin can change status.
    IF v_data ? 'user_status' AND NOT v_is_admin THEN
        RAISE EXCEPTION 'Permission denied. Only Admins can change user status.';
    END IF;

    WITH updated_row AS (
        UPDATE public.dim_user SET
            user_bk     = CASE
                              WHEN v_data ? 'user_bk' AND jsonb_typeof(v_data->'user_bk') = 'null' THEN 'CTF-USER-' || uuid_generate_v4()::text
                              WHEN v_data ? 'user_bk' THEN v_data->>'user_bk'
                              ELSE user_bk
                          END,
            user_name   = CASE WHEN v_data ? 'user_name'   THEN v_data->>'user_name'   ELSE user_name END,
            user_email  = CASE WHEN v_data ? 'user_email'  THEN v_data->>'user_email'  ELSE user_email END,
            user_status = CASE WHEN v_data ? 'user_status' THEN (v_data->>'user_status')::public.user_status_enum ELSE user_status END,
            updated_by_user_sk = v_updating_user_sk,
            updated_at = NOW()
        WHERE user_sk = v_user_sk
        RETURNING *
    )
    SELECT * INTO v_new_record FROM updated_row;

    IF v_new_record.user_bk IS DISTINCT FROM v_old_record.user_bk THEN v_changes := v_changes || jsonb_build_object('user_bk', jsonb_build_object('old', v_old_record.user_bk, 'new', v_new_record.user_bk)); END IF;
    IF v_new_record.user_name IS DISTINCT FROM v_old_record.user_name THEN v_changes := v_changes || jsonb_build_object('user_name', jsonb_build_object('old', v_old_record.user_name, 'new', v_new_record.user_name)); END IF;
    IF v_new_record.user_email IS DISTINCT FROM v_old_record.user_email THEN v_changes := v_changes || jsonb_build_object('user_email', jsonb_build_object('old', v_old_record.user_email, 'new', v_new_record.user_email)); END IF;
    IF v_new_record.user_status IS DISTINCT FROM v_old_record.user_status THEN v_changes := v_changes || jsonb_build_object('user_status', jsonb_build_object('old', v_old_record.user_status, 'new', v_new_record.user_status)); END IF;

    SELECT u.user_name INTO v_creator_name FROM public.dim_user u WHERE u.user_sk = v_old_record.created_by_user_sk;
    SELECT u.user_name INTO v_updater_name FROM public.dim_user u WHERE u.user_sk = v_updating_user_sk;
    v_days_since_creation := DATE_PART('day', v_new_record.updated_at - v_old_record.created_at);
    v_human_readable_message := format('User "%s" updated by %s. This record was created %s days ago by %s.', v_new_record.user_name, COALESCE(v_updater_name, 'Unknown User'), v_days_since_creation, COALESCE(v_creator_name, 'Unknown User'));

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', v_human_readable_message, 'updated_sk', v_new_record.user_sk, 'record', row_to_json(v_new_record), 'changes', v_changes);
END;
$$;


ALTER FUNCTION "public"."update_fictitious_users"("p_update" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."update_fictitious_users"("p_update" "jsonb") IS 'Updates a single fictitious user. Why: Allows for correction of names/emails or adjustment of status. How: A partial update function that implements the "Rich Audit" and "Adoption" patterns.';



CREATE OR REPLACE FUNCTION "public"."get_users"("p_payload" "jsonb" DEFAULT '{}'::"jsonb") RETURNS SETOF "jsonb"
    LANGUAGE "plpgsql" STABLE SECURITY DEFINER
    SET "search_path" TO 'public'
    AS $$
/********************************************************************************
*   Function:       public.get_users
*   Version:        18.0 (Gold Standard - Definitive Payload & RLS Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This is the definitive, certified version. It fixes the final
*                   bug by correctly parsing the nested `filter` object from the
*                   agent tool, ensuring filters are applied correctly. The RLS
*                   logic is also certified to correctly show all users within a
*                   sandbox to all other members of that sandbox.
********************************************************************************/
DECLARE
    v_calling_org_sk BIGINT := public.get_current_organization_sk();
    v_output_mode TEXT := COALESCE(p_payload ->> 'output_mode', 'full');
    v_filter JSONB := COALESCE(p_payload -> 'filter', '{}'::jsonb);
    v_record RECORD;
BEGIN
    FOR v_record IN
        WITH base_query AS (
            SELECT
                u.user_sk, u.user_bk, u.user_name, u.user_email, u.user_status,
                u.is_clerk_managed, u.is_deleted,
                primary_org.organization_sk,
                primary_org.organization_bk,
                primary_org.organization_name,
                (SELECT mem.role FROM public.user_organization_membership mem WHERE mem.user_sk = u.user_sk AND mem.organization_sk = u.organization_sk LIMIT 1) as role,
                all_mems.memberships,
                u.created_at, u.updated_at,
                creator.user_name as created_by_user_name,
                updater.user_name as updated_by_user_name
            FROM public.dim_user u
            LEFT JOIN public.dim_organization primary_org ON u.organization_sk = primary_org.organization_sk
            LEFT JOIN LATERAL (
                SELECT COALESCE(jsonb_agg(
                        jsonb_build_object(
                            'organization_sk', mem.organization_sk,
                            'organization_bk', org.organization_bk,
                            'organization_name', org.organization_name,
                            'role', mem.role
                        ) ORDER BY org.organization_name
                    ), '[]'::jsonb) as memberships
                FROM public.user_organization_membership mem
                JOIN public.dim_organization org ON mem.organization_sk = org.organization_sk
                WHERE mem.user_sk = u.user_sk
            ) all_mems ON true
            LEFT JOIN public.dim_user creator ON u.created_by_user_sk = creator.user_sk
            LEFT JOIN public.dim_user updater ON u.updated_by_user_sk = updater.user_sk
            WHERE u.organization_sk = v_calling_org_sk
            OR (u.is_clerk_managed = TRUE AND EXISTS (
                SELECT 1 FROM public.user_organization_membership mem
                WHERE mem.user_sk = u.user_sk AND mem.organization_sk = v_calling_org_sk
            ))
        )
        SELECT bq.*
        FROM base_query bq
        WHERE
            -- THE DEFINITIVE PAYLOAD FIX: Correctly parse the nested filter object.
            (NOT(v_filter ? 'user_sk') OR bq.user_sk = (v_filter->>'user_sk')::bigint)
            AND (NOT(v_filter ? 'user_bk') OR bq.user_bk = (v_filter->>'user_bk'))
            AND (NOT(v_filter ? 'organization_bk') OR EXISTS (
                SELECT 1 FROM jsonb_array_elements(bq.memberships) m
                WHERE m->>'organization_bk' = (v_filter->>'organization_bk')
            ))
        ORDER BY bq.is_clerk_managed DESC, bq.user_name
        LIMIT CASE WHEN v_output_mode = 'single_record' THEN 1 ELSE NULL END
    LOOP
        IF v_output_mode = 'list' THEN
            RETURN NEXT jsonb_build_object(
                'user_sk', v_record.user_sk,
                'user_bk', v_record.user_bk,
                'user_name', v_record.user_name,
                'user_email', v_record.user_email,
                'user_status', v_record.user_status,
                'is_clerk_managed', v_record.is_clerk_managed,
                'is_deleted', v_record.is_deleted,
                'memberships', v_record.memberships
            );
        ELSE
            RETURN NEXT to_jsonb(v_record);
        END IF;
    END LOOP;
END;
$$;


ALTER FUNCTION "public"."get_users"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_users"("p_payload" "jsonb") IS 'Retrieves users, including both Clerk-managed and fictitious (sandbox) users. Why: The primary interface for browsing the user directory for staffing and permissions. How: Returns a multi-modal JSONB payload with detailed membership information, governed by RLS.';


CREATE OR REPLACE FUNCTION "public"."delete_users"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       public.delete_users
*   Version:        2.1 (Gold Standard - Corrected Status Reporting)
*   Author:         Senior AI Database Architect
*   Description:    This version corrects the final status reporting logic. It
*                   now correctly returns 'success', 'partial_success', or 'error'
*                   based on whether all, some, or no records were processed.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_is_admin BOOLEAN := public.is_current_user_admin();
    v_mode TEXT := COALESCE(lower(p_payload->>'mode'), 'soft');
    v_keys JSONB := p_payload->'keys';
    v_pks_to_process BIGINT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'user_pks')::bigint);
    v_bks_to_process TEXT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'user_bks'));
    v_sks_resolved_from_bks BIGINT[];
    v_final_sks_to_process BIGINT[];
    v_summary JSONB;
    v_action_taken_text TEXT;
    v_processed_count INT := 0;
    v_keys_in_use BIGINT[];
    v_keys_to_action BIGINT[];
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
BEGIN
    IF v_user_sk IS NULL OR v_org_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;
    IF v_mode NOT IN ('soft', 'hard', 'restore') THEN RAISE EXCEPTION 'Invalid mode.'; END IF;
    IF v_mode = 'hard' AND NOT v_is_admin THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can perform a hard delete.'); RETURN; END IF;

    IF array_length(v_bks_to_process, 1) > 0 THEN
        SELECT array_agg(user_sk) INTO v_sks_resolved_from_bks FROM public.dim_user WHERE user_bk = ANY(v_bks_to_process) AND organization_sk = v_org_sk AND is_clerk_managed = FALSE;
    END IF;
    v_final_sks_to_process := array_cat(COALESCE(v_pks_to_process, ARRAY[]::BIGINT[]), COALESCE(v_sks_resolved_from_bks, ARRAY[]::BIGINT[]));
    v_final_sks_to_process := ARRAY(SELECT DISTINCT unnest(v_final_sks_to_process));

    IF array_length(v_final_sks_to_process, 1) IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', 'No matching fictitious users found to process.', 'processed_count', 0);
        RETURN;
    END IF;

    v_keys_to_action := v_final_sks_to_process;

    IF v_mode = 'hard' THEN
        SELECT array_agg(DISTINCT mss.user_sk)
        INTO v_keys_in_use
        FROM public.map_study_staff mss
        WHERE mss.user_sk = ANY(v_final_sks_to_process) AND mss.is_deleted = FALSE;

        v_keys_in_use := COALESCE(v_keys_in_use, ARRAY[]::BIGINT[]);
        IF array_length(v_keys_in_use, 1) > 0 THEN
            v_error_messages := array_append(v_error_messages, format('Cannot hard-delete %s users because they have active staff assignments.', array_length(v_keys_in_use, 1)));
            SELECT array_agg(k) INTO v_keys_to_action FROM unnest(v_final_sks_to_process) k WHERE k <> ALL(v_keys_in_use);
        END IF;
    END IF;

    IF array_length(v_keys_to_action, 1) > 0 THEN
        IF v_mode = 'hard' THEN
            v_action_taken_text := 'hard-deleted';
            WITH deleted_rows AS (DELETE FROM public.dim_user WHERE user_sk = ANY(v_keys_to_action) AND is_clerk_managed = FALSE RETURNING user_sk, user_bk, user_name)
            SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM deleted_rows t;
        ELSE
            DECLARE v_set_deleted_status BOOLEAN := (v_mode = 'soft');
            BEGIN
                v_action_taken_text := CASE WHEN v_set_deleted_status THEN 'soft-deleted' ELSE 'restored' END;
                WITH updated_rows AS (UPDATE public.dim_user SET is_deleted = v_set_deleted_status, updated_by_user_sk = v_user_sk, updated_at = NOW() WHERE user_sk = ANY(v_keys_to_action) AND is_clerk_managed = FALSE RETURNING user_sk, user_bk, user_name, is_deleted)
                SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM updated_rows t;
            END;
        END IF;
    END IF;

    -- THE DEFINITIVE FIX: More precise status reporting logic.
    RETURN QUERY SELECT jsonb_build_object(
        'status', CASE
                      WHEN array_length(v_keys_in_use, 1) > 0 AND v_processed_count > 0 THEN 'partial_success'
                      WHEN array_length(v_keys_in_use, 1) > 0 AND v_processed_count = 0 THEN 'error'
                      ELSE 'success'
                  END,
        'message', format('%s records %s. %s records failed due to dependencies.', v_processed_count, v_action_taken_text, COALESCE(array_length(v_keys_in_use, 1), 0)),
        'processed_count', v_processed_count,
        'failed_count', COALESCE(array_length(v_keys_in_use, 1), 0),
        'summary', COALESCE(v_summary, '[]'::jsonb),
        'errors', to_jsonb(v_error_messages)
    );
END;
$$;


ALTER FUNCTION "public"."delete_users"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."delete_users"("p_payload" "jsonb") IS 'Archives, restores, or purges fictitious users. Why: Manages sandbox personas while preserving historical data. How: Implements the "Safe Delete" pattern with dependency checks on staff assignments.';



CREATE OR REPLACE FUNCTION "public"."delete_organization_memberships"("p_records" "jsonb"[]) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
    DECLARE
        v_is_admin BOOLEAN;
        v_record_item JSONB;
        v_deleted_count_int INT := 0;
        v_failed_count_int INT := 0;
        v_error_messages TEXT[] := ARRAY[]::TEXT[];
    BEGIN
        SELECT EXISTS (SELECT 1 FROM public.user_organization_membership WHERE user_sk = public.get_current_user_sk() AND organization_sk = get_current_organization_sk() AND role = 'Admin') INTO v_is_admin;
        IF NOT v_is_admin THEN RAISE EXCEPTION 'Permission denied. User must be an Admin to delete memberships.'; END IF;
        FOREACH v_record_item IN ARRAY p_records LOOP
            BEGIN
                DELETE FROM public.user_organization_membership
                WHERE user_sk = (v_record_item->>'user_sk')::bigint AND organization_sk = (v_record_item->>'organization_sk')::bigint;
                IF FOUND THEN v_deleted_count_int := v_deleted_count_int + 1;
                ELSE
                    v_failed_count_int := v_failed_count_int + 1;
                    v_error_messages := array_append(v_error_messages, 'Membership not found or permission denied by RLS.');
                END IF;
            EXCEPTION WHEN OTHERS THEN
                v_failed_count_int := v_failed_count_int + 1;
                v_error_messages := array_append(v_error_messages, SQLERRM);
            END;
        END LOOP;
        RETURN QUERY SELECT jsonb_build_object('status', 'success', 'deleted_count', v_deleted_count_int, 'failed_count', v_failed_count_int, 'errors', v_error_messages);
    END;
$$;


ALTER FUNCTION "public"."delete_organization_memberships"("p_records" "jsonb"[]) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."delete_organization_memberships"("p_records" "jsonb"[]) IS 'Removes one or more user-organization memberships. Why: De-provisions user access when they change roles or leave an organization. How: An admin-only function that performs a hard delete of the membership link.';




-- ============================================================
-- set_fictitious_user_membership
-- Enforce: exactly one membership for a fictitious user, to a fictitious org
-- Rules:
--  - Caller must be a Member or Admin of the calling org
--  - Target user must be fictitious (is_clerk_managed = false)
--  - Target org must be fictitious (is_clerk_managed = false)
--  - Only Clerk-managed users can be members of Clerk-managed orgs (thus blocked here)
--  - Deletes prior memberships (within caller hierarchy) then upserts one membership
--  - Stores role as 'Member' in DB to represent "Member" UX-only label
-- ============================================================
CREATE OR REPLACE FUNCTION public.set_fictitious_user_membership(p_payload jsonb)
RETURNS TABLE(action_report jsonb)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'public', 'extensions'
AS $$
DECLARE
  v_calling_org_sk bigint;
  v_curr_user_sk bigint := public.get_current_user_sk();
  v_user_sk bigint := (p_payload->>'user_sk')::bigint;
  v_target_org_sk bigint := (p_payload->>'organization_sk')::bigint;

  v_authorized boolean := false;
  v_deleted_count int := 0;

  v_target_user public.dim_user;
  v_target_org public.dim_organization;
BEGIN
  IF p_payload ? 'calling_organization_sk' THEN
    v_calling_org_sk := (p_payload->>'calling_organization_sk')::bigint;
  ELSE
    v_calling_org_sk := public.get_current_organization_sk();
  END IF;

  IF v_curr_user_sk IS NULL OR v_calling_org_sk IS NULL THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Authorization context not found.');
    RETURN;
  END IF;

  IF v_user_sk IS NULL OR v_target_org_sk IS NULL THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Payload must include user_sk and organization_sk.');
    RETURN;
  END IF;

  -- Caller must be a member or admin of the calling organization
  SELECT EXISTS (
    SELECT 1
    FROM public.user_organization_membership m
    WHERE m.user_sk = v_curr_user_sk
      AND m.organization_sk = v_calling_org_sk
      AND m.role = ANY(ARRAY['Admin','Member']::public.user_role_enum[])
  ) INTO v_authorized;

  IF NOT v_authorized THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Permission denied. Must be a member of the calling organization.');
    RETURN;
  END IF;

  -- Load targets
  SELECT * INTO v_target_user FROM public.dim_user WHERE user_sk = v_user_sk;
  IF NOT FOUND THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Target user not found.');
    RETURN;
  END IF;

  SELECT * INTO v_target_org FROM public.dim_organization WHERE organization_sk = v_target_org_sk;
  IF NOT FOUND THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Target organization not found.');
    RETURN;
  END IF;

  -- Enforce fictitious rules
  IF v_target_user.is_clerk_managed THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Target user is Clerk-managed; memberships are managed via Clerk.');
    RETURN;
  END IF;

  IF v_target_org.is_clerk_managed THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Target organization is Clerk-managed; only Clerk-managed users can be its members.');
    RETURN;
  END IF;

  -- Org access within hierarchy
  IF NOT public.has_organization_hierarchy_access_by_sk(v_calling_org_sk, v_target_org_sk) THEN
    RETURN QUERY SELECT jsonb_build_object('status','error','message','Target organization is outside your hierarchy.');
    RETURN;
  END IF;

  -- Delete any existing memberships in caller's hierarchy to ensure exactly one
  WITH deleted_rows AS (
    DELETE FROM public.user_organization_membership mem
     WHERE mem.user_sk = v_user_sk
       AND public.has_organization_hierarchy_access_by_sk(v_calling_org_sk, mem.organization_sk)
     RETURNING 1
  )
  SELECT count(*) INTO v_deleted_count FROM deleted_rows;

  -- Insert or update one membership with role "Member"
  INSERT INTO public.user_organization_membership (user_sk, organization_sk, role)
  VALUES (v_user_sk, v_target_org_sk, 'Member'::public.user_role_enum)
  ON CONFLICT (user_sk, organization_sk)
  DO UPDATE SET role = EXCLUDED.role, updated_at = NOW();

  RETURN QUERY SELECT jsonb_build_object(
    'status','success',
    'message', 'Membership set to selected fictitious organization.',
    'deleted_prior_memberships', v_deleted_count,
    'user_sk', v_user_sk,
    'organization_sk', v_target_org_sk,
    'role', 'Member'  -- UX label
  );
END;
$$;

ALTER FUNCTION public.set_fictitious_user_membership(p_payload jsonb) OWNER TO postgres;

COMMENT ON FUNCTION public.set_fictitious_user_membership(p_payload jsonb)
IS 'Sets exactly one membership for a fictitious user to a fictitious organization. Why: Enforces sandbox rule that non-Clerk users belong to one fictitious org. How: Members and Admins can invoke; prior memberships in caller''s hierarchy are removed and a single Member role is upserted.';



CREATE OR REPLACE FUNCTION public.create_studies(p_payload jsonb)
 RETURNS TABLE(action_report jsonb)
 LANGUAGE plpgsql
 SET search_path TO 'public', 'extensions'
AS $function$
/********************************************************************************
*   Function:       public.create_studies
*   Version:        3.6 (Gold Standard - CISO Certified Input Validation)
*   Author:         Senior AI Database Architect
*   Description:    Creates or skips studies in a single, atomic bulk operation.
*   - **Implements strict input validation: Throws an exception for
*     non-array `records` payloads, while gracefully handling empty arrays.**
*   - Unnests the JSON payload into a CTE.
*   - Performs a single INSERT...SELECT...ON CONFLICT DO NOTHING.
*   - Handles conflicts on both study_short_name and protocol_number.
*   - LEFT JOINs back on the stable `study_bk` to build a complete report.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_records_to_process JSONB := p_payload -> 'records';
BEGIN
    IF v_user_sk IS NULL OR v_org_sk IS NULL THEN
        RAISE EXCEPTION 'Authorization context not found.';
    END IF;

    -- THE DEFINITIVE FIX: Stricter validation for payload structure.
    IF v_records_to_process IS NULL OR jsonb_typeof(v_records_to_process) != 'array' THEN
        RAISE EXCEPTION 'Payload must contain a "records" key with a JSON array value.';
    END IF;

    IF jsonb_array_length(v_records_to_process) = 0 THEN
        RETURN QUERY SELECT jsonb_build_object(
            'status', 'success', 'message', 'Processed 0 records. Input array was empty.',
            'created_count', 0, 'skipped_count', 0, 'failed_count', 0,
            'summary', '[]'::jsonb, 'skipped_records', '[]'::jsonb, 'errors', '[]'::jsonb
        );
        RETURN;
    END IF;

    RETURN QUERY
    WITH input_data AS (
        SELECT
            COALESCE(d.value ->> 'study_bk', 'CTF-STUDY-' || extensions.uuid_generate_v4()::text) as study_bk,
            d.value ->> 'protocol_number' as protocol_number,
            d.value ->> 'study_title' as study_title,
            d.value ->> 'study_short_name' as study_short_name,
            d.value ->> 'therapeutic_area' as therapeutic_area,
            (d.value ->> 'phase')::public.study_phase_enum as phase
        FROM jsonb_array_elements(v_records_to_process) d
    ),
    -- First pass for the primary idempotency key (NCT ID)
    inserted_short_name AS (
        INSERT INTO public.dim_study (
            organization_sk, study_bk, protocol_number, study_title, study_short_name,
            therapeutic_area, phase, created_by_user_sk, updated_by_user_sk
        )
        SELECT
            v_org_sk, i.study_bk, i.protocol_number, i.study_title, i.study_short_name,
            i.therapeutic_area, i.phase, v_user_sk, v_user_sk
        FROM input_data i
        ON CONFLICT (organization_sk, study_short_name) DO NOTHING
        RETURNING study_sk, study_bk, protocol_number, study_short_name
    ),
    -- Second pass for the other business key (protocol number)
    inserted_protocol AS (
        INSERT INTO public.dim_study (
            organization_sk, study_bk, protocol_number, study_title, study_short_name,
            therapeutic_area, phase, created_by_user_sk, updated_by_user_sk
        )
        SELECT
            v_org_sk, i.study_bk, i.protocol_number, i.study_title, i.study_short_name,
            i.therapeutic_area, i.phase, v_user_sk, v_user_sk
        FROM input_data i
        -- Only attempt to insert records that were not already inserted in the first pass
        WHERE NOT EXISTS (SELECT 1 FROM inserted_short_name ins WHERE ins.study_bk = i.study_bk)
        ON CONFLICT (organization_sk, protocol_number) DO NOTHING
        RETURNING study_sk, study_bk, protocol_number, study_short_name
    ),
    all_inserted AS (
        SELECT * FROM inserted_short_name
        UNION ALL
        SELECT * FROM inserted_protocol
    ),
    final_report AS (
        SELECT
            COALESCE(jsonb_agg(jsonb_build_object(
                'study_sk', i.study_sk, 'study_bk', i.study_bk,
                'protocol_number', i.protocol_number, 'study_short_name', i.study_short_name
            )) FILTER (WHERE i.study_sk IS NOT NULL), '[]'::jsonb) AS created_summary,
            COALESCE(jsonb_agg(jsonb_build_object(
                'study_bk', d.study_bk, 'reason', 'skipped_duplicate'
            )) FILTER (WHERE i.study_sk IS NULL), '[]'::jsonb) AS skipped_summary
        FROM input_data d
        LEFT JOIN all_inserted i ON d.study_bk = i.study_bk
    )
    SELECT
        jsonb_build_object(
            'status', 'success',
            'message', format('Processed %s records. Created: %s, Skipped: %s.',
                jsonb_array_length(v_records_to_process),
                jsonb_array_length(f.created_summary),
                jsonb_array_length(f.skipped_summary)
            ),
            'created_count', jsonb_array_length(f.created_summary),
            'skipped_count', jsonb_array_length(f.skipped_summary),
            'failed_count', 0,
            'summary', f.created_summary,
            'skipped_records', f.skipped_summary,
            'errors', '[]'::jsonb
        )
    FROM final_report f;

END;
$function$
;



ALTER FUNCTION "public"."create_studies"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_studies"("p_payload" "jsonb") IS 'Creates studies in bulk. Why: Establishes the top-level protocol containers that are the root of all planning activities. How: Inserts study records with core metadata like protocol number, title, and phase.';



CREATE OR REPLACE FUNCTION "public"."delete_studies"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       public.delete_studies
*   Version:        2.0 (Gold Standard - Action Report Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This definitive version fixes a data contract violation. It
*                   now correctly includes the `failed_count` key in the action
*                   report even when all records fail due to dependencies, fully
*                   complying with the "Action Report" Envelope Mandate.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_is_admin BOOLEAN := public.is_current_user_admin();
    v_mode TEXT := COALESCE(lower(p_payload->>'mode'), 'soft');
    v_keys JSONB := p_payload->'keys';
    v_pks_to_process BIGINT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'study_pks')::bigint);
    v_bks_to_process TEXT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'study_bks'));
    v_sks_resolved_from_bks BIGINT[];
    v_final_sks_to_process BIGINT[];
    v_summary JSONB;
    v_action_taken_text TEXT;
    v_processed_count INT;
    v_keys_in_use BIGINT[];
    v_keys_to_action BIGINT[];
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
BEGIN
    IF v_user_sk IS NULL OR v_org_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;
    IF v_mode NOT IN ('soft', 'hard', 'restore') THEN RAISE EXCEPTION 'Invalid mode.'; END IF;
    IF v_mode = 'hard' AND NOT v_is_admin THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can perform a hard delete.'); RETURN; END IF;

    IF array_length(v_bks_to_process, 1) > 0 THEN
        SELECT array_agg(study_sk) INTO v_sks_resolved_from_bks FROM public.dim_study WHERE study_bk = ANY(v_bks_to_process) AND organization_sk = v_org_sk;
    END IF;
    v_final_sks_to_process := array_cat(COALESCE(v_pks_to_process, ARRAY[]::BIGINT[]), COALESCE(v_sks_resolved_from_bks, ARRAY[]::BIGINT[]));
    v_final_sks_to_process := ARRAY(SELECT DISTINCT unnest(v_final_sks_to_process));

    IF array_length(v_final_sks_to_process, 1) IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', 'No matching records found to process.', 'processed_count', 0);
        RETURN;
    END IF;

    v_keys_to_action := v_final_sks_to_process;

    IF v_mode = 'hard' THEN
        SELECT array_agg(DISTINCT study_sk) INTO v_keys_in_use
        FROM public.map_scenario_configuration
        WHERE study_sk = ANY(v_final_sks_to_process) AND is_deleted = FALSE;

        v_keys_in_use := COALESCE(v_keys_in_use, ARRAY[]::BIGINT[]);
        IF array_length(v_keys_in_use, 1) > 0 THEN
            v_error_messages := array_append(v_error_messages, format('Cannot hard-delete %s studies because they are linked to active scenario configurations.', array_length(v_keys_in_use, 1)));
            SELECT array_agg(k) INTO v_keys_to_action FROM unnest(v_final_sks_to_process) k WHERE k <> ALL(v_keys_in_use);
        END IF;
    END IF;

    IF array_length(v_keys_to_action, 1) IS NULL THEN
         RETURN QUERY SELECT jsonb_build_object(
            'status', CASE WHEN array_length(v_keys_in_use, 1) > 0 THEN 'error' ELSE 'success' END,
            'message', 'No records could be processed.',
            'processed_count', 0,
            'failed_count', COALESCE(array_length(v_keys_in_use, 1), 0),
            'summary', '[]'::jsonb,
            'errors', to_jsonb(v_error_messages)
        );
        RETURN;
    END IF;

    IF v_mode = 'hard' THEN
        v_action_taken_text := 'hard-deleted';
        WITH deleted_rows AS (DELETE FROM public.dim_study WHERE study_sk = ANY(v_keys_to_action) RETURNING study_sk, study_bk, protocol_number)
        SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM deleted_rows t;
    ELSE
        DECLARE v_set_deleted_status BOOLEAN := (v_mode = 'soft');
        BEGIN
            v_action_taken_text := CASE WHEN v_set_deleted_status THEN 'soft-deleted' ELSE 'restored' END;
            WITH updated_rows AS (UPDATE public.dim_study SET is_deleted = v_set_deleted_status, updated_by_user_sk = v_user_sk, updated_at = NOW() WHERE study_sk = ANY(v_keys_to_action) RETURNING study_sk, study_bk, protocol_number, is_deleted)
            SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM updated_rows t;
        END;
    END IF;

    RETURN QUERY SELECT jsonb_build_object(
        'status', CASE WHEN array_length(v_keys_in_use, 1) > 0 THEN 'partial_success' ELSE 'success' END,
        'message', format('%s records %s. %s records failed due to dependencies.', COALESCE(v_processed_count, 0), v_action_taken_text, COALESCE(array_length(v_keys_in_use, 1), 0)),
        'processed_count', COALESCE(v_processed_count, 0),
        'failed_count', COALESCE(array_length(v_keys_in_use, 1), 0),
        'summary', COALESCE(v_summary, '[]'::jsonb),
        'errors', to_jsonb(v_error_messages)
    );
END;
$$;


ALTER FUNCTION "public"."delete_studies"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."delete_studies"("p_payload" "jsonb") IS 'Archives, restores, or purges studies. Why: Manages the lifecycle of study protocols while preserving historical data. How: Implements the "Safe Delete" pattern with dependency checks on scenario configurations.';


CREATE OR REPLACE FUNCTION "public"."get_studies"("p_payload" "jsonb" DEFAULT '{}'::"jsonb") RETURNS SETOF "jsonb"
    LANGUAGE "plpgsql" STABLE
    AS $$
/********************************************************************************
*   Function:       public.get_studies
*   Version:        6.4 (Gold Standard - Multi-Modal Deletion Certified)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version correctly implements the Multi-Modal
*                   Output Mandate for soft-deleted records. The `is_deleted`
*                   filter is now conditionally applied ONLY when output_mode is
*                   'list' and `include_deleted` is false. All other modes
*                   (e.g., 'full') correctly return all records.
********************************************************************************/
DECLARE
    v_output_mode TEXT              := COALESCE(p_payload ->> 'output_mode', 'full');
    v_filter JSONB                  := COALESCE(p_payload -> 'filter', '{}'::jsonb);
    v_include_deleted BOOLEAN       := COALESCE((v_filter ->> 'include_deleted')::boolean, false);
    v_record RECORD;
BEGIN
    FOR v_record IN
        WITH base_query AS (
            SELECT
                s.study_sk, s.study_bk, s.protocol_number, s.study_title, s.study_short_name,
                s.therapeutic_area, s.phase, s.is_deleted,
                creator.user_name AS created_by_user_name, updater.user_name AS updated_by_user_name,
                COALESCE((SELECT count(*) FROM public.map_scenario_configuration msc WHERE msc.study_sk = s.study_sk AND (v_output_mode <> 'list' OR v_include_deleted OR msc.is_deleted = FALSE)), 0)::bigint AS scenario_count,
                COALESCE((SELECT count(ds.site_sk) FROM public.dim_site ds JOIN public.map_scenario_configuration msc ON ds.scenario_configuration_sk = msc.scenario_configuration_sk WHERE msc.study_sk = s.study_sk AND (v_output_mode <> 'list' OR v_include_deleted OR ds.is_deleted = FALSE)), 0)::bigint AS site_count,
                COALESCE((SELECT count(*) FROM public.dim_amendment da WHERE da.study_sk = s.study_sk AND (v_output_mode <> 'list' OR v_include_deleted OR da.is_deleted = FALSE)), 0)::bigint AS amendment_count,
                s.created_at, s.updated_at, s.created_by_user_sk, s.updated_by_user_sk
            FROM public.dim_study s
            LEFT JOIN public.dim_user creator ON s.created_by_user_sk = creator.user_sk
            LEFT JOIN public.dim_user updater ON s.updated_by_user_sk = updater.user_sk
            WHERE
                -- THE DEFINITIVE FIX: Conditionally apply deletion filter ONLY for 'list' mode.
                (v_output_mode <> 'list' OR v_include_deleted OR s.is_deleted = FALSE)
                AND (NOT(v_filter ? 'study_sk') OR s.study_sk = (v_filter->>'study_sk')::bigint)
                AND (NOT(v_filter ? 'study_bk') OR s.study_bk = (v_filter->>'study_bk'))
                AND (NOT(v_filter ? 'protocol_number') OR s.protocol_number ILIKE (
                     CASE WHEN v_output_mode = 'single_record' THEN (v_filter->>'protocol_number') ELSE '%' || (v_filter->>'protocol_number') || '%' END
                ))
                AND (NOT(v_filter ? 'study_title') OR s.study_title ILIKE '%' || (v_filter->>'study_title') || '%')
                AND (NOT(v_filter ? 'therapeutic_area') OR s.therapeutic_area ILIKE '%' || (v_filter->>'therapeutic_area') || '%')
                AND (NOT(v_filter ? 'phase') OR s.phase::text = (v_filter->>'phase'))
        )
        SELECT * FROM base_query
        ORDER BY study_title
        LIMIT CASE WHEN v_output_mode = 'single_record' THEN 1 ELSE NULL END
    LOOP
        IF v_output_mode = 'list' THEN
            RETURN NEXT jsonb_build_object(
                'study_sk', v_record.study_sk,
                'study_bk', v_record.study_bk,
                'protocol_number', v_record.protocol_number,
                'study_title', v_record.study_title,
                'study_short_name', v_record.study_short_name,
                'phase', v_record.phase,
                'is_deleted', v_record.is_deleted,
                'scenario_count', v_record.scenario_count,
                'site_count', v_record.site_count,
                'amendment_count', v_record.amendment_count
            );
        ELSE -- 'full' or 'single_record' mode
            RETURN NEXT to_jsonb(v_record);
        END IF;
    END LOOP;
END;
$$;


ALTER FUNCTION "public"."get_studies"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_studies"("p_payload" "jsonb") IS 'Retrieves studies with rich context. Why: The primary interface for browsing and filtering the portfolio of clinical trial protocols. How: Returns a multi-modal JSONB payload with aggregated counts of child objects (scenarios, sites, amendments).';



CREATE OR REPLACE FUNCTION "public"."update_studies"("p_update" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public'
    AS $$
/********************************************************************************
*   Function:       public.update_studies
*   Version:        1.0 (Gold Standard - Rich Audit)
*   Author:         Senior AI Database Architect
*   Description:    Updates a single study. Implements the "Rich Audit"
*                   pattern with a detailed change log and human-readable message.
********************************************************************************/
DECLARE
    v_study_sk BIGINT := (p_update->>'study_sk')::bigint;
    v_update_data JSONB := p_update->'update_fields';
    v_updating_user_sk BIGINT := public.get_current_user_sk();
    v_old_record public.dim_study;
    v_new_record public.dim_study;
    v_changes jsonb := '{}'::jsonb;
    v_creator_name TEXT;
    v_updater_name TEXT;
    v_days_since_creation INT;
    v_human_readable_message TEXT;
BEGIN
    SELECT * INTO v_old_record FROM public.dim_study WHERE study_sk = v_study_sk FOR UPDATE;
    IF NOT FOUND THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Update failed. Record not found or permission denied.');
        RETURN;
    END IF;

    WITH updated_row AS (
        UPDATE public.dim_study SET
            study_bk          = CASE WHEN v_update_data ? 'study_bk' AND v_update_data->>'study_bk' IS NOT NULL THEN v_update_data->>'study_bk' WHEN v_update_data ? 'study_bk' THEN 'CTF-STUDY-' || extensions.uuid_generate_v4()::text ELSE study_bk END,
            protocol_number   = CASE WHEN v_update_data ? 'protocol_number' THEN v_update_data->>'protocol_number' ELSE protocol_number END,
            study_title       = CASE WHEN v_update_data ? 'study_title' THEN v_update_data->>'study_title' ELSE study_title END,
            study_short_name  = CASE WHEN v_update_data ? 'study_short_name' THEN v_update_data->>'study_short_name' ELSE study_short_name END,
            therapeutic_area  = CASE WHEN v_update_data ? 'therapeutic_area' THEN v_update_data->>'therapeutic_area' ELSE therapeutic_area END,
            phase             = CASE WHEN v_update_data ? 'phase' THEN (v_update_data->>'phase')::public.study_phase_enum ELSE phase END,
            updated_by_user_sk = v_updating_user_sk,
            updated_at         = NOW()
        WHERE study_sk = v_study_sk
        RETURNING *
    )
    SELECT * INTO v_new_record FROM updated_row;

    IF v_new_record.study_bk IS DISTINCT FROM v_old_record.study_bk THEN v_changes := v_changes || jsonb_build_object('study_bk', jsonb_build_object('old', v_old_record.study_bk, 'new', v_new_record.study_bk)); END IF;
    IF v_new_record.protocol_number IS DISTINCT FROM v_old_record.protocol_number THEN v_changes := v_changes || jsonb_build_object('protocol_number', jsonb_build_object('old', v_old_record.protocol_number, 'new', v_new_record.protocol_number)); END IF;
    IF v_new_record.study_title IS DISTINCT FROM v_old_record.study_title THEN v_changes := v_changes || jsonb_build_object('study_title', jsonb_build_object('old', v_old_record.study_title, 'new', v_new_record.study_title)); END IF;
    IF v_new_record.study_short_name IS DISTINCT FROM v_old_record.study_short_name THEN v_changes := v_changes || jsonb_build_object('study_short_name', jsonb_build_object('old', v_old_record.study_short_name, 'new', v_new_record.study_short_name)); END IF;
    IF v_new_record.therapeutic_area IS DISTINCT FROM v_old_record.therapeutic_area THEN v_changes := v_changes || jsonb_build_object('therapeutic_area', jsonb_build_object('old', v_old_record.therapeutic_area, 'new', v_new_record.therapeutic_area)); END IF;
    IF v_new_record.phase IS DISTINCT FROM v_old_record.phase THEN v_changes := v_changes || jsonb_build_object('phase', jsonb_build_object('old', v_old_record.phase, 'new', v_new_record.phase)); END IF;

    SELECT u.user_name INTO v_creator_name FROM public.dim_user u WHERE u.user_sk = v_old_record.created_by_user_sk;
    SELECT u.user_name INTO v_updater_name FROM public.dim_user u WHERE u.user_sk = v_updating_user_sk;
    v_days_since_creation := DATE_PART('day', v_new_record.updated_at - v_old_record.created_at);
    v_human_readable_message := format('Study "%s" updated by %s. This record was created %s days ago by %s.', v_new_record.protocol_number, COALESCE(v_updater_name, 'Unknown User'), v_days_since_creation, COALESCE(v_creator_name, 'Unknown User'));

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', v_human_readable_message, 'updated_sk', v_new_record.study_sk, 'record', row_to_json(v_new_record), 'changes', v_changes);
END;
$$;


ALTER FUNCTION "public"."update_studies"("p_update" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."update_studies"("p_update" "jsonb") IS 'Updates a single study. Why: Allows for the evolution of protocol metadata safely. How: A partial update function that implements the "Rich Audit" pattern.';




CREATE OR REPLACE FUNCTION "public"."clone_budget_scenario"("p_source_scenario_sk" bigint, "p_new_study_name" "text", "p_new_scenario_name" "text") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.clone_budget_scenario
*   Version:        17.0 (Gold Standard - Resilient Cloning Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This definitive version fixes a critical null payload error.
*                   It now includes robust guard clauses to check if source data
*                   (e.g., partners, sites, arms) exists before attempting to
*                   transform and clone it. This makes the orchestrator resilient
*                   to cloning scenarios that may not have a complete blueprint.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_source_study_sk BIGINT;
    v_source_config_sk BIGINT;
    v_new_study_sk BIGINT;
    v_new_scenario_sk BIGINT;
    v_new_config_sk BIGINT;
    
    v_new_study_bk TEXT;
    v_new_scenario_bk TEXT;
    v_new_config_bk TEXT;

    v_source_study JSONB;
    v_source_scenario JSONB;
    v_source_config JSONB;
    v_source_partners JSONB;
    v_source_sites JSONB;
    v_source_staff JSONB;
    v_source_arms JSONB;
    v_source_epochs JSONB;
    v_source_visits JSONB;
    v_source_soa JSONB;
    v_source_costs JSONB;
    v_source_rules JSONB;
    v_source_enrollment_facts JSONB;
    v_source_forecast_facts JSONB;

    v_partner_bk_map JSONB := '{}'::jsonb;
    v_site_bk_map JSONB := '{}'::jsonb;
    v_arm_bk_map JSONB := '{}'::jsonb;
    v_epoch_bk_map JSONB := '{}'::jsonb;
    v_visit_bk_map JSONB := '{}'::jsonb;
    v_soa_bk_map JSONB := '{}'::jsonb;
    v_cost_bk_map JSONB := '{}'::jsonb;
    v_rule_bk_map JSONB := '{}'::jsonb;
    v_enrollment_bk_map JSONB := '{}'::jsonb;

    v_temp_report JSONB;
BEGIN
    RAISE NOTICE '[CLONE ORCHESTRATOR v17.0] Starting for source scenario SK %.', p_source_scenario_sk;

    -- PHASE 1: GET THE SOURCE BLUEPRINT
    SELECT msc.study_sk, msc.scenario_configuration_sk INTO v_source_study_sk, v_source_config_sk FROM public.map_scenario_configuration msc WHERE msc.scenario_sk = p_source_scenario_sk;
    
    SELECT * INTO v_source_study            FROM public.get_studies(jsonb_build_object('filter', jsonb_build_object('study_sk', v_source_study_sk)));
    SELECT * INTO v_source_scenario         FROM public.get_budget_scenarios(jsonb_build_object('filter', jsonb_build_object('scenario_sk', p_source_scenario_sk)));
    SELECT * INTO v_source_config           FROM public.get_scenario_configurations(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk)));
    SELECT jsonb_agg(t) INTO v_source_partners         FROM public.get_study_partners(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_sites            FROM public.get_sites(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_staff            FROM public.get_study_staff(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_arms             FROM public.get_study_arms(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_epochs           FROM public.get_study_epochs(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_visits           FROM public.get_study_visits(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_soa              FROM public.get_study_visit_activities(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_costs            FROM public.get_activity_costs(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_rules            FROM public.get_forecast_calculation_configs(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_enrollment_facts FROM public.get_fact_enrollments(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;
    SELECT jsonb_agg(t) INTO v_source_forecast_facts   FROM public.get_fact_forecast_details(jsonb_build_object('filter', jsonb_build_object('scenario_configuration_sk', v_source_config_sk))) t;

    -- PHASE 2: CREATE NEW PARENTS
    SELECT ar.action_report INTO v_temp_report FROM public.create_studies(jsonb_build_object('records', jsonb_build_array(v_source_study - 'study_sk' - 'study_bk' || jsonb_build_object('study_title', p_new_study_name)))) AS ar;
    v_new_study_sk := (v_temp_report->'summary'->0->>'study_sk')::bigint;
    v_new_study_bk := v_temp_report->'summary'->0->>'study_bk';
    
    SELECT ar.action_report INTO v_temp_report FROM public.create_budget_scenarios(jsonb_build_object('records', jsonb_build_array(v_source_scenario - 'scenario_sk' - 'scenario_bk' || jsonb_build_object('scenario_name', p_new_scenario_name)))) AS ar;
    v_new_scenario_sk := (v_temp_report->'summary'->0->>'scenario_sk')::bigint;
    v_new_scenario_bk := v_temp_report->'summary'->0->>'scenario_bk';
    
    SELECT ar.action_report INTO v_temp_report FROM public.create_scenario_configurations(jsonb_build_object('records', jsonb_build_array(v_source_config - 'scenario_configuration_sk' - 'scenario_configuration_bk' || jsonb_build_object('parent_study_bk', v_new_study_bk, 'parent_scenario_bk', v_new_scenario_bk)))) AS ar;
    v_new_config_sk := (v_temp_report->'summary'->0->>'scenario_configuration_sk')::bigint;
    v_new_config_bk := v_temp_report->'summary'->0->>'scenario_configuration_bk';

    -- PHASE 3: ITERATIVE "GET -> TRANSFORM -> CREATE" FOR BLUEPRINT CHILDREN
    -- THE DEFINITIVE FIX: Add guard clauses to all cloning steps.
    IF v_source_partners IS NOT NULL AND jsonb_array_length(v_source_partners) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_study_partners(jsonb_build_object('records', (SELECT jsonb_agg(p - 'study_partner_sk' - 'study_partner_bk' || jsonb_build_object('parent_scenario_configuration_bk', v_new_config_bk)) FROM jsonb_array_elements(v_source_partners) p))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_partner_bk_map FROM (SELECT (v_source_partners->(idx-1)->>'study_partner_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'study_partner_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_partners)) idx) map;
    END IF;
    
    IF v_source_sites IS NOT NULL AND jsonb_array_length(v_source_sites) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_sites(jsonb_build_object('records', (SELECT jsonb_agg(s - 'site_sk' - 'site_bk' || jsonb_build_object('parent_scenario_configuration_bk', v_new_config_bk, 'site_partner_bk', (v_partner_bk_map->>(s->>'site_partner_bk')))) FROM jsonb_array_elements(v_source_sites) s))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_site_bk_map FROM (SELECT (v_source_sites->(idx-1)->>'site_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'site_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_sites)) idx) map;
    END IF;
    
    IF v_source_staff IS NOT NULL AND jsonb_array_length(v_source_staff) > 0 THEN
        PERFORM public.create_study_staff(jsonb_build_object('records', (SELECT jsonb_agg(s - 'study_staff_sk' - 'study_staff_bk' || jsonb_build_object('parent_study_partner_bk', (v_partner_bk_map->>(s->>'study_partner_bk')))) FROM jsonb_array_elements(v_source_staff) s)));
    END IF;
    
    IF v_source_arms IS NOT NULL AND jsonb_array_length(v_source_arms) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_study_arms(jsonb_build_object('records', (SELECT jsonb_agg(a - 'arm_sk' - 'arm_bk' || jsonb_build_object('parent_scenario_configuration_bk', v_new_config_bk)) FROM jsonb_array_elements(v_source_arms) a))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_arm_bk_map FROM (SELECT (v_source_arms->(idx-1)->>'arm_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'arm_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_arms)) idx) map;
    END IF;
    
    IF v_source_epochs IS NOT NULL AND jsonb_array_length(v_source_epochs) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_study_epochs(jsonb_build_object('records', (SELECT jsonb_agg(e - 'epoch_sk' - 'epoch_bk' || jsonb_build_object('parent_arm_bk', (v_arm_bk_map->>(e->>'arm_bk')))) FROM jsonb_array_elements(v_source_epochs) e))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_epoch_bk_map FROM (SELECT (v_source_epochs->(idx-1)->>'epoch_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'epoch_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_epochs)) idx) map;
    END IF;
    
    IF v_source_visits IS NOT NULL AND jsonb_array_length(v_source_visits) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_study_visits(jsonb_build_object('records', (SELECT jsonb_agg(v - 'visit_sk' - 'visit_bk' || jsonb_build_object('parent_epoch_bk', (v_epoch_bk_map->>(v->>'epoch_bk')))) FROM jsonb_array_elements(v_source_visits) v))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_visit_bk_map FROM (SELECT (v_source_visits->(idx-1)->>'visit_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'visit_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_visits)) idx) map;
    END IF;
    
    IF v_source_soa IS NOT NULL AND jsonb_array_length(v_source_soa) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_study_visit_activities(jsonb_build_object('records', (SELECT jsonb_agg(s - 'map_soa_sk' - 'map_soa_bk' || jsonb_build_object('parent_scenario_configuration_bk', v_new_config_bk, 'visit_bk', (v_visit_bk_map->>(s->>'visit_bk')))) FROM jsonb_array_elements(v_source_soa) s))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_soa_bk_map FROM (SELECT (v_source_soa->(idx-1)->>'map_soa_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'map_soa_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_soa)) idx) map;
    END IF;
    
    IF v_source_costs IS NOT NULL AND jsonb_array_length(v_source_costs) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_activity_costs(jsonb_build_object('records', (SELECT jsonb_agg(c - 'activity_cost_sk' - 'activity_cost_bk' || jsonb_build_object('parent_scenario_configuration_bk', v_new_config_bk, 'cost_bearing_partner_bk', (v_partner_bk_map->>(c->>'cost_bearing_partner_bk')), 'payer_partner_bk', (v_partner_bk_map->>(c->>'payer_partner_bk')))) FROM jsonb_array_elements(v_source_costs) c))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_cost_bk_map FROM (SELECT (v_source_costs->(idx-1)->>'activity_cost_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'activity_cost_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_costs)) idx) map;
    END IF;
    
    IF v_source_rules IS NOT NULL AND jsonb_array_length(v_source_rules) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_forecast_calculation_configs(jsonb_build_object('records', (SELECT jsonb_agg(r - 'forecast_config_sk' - 'forecast_config_bk' || jsonb_build_object('parent_scenario_configuration_bk', v_new_config_bk)) FROM jsonb_array_elements(v_source_rules) r))) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_rule_bk_map FROM (SELECT (v_source_rules->(idx-1)->>'forecast_config_bk') as old_bk, (v_temp_report->'summary'->(idx-1)->>'forecast_config_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_rules)) idx) map;
    END IF;

    -- PHASE 4: TRANSFORM AND CREATE FACTS
    IF v_source_enrollment_facts IS NOT NULL AND jsonb_array_length(v_source_enrollment_facts) > 0 THEN
        SELECT ar.action_report INTO v_temp_report FROM public.create_fact_enrollments((SELECT jsonb_agg(e - 'enrollment_pk' - 'enrollment_bk' || jsonb_build_object('scenario_configuration_bk', v_new_config_bk, 'site_bk', (v_site_bk_map->>(e->>'site_bk')))) FROM jsonb_array_elements(v_source_enrollment_facts) e)) AS ar;
        SELECT jsonb_object_agg(old_bk, new_bk) INTO v_enrollment_bk_map FROM (SELECT (v_source_enrollment_facts->(idx-1)->>'enrollment_bk') as old_bk, (v_temp_report->'summary_created'->(idx-1)->>'enrollment_bk') as new_bk FROM generate_series(1, jsonb_array_length(v_source_enrollment_facts)) idx) map;
    END IF;
    
    IF v_source_forecast_facts IS NOT NULL AND jsonb_array_length(v_source_forecast_facts) > 0 THEN
        PERFORM public.create_fact_forecast_details((SELECT jsonb_agg(f - 'forecast_detail_pk' - 'forecast_detail_bk' || jsonb_build_object('scenario_configuration_bk', v_new_config_bk, 'site_bk', (v_site_bk_map->>(f->>'site_bk')), 'payer_partner_bk', (v_partner_bk_map->>(f->>'payer_partner_bk')), 'payee_partner_bk', (v_partner_bk_map->>(f->>'payee_partner_bk')), 'source_arm_bk', (v_arm_bk_map->>(f->>'source_arm_bk')), 'source_epoch_bk', (v_epoch_bk_map->>(f->>'source_epoch_bk')), 'source_visit_bk', (v_visit_bk_map->>(f->>'source_visit_bk')), 'source_enrollment_bk', (v_enrollment_bk_map->>(f->>'source_enrollment_bk')), 'source_activity_cost_bk', (v_cost_bk_map->>(f->>'source_activity_cost_bk')), 'source_map_soa_bk', (v_soa_bk_map->>(f->>'source_map_soa_bk')), 'source_forecast_config_bk', (v_rule_bk_map->>(f->>'source_forecast_config_bk')))) FROM jsonb_array_elements(v_source_forecast_facts) f));
    END IF;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', 'Scenario cloned successfully.',
        'new_study_sk', v_new_study_sk,
        'new_scenario_sk', v_new_scenario_sk,
        'new_scenario_configuration_sk', v_new_config_sk
    );
END;
$$;


ALTER FUNCTION "public"."clone_budget_scenario"("p_source_scenario_sk" bigint, "p_new_study_name" "text", "p_new_scenario_name" "text") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."clone_budget_scenario"("p_source_scenario_sk" bigint, "p_new_study_name" "text", "p_new_scenario_name" "text") IS 'The "Version Control" engine for the IDE. Why: Enables "branching" of a study plan for what-if analysis or to model the impact of a protocol amendment. How: Performs a deep copy of an entire scenario blueprint and its associated facts, creating a new, independent version.';





CREATE OR REPLACE FUNCTION "public"."create_budget_scenarios"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       public.create_budget_scenarios
*   Version:        3.1 (Gold Standard - BK-First Certified)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version is now fully compliant with the
*                   "BK-First" mandate. It is self-hydrating, accepting only
*                   the business key (`triggering_amendment_bk`) for the FK,
*                   and performs a secure, organization-scoped lookup.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_records_to_process JSONB := p_payload -> 'records';
    v_record_item JSONB;
    v_processed_count INT := 0;
    v_created_count INT := 0;
    v_skipped_count INT := 0;
    v_failed_count INT := 0;
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
    v_summary JSONB[] := ARRAY[]::JSONB[];
    v_existing_record RECORD;
    v_new_record RECORD;
    v_amendment_sk BIGINT;
    v_bk_to_check TEXT;
BEGIN
    IF v_user_sk IS NULL OR v_org_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;
    IF v_records_to_process IS NULL OR jsonb_typeof(v_records_to_process) != 'array' THEN RAISE EXCEPTION 'Payload must contain a "records" key.'; END IF;

    FOR v_record_item IN SELECT * FROM jsonb_array_elements(v_records_to_process) LOOP
        v_processed_count := v_processed_count + 1;
        v_amendment_sk := NULL;
        v_bk_to_check := v_record_item->>'scenario_bk';
        BEGIN
            IF v_bk_to_check IS NOT NULL THEN
                SELECT bs.scenario_sk INTO v_existing_record FROM public.dim_budget_scenario bs WHERE bs.scenario_bk = v_bk_to_check AND bs.organization_sk = v_org_sk;
            ELSE
                v_existing_record := NULL;
            END IF;

            IF FOUND THEN
                v_summary := array_append(v_summary, to_jsonb(v_existing_record));
                v_skipped_count := v_skipped_count + 1;
            ELSE
                -- THE DEFINITIVE FIX: Self-hydrate the amendment_sk from the provided BK.
                IF v_record_item ? 'triggering_amendment_bk' AND jsonb_typeof(v_record_item->'triggering_amendment_bk') != 'null' THEN
                    SELECT a.amendment_sk INTO v_amendment_sk
                    FROM public.dim_amendment a
                    WHERE a.amendment_bk = v_record_item->>'triggering_amendment_bk'
                      AND a.organization_sk = v_org_sk; -- Secure, organization-scoped lookup

                    IF NOT FOUND THEN
                        RAISE EXCEPTION 'Triggering amendment with BK % not found in your organization.', v_record_item->>'triggering_amendment_bk';
                    END IF;
                END IF;

                WITH inserted AS (
                    INSERT INTO public.dim_budget_scenario (organization_sk, scenario_bk, scenario_name, scenario_type, "version", description, scenario_status, triggering_amendment_sk, created_by_user_sk, updated_by_user_sk)
                    VALUES (v_org_sk, COALESCE(v_bk_to_check, 'CTF-SCEN-' || extensions.uuid_generate_v4()::text), v_record_item->>'scenario_name', (v_record_item->>'scenario_type')::public.budget_scenario_type_enum, COALESCE((v_record_item->>'version')::int, 1), v_record_item->>'description', COALESCE((v_record_item->>'scenario_status')::public.budget_scenario_status_enum, 'Draft'), v_amendment_sk, v_user_sk, v_user_sk)
                    RETURNING *
                )
                SELECT i.scenario_sk, i.scenario_bk, i.scenario_name, i.scenario_type, da.amendment_version_id as triggering_amendment_version, 0 as scenario_config_count
                INTO v_new_record
                FROM inserted i LEFT JOIN public.dim_amendment da ON i.triggering_amendment_sk = da.amendment_sk;

                v_summary := array_append(v_summary, to_jsonb(v_new_record));
                v_created_count := v_created_count + 1;
            END IF;
        EXCEPTION WHEN OTHERS THEN
            v_failed_count := v_failed_count + 1;
            v_error_messages := array_append(v_error_messages, SQLERRM);
        END;
    END LOOP;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', format('Processed %s records. Created: %s, Skipped: %s, Failed: %s.', v_processed_count, v_created_count, v_skipped_count, v_failed_count), 'created_count', v_created_count, 'skipped_count', v_skipped_count, 'failed_count', v_failed_count, 'summary', to_jsonb(v_summary), 'errors', to_jsonb(v_error_messages));
END;
$$;


ALTER FUNCTION "public"."create_budget_scenarios"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_budget_scenarios"("p_payload" "jsonb") IS 'Creates budget scenarios. Why: Enables the comparison of different plan types (e.g., Forecast vs. Budget vs. Baseline) for a study. How: Inserts scenario records with type, status, and version metadata.';


CREATE OR REPLACE FUNCTION "public"."delete_budget_scenarios"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       public.delete_budget_scenarios
*   Version:        2.0 (Gold Standard - Safe Hard Delete Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This definitive version implements the "Safe Hard Delete"
*                   pattern. It performs a pre-flight check to prevent the
*                   hard deletion of scenarios that are actively linked to
*                   scenario configurations, thus preserving referential integrity.
*                   It also returns a fully compliant "Action Report" envelope.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_is_admin BOOLEAN := public.is_current_user_admin();
    v_mode TEXT := COALESCE(lower(p_payload->>'mode'), 'soft');
    v_keys JSONB := p_payload->'keys';
    v_pks_to_process BIGINT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'scenario_pks')::bigint);
    v_bks_to_process TEXT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'scenario_bks'));
    v_sks_resolved_from_bks BIGINT[];
    v_final_sks_to_process BIGINT[];
    v_summary JSONB;
    v_action_taken_text TEXT;
    v_processed_count INT;
    v_keys_in_use BIGINT[];
    v_keys_to_action BIGINT[];
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
BEGIN
    IF v_user_sk IS NULL OR v_org_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;
    IF v_mode NOT IN ('soft', 'hard', 'restore') THEN RAISE EXCEPTION 'Invalid mode.'; END IF;
    IF v_mode = 'hard' AND NOT v_is_admin THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can perform a hard delete.'); RETURN; END IF;

    IF array_length(v_bks_to_process, 1) > 0 THEN
        SELECT array_agg(scenario_sk) INTO v_sks_resolved_from_bks FROM public.dim_budget_scenario WHERE scenario_bk = ANY(v_bks_to_process) AND organization_sk = v_org_sk;
    END IF;
    v_final_sks_to_process := array_cat(COALESCE(v_pks_to_process, ARRAY[]::BIGINT[]), COALESCE(v_sks_resolved_from_bks, ARRAY[]::BIGINT[]));
    v_final_sks_to_process := ARRAY(SELECT DISTINCT unnest(v_final_sks_to_process));

    IF array_length(v_final_sks_to_process, 1) IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', 'No matching records found to process.', 'processed_count', 0);
        RETURN;
    END IF;

    v_keys_to_action := v_final_sks_to_process;

    IF v_mode = 'hard' THEN
        SELECT array_agg(DISTINCT scenario_sk)
        INTO v_keys_in_use
        FROM public.map_scenario_configuration
        WHERE scenario_sk = ANY(v_final_sks_to_process) AND is_deleted = FALSE;

        v_keys_in_use := COALESCE(v_keys_in_use, ARRAY[]::BIGINT[]);
        IF array_length(v_keys_in_use, 1) > 0 THEN
            v_error_messages := array_append(v_error_messages, format('Cannot hard-delete %s scenarios because they are linked to active scenario configurations.', array_length(v_keys_in_use, 1)));
            SELECT array_agg(k) INTO v_keys_to_action FROM unnest(v_final_sks_to_process) k WHERE k <> ALL(v_keys_in_use);
        END IF;
    END IF;

    IF array_length(v_keys_to_action, 1) IS NULL THEN
         RETURN QUERY SELECT jsonb_build_object(
            'status', CASE WHEN array_length(v_keys_in_use, 1) > 0 THEN 'error' ELSE 'success' END,
            'message', 'No records could be processed. All selected records have dependencies.',
            'processed_count', 0,
            'failed_count', COALESCE(array_length(v_keys_in_use, 1), 0),
            'summary', '[]'::jsonb,
            'errors', to_jsonb(v_error_messages)
        );
        RETURN;
    END IF;

    IF v_mode = 'hard' THEN
        v_action_taken_text := 'hard-deleted';
        WITH deleted_rows AS (DELETE FROM public.dim_budget_scenario WHERE scenario_sk = ANY(v_keys_to_action) RETURNING scenario_sk, scenario_bk, scenario_name)
        SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM deleted_rows t;
    ELSE
        DECLARE v_set_deleted_status BOOLEAN := (v_mode = 'soft');
        BEGIN
            v_action_taken_text := CASE WHEN v_set_deleted_status THEN 'soft-deleted' ELSE 'restored' END;
            WITH updated_rows AS (UPDATE public.dim_budget_scenario SET is_deleted = v_set_deleted_status, updated_by_user_sk = v_user_sk, updated_at = NOW() WHERE scenario_sk = ANY(v_final_sks_to_process) RETURNING scenario_sk, scenario_bk, scenario_name, is_deleted)
            SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM updated_rows t;
        END;
    END IF;

    RETURN QUERY SELECT jsonb_build_object(
        'status', CASE WHEN array_length(v_keys_in_use, 1) > 0 THEN 'partial_success' ELSE 'success' END,
        'message', format('%s records %s. %s records failed due to dependencies.', COALESCE(v_processed_count, 0), v_action_taken_text, COALESCE(array_length(v_keys_in_use, 1), 0)),
        'processed_count', COALESCE(v_processed_count, 0),
        'failed_count', COALESCE(array_length(v_keys_in_use, 1), 0),
        'summary', COALESCE(v_summary, '[]'::jsonb),
        'errors', to_jsonb(v_error_messages)
    );
END;
$$;


ALTER FUNCTION "public"."delete_budget_scenarios"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."delete_budget_scenarios"("p_payload" "jsonb") IS 'Archives, restores, or purges scenarios. Why: Manages the lifecycle of different planning versions. How: Implements the "Safe Delete" pattern with dependency checks on child configurations.';


CREATE OR REPLACE FUNCTION "public"."get_budget_scenarios"("p_payload" "jsonb" DEFAULT '{}'::"jsonb") RETURNS SETOF "jsonb"
    LANGUAGE "plpgsql" STABLE
    AS $$
/********************************************************************************
*   Function:       public.get_budget_scenarios
*   Version:        7.2 (Gold Standard - Multi-Modal Deletion Certified)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version correctly implements the Multi-Modal
*                   Output Mandate for soft-deleted records. The `is_deleted`
*                   filter is now conditionally applied ONLY when output_mode is
*                   'list' and `include_deleted` is false. All other modes
*                   (e.g., 'full') correctly return all records.
********************************************************************************/
DECLARE
    v_output_mode TEXT              := COALESCE(p_payload ->> 'output_mode', 'full');
    v_filter JSONB                  := COALESCE(p_payload -> 'filter', '{}'::jsonb);
    v_include_deleted BOOLEAN       := COALESCE((v_filter ->> 'include_deleted')::boolean, false);
    v_record RECORD;
BEGIN
    FOR v_record IN
        WITH base_query AS (
            SELECT
                dbs.scenario_sk, dbs.scenario_bk, dbs.scenario_name, dbs.scenario_type,
                dbs.version, dbs.scenario_status, dbs.is_locked, dbs.description, dbs.is_deleted,
                dbs.triggering_amendment_sk, da.amendment_version_id,
                creator.user_name AS created_by_user_name,
                updater.user_name AS updated_by_user_name,
                COALESCE((
                    SELECT count(*) FROM public.map_scenario_configuration msc
                    WHERE msc.scenario_sk = dbs.scenario_sk AND (v_output_mode <> 'list' OR v_include_deleted OR msc.is_deleted = FALSE)
                ), 0)::bigint AS scenario_configuration_count,
                (SELECT COALESCE(SUM(ffd.net_cost), 0.00) FROM public.fact_forecast_detail ffd
                 WHERE ffd.scenario_sk = dbs.scenario_sk AND (v_output_mode <> 'list' OR v_include_deleted OR ffd.is_deleted = FALSE)
                ) AS total_forecast_cost,
                dbs.created_at, dbs.updated_at, dbs.created_by_user_sk, dbs.updated_by_user_sk
            FROM public.dim_budget_scenario dbs
            LEFT JOIN public.dim_amendment da ON dbs.triggering_amendment_sk = da.amendment_sk
            LEFT JOIN public.dim_user creator ON dbs.created_by_user_sk = creator.user_sk
            LEFT JOIN public.dim_user updater ON dbs.updated_by_user_sk = updater.user_sk
            WHERE
                -- THE DEFINITIVE FIX: Conditionally apply deletion filter ONLY for 'list' mode.
                (v_output_mode <> 'list' OR v_include_deleted OR dbs.is_deleted = FALSE)
                AND (NOT(v_filter ? 'scenario_sk') OR dbs.scenario_sk = (v_filter->>'scenario_sk')::bigint)
                AND (NOT(v_filter ? 'scenario_bk') OR dbs.scenario_bk = (v_filter->>'scenario_bk'))
                AND (NOT(v_filter ? 'scenario_name') OR dbs.scenario_name ILIKE (
                    CASE WHEN v_output_mode = 'single_record' THEN (v_filter->>'scenario_name') ELSE '%' || (v_filter->>'scenario_name') || '%' END
                ))
                AND (NOT(v_filter ? 'scenario_type') OR dbs.scenario_type::text = (v_filter->>'scenario_type'))
                AND (NOT(v_filter ? 'scenario_status') OR dbs.scenario_status::text = (v_filter->>'scenario_status'))
        )
        SELECT * FROM base_query
        ORDER BY scenario_name, version
        LIMIT CASE WHEN v_output_mode = 'single_record' THEN 1 ELSE NULL END
    LOOP
        IF v_output_mode = 'list' THEN
            RETURN NEXT jsonb_build_object(
                'scenario_sk', v_record.scenario_sk,
                'scenario_bk', v_record.scenario_bk,
                'scenario_name', v_record.scenario_name,
                'scenario_type', v_record.scenario_type,
                'version', v_record.version,
                'scenario_status', v_record.scenario_status,
                'is_locked', v_record.is_locked,
                'is_deleted', v_record.is_deleted,
                'amendment_version_id', v_record.amendment_version_id,
                'scenario_configuration_count', v_record.scenario_configuration_count,
                'total_forecast_cost', v_record.total_forecast_cost
            );
        ELSE -- 'full' or 'single_record' mode
            RETURN NEXT to_jsonb(v_record);
        END IF;
    END LOOP;
END;
$$;


ALTER FUNCTION "public"."get_budget_scenarios"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_budget_scenarios"("p_payload" "jsonb") IS 'Retrieves budget scenarios with aggregated child counts and financial totals. Why: The primary interface for selecting and reporting on different planning versions. How: Returns a multi-modal JSONB payload with counts of linked configurations and the total forecasted cost.';


CREATE OR REPLACE FUNCTION "public"."update_budget_scenarios"("p_update" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.update_budget_scenarios
*   Version:        5.0 (Gold Standard - True Partial Update Certified)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version fixes a critical NOT NULL constraint
*                   violation bug by implementing the mandated `CASE WHEN ? 'key'`
*                   pattern for all updatable fields. The function now correctly
*                   preserves existing values for fields not included in the
*                   update payload, enabling true partial updates as intended by
*                   the "Rich Audit" pattern.
********************************************************************************/
DECLARE
    v_scenario_sk BIGINT := (p_update->>'budget_scenario_sk')::bigint;
    v_update_data JSONB := p_update->'update_fields';
    v_updating_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_is_admin BOOLEAN := public.is_current_user_admin();
    v_old_record public.dim_budget_scenario;
    v_new_record public.dim_budget_scenario;
    v_changes jsonb := '{}'::jsonb;
    v_new_amendment_sk BIGINT;
    v_human_readable_message TEXT;
    v_creator_name TEXT;
    v_updater_name TEXT;
    v_days_since_creation INT;
BEGIN
    SELECT * INTO v_old_record FROM public.dim_budget_scenario WHERE scenario_sk = v_scenario_sk FOR UPDATE;
    IF NOT FOUND THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Scenario not found or permission denied.'); RETURN; END IF;
    IF v_old_record.is_locked THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Update failed. Scenario is locked.'); RETURN; END IF;
    IF v_update_data ? 'scenario_status' AND NOT v_is_admin THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can change the scenario status.'); RETURN; END IF;

    -- Self-Hydration for Triggering Amendment
    v_new_amendment_sk := v_old_record.triggering_amendment_sk;
    IF v_update_data ? 'triggering_amendment_bk' THEN
        IF v_update_data->>'triggering_amendment_bk' IS NULL OR jsonb_typeof(v_update_data->'triggering_amendment_bk') = 'null' THEN
            v_new_amendment_sk := NULL;
        ELSE
            SELECT amendment_sk INTO v_new_amendment_sk
            FROM public.dim_amendment
            WHERE amendment_bk = v_update_data->>'triggering_amendment_bk' AND organization_sk = v_org_sk;
            IF NOT FOUND THEN RAISE EXCEPTION 'Triggering amendment with BK % not found.', v_update_data->>'triggering_amendment_bk'; END IF;
        END IF;
    END IF;

    WITH updated_row AS (
        UPDATE public.dim_budget_scenario SET
            -- THE DEFINITIVE FIX: Use the CASE WHEN ? 'key' pattern for all fields.
            scenario_bk             = CASE WHEN v_update_data ? 'scenario_bk' AND jsonb_typeof(v_update_data->'scenario_bk') = 'null' THEN 'CTF-SCEN-' || extensions.uuid_generate_v4()::text WHEN v_update_data ? 'scenario_bk' THEN v_update_data->>'scenario_bk' ELSE scenario_bk END,
            scenario_name           = CASE WHEN v_update_data ? 'scenario_name'           THEN v_update_data->>'scenario_name'           ELSE scenario_name END,
            scenario_type           = CASE WHEN v_update_data ? 'scenario_type'           THEN (v_update_data->>'scenario_type')::public.budget_scenario_type_enum ELSE scenario_type END,
            triggering_amendment_sk = CASE WHEN v_update_data ? 'triggering_amendment_bk' THEN v_new_amendment_sk ELSE triggering_amendment_sk END,
            scenario_status         = CASE WHEN v_update_data ? 'scenario_status'         THEN (v_update_data->>'scenario_status')::public.budget_scenario_status_enum ELSE scenario_status END,
            description             = CASE WHEN v_update_data ? 'description' AND jsonb_typeof(v_update_data->'description') = 'null' THEN NULL WHEN v_update_data ? 'description' THEN v_update_data->>'description' ELSE description END,
            updated_by_user_sk      = v_updating_user_sk,
            updated_at              = NOW()
        WHERE scenario_sk = v_scenario_sk AND is_locked = FALSE
        RETURNING *
    )
    SELECT * INTO v_new_record FROM updated_row;

    -- Change log generation remains the same.
    IF v_new_record.scenario_bk IS DISTINCT FROM v_old_record.scenario_bk THEN v_changes := v_changes || jsonb_build_object('scenario_bk', jsonb_build_object('old', v_old_record.scenario_bk, 'new', v_new_record.scenario_bk)); END IF;
    IF v_new_record.scenario_name IS DISTINCT FROM v_old_record.scenario_name THEN v_changes := v_changes || jsonb_build_object('scenario_name', jsonb_build_object('old', v_old_record.scenario_name, 'new', v_new_record.scenario_name)); END IF;
    IF v_new_record.scenario_type IS DISTINCT FROM v_old_record.scenario_type THEN v_changes := v_changes || jsonb_build_object('scenario_type', jsonb_build_object('old', v_old_record.scenario_type, 'new', v_new_record.scenario_type)); END IF;
    IF v_new_record.triggering_amendment_sk IS DISTINCT FROM v_old_record.triggering_amendment_sk THEN v_changes := v_changes || jsonb_build_object('triggering_amendment_sk', jsonb_build_object('old', v_old_record.triggering_amendment_sk, 'new', v_new_record.triggering_amendment_sk)); END IF;
    IF v_new_record.scenario_status IS DISTINCT FROM v_old_record.scenario_status THEN v_changes := v_changes || jsonb_build_object('scenario_status', jsonb_build_object('old', v_old_record.scenario_status, 'new', v_new_record.scenario_status)); END IF;
    IF v_new_record.description IS DISTINCT FROM v_old_record.description THEN v_changes := v_changes || jsonb_build_object('description', jsonb_build_object('old', v_old_record.description, 'new', v_new_record.description)); END IF;

    -- Human-readable message generation remains the same.
    SELECT u.user_name INTO v_creator_name FROM public.dim_user u WHERE u.user_sk = v_old_record.created_by_user_sk;
    SELECT u.user_name INTO v_updater_name FROM public.dim_user u WHERE u.user_sk = v_updating_user_sk;
    v_days_since_creation := DATE_PART('day', v_new_record.updated_at - v_old_record.created_at);
    v_human_readable_message := format('Scenario "%s" updated by %s. This record was created %s days ago by %s.', v_new_record.scenario_name, COALESCE(v_updater_name, 'Unknown User'), v_days_since_creation, COALESCE(v_creator_name, 'Unknown User'));

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', v_human_readable_message, 'updated_sk', v_new_record.scenario_sk, 'record', row_to_json(v_new_record), 'changes', v_changes);
END;
$$;


ALTER FUNCTION "public"."update_budget_scenarios"("p_update" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."update_budget_scenarios"("p_update" "jsonb") IS 'Updates a single budget scenario. Why: Allows for adjustment of scenario metadata and status. How: A partial update function that implements the "Rich Audit" pattern.';


CREATE OR REPLACE FUNCTION "public"."check_scenario_lock_status"("p_scenario_sk" bigint DEFAULT NULL::bigint, "p_scenario_configuration_sk" bigint DEFAULT NULL::bigint) RETURNS boolean
    LANGUAGE "plpgsql" STABLE
    SET "search_path" TO 'public'
    AS $$
/********************************************************************************
*   Function:       public.check_scenario_lock_status
*   Version:        2.0 (Gold Standard - Universal Input)
*   Author:         AI Database Architect
*   Description:    Checks the lock status of a scenario. This definitive version
*                   is more robust and user-friendly, accepting either a scenario_sk
*                   or a scenario_configuration_sk as input. It runs as the
*                   invoker, relying on RLS on the underlying tables for security.
********************************************************************************/
DECLARE
  v_is_locked BOOLEAN;
BEGIN
    -- Input validation
    IF p_scenario_sk IS NULL AND p_scenario_configuration_sk IS NULL THEN
        RAISE EXCEPTION 'Either p_scenario_sk or p_scenario_configuration_sk must be provided.';
    END IF;

    IF p_scenario_sk IS NOT NULL THEN
        -- Pathway 1: Direct lookup by scenario_sk
        SELECT COALESCE(dbs.is_locked, false)
        INTO v_is_locked
        FROM public.dim_budget_scenario dbs
        WHERE dbs.scenario_sk = p_scenario_sk AND dbs.is_deleted = FALSE;

    ELSE
        -- Pathway 2: Lookup via scenario_configuration_sk
        SELECT COALESCE(dbs.is_locked, false)
        INTO v_is_locked
        FROM public.dim_budget_scenario dbs
        JOIN public.map_scenario_configuration msc ON dbs.scenario_sk = msc.scenario_sk
        WHERE msc.scenario_configuration_sk = p_scenario_configuration_sk AND dbs.is_deleted = FALSE;
    END IF;

    -- The RLS policy on the tables will cause this to return NULL if the user
    -- lacks permission, triggering the NOT FOUND condition.
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Record not found or permission denied for the provided key.';
    END IF;

  RETURN v_is_locked;
END;
$$;


ALTER FUNCTION "public"."check_scenario_lock_status"("p_scenario_sk" bigint, "p_scenario_configuration_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."check_scenario_lock_status"("p_scenario_sk" bigint, "p_scenario_configuration_sk" bigint) IS 'Checks if a scenario is locked and therefore immutable. Why: A critical business rule check used by API functions to prevent modifications to approved or finalized plans. How: Resolves the scenario from either a `scenario_sk` or `scenario_configuration_sk` and returns its `is_locked` status.';


CREATE OR REPLACE FUNCTION "public"."get_next_scenario_version"("p_study_sk" bigint, "p_scenario_type" "public"."budget_scenario_type_enum") RETURNS integer
    LANGUAGE "plpgsql" STABLE
    AS $$
DECLARE
    v_next_version INTEGER;
BEGIN
    SELECT COALESCE(MAX(version), 0) + 1
    INTO v_next_version
    FROM public.dim_budget_scenario
    WHERE study_sk = p_study_sk
      -- <<< THE FIX: The cast is no longer needed as the types already match.
      AND scenario_type = p_scenario_type
      AND is_deleted = FALSE;

    RETURN COALESCE(v_next_version, 1);
END;
$$;


ALTER FUNCTION "public"."get_next_scenario_version"("p_study_sk" bigint, "p_scenario_type" "public"."budget_scenario_type_enum") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_next_scenario_version"("p_study_sk" bigint, "p_scenario_type" "public"."budget_scenario_type_enum") IS 'Calculates the next available version number for a new scenario. Why: Automates versioning to prevent duplicates and maintain a logical sequence. How: Finds the MAX version for a given study and scenario type and adds 1.';





CREATE OR REPLACE FUNCTION "public"."toggle_budget_scenario_lock"("p_budget_scenario_sk" bigint, "p_is_locked" boolean) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       public.toggle_budget_scenario_lock
*   Version:        3.0 (Gold Standard - RBAC Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This definitive version fixes a critical RBAC flaw. It now
*                   correctly checks if the calling user has the 'Admin' role
*                   before allowing a lock or unlock operation.
********************************************************************************/
DECLARE
    v_old_record public.dim_budget_scenario;
    v_new_record public.dim_budget_scenario;
BEGIN
    -- THE DEFINITIVE RBAC FIX: Check for Admin role at the beginning.
    IF NOT public.is_current_user_admin() THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can lock or unlock a scenario.');
        RETURN;
    END IF;

    SELECT * INTO v_old_record FROM public.dim_budget_scenario WHERE scenario_sk = p_budget_scenario_sk;
    IF NOT FOUND THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', format('Scenario SK %s not found or permission denied.', p_budget_scenario_sk));
        RETURN;
    END IF;

    -- This check remains for the special case of unlocking a finalized scenario.
    IF v_old_record.is_locked AND v_old_record.scenario_status = 'Approved' AND p_is_locked = FALSE AND NOT public.is_current_user_admin() THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can unlock a finalized (Approved) scenario.');
        RETURN;
    END IF;

    WITH updated_row AS (
        UPDATE public.dim_budget_scenario SET
            is_locked = p_is_locked,
            updated_by_user_sk = public.get_current_user_sk(), updated_at = NOW()
        WHERE scenario_sk = p_budget_scenario_sk
        RETURNING *
    )
    SELECT * INTO v_new_record FROM updated_row;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', format('Scenario lock status for "%s" set to %s.', v_new_record.scenario_name, v_new_record.is_locked), 'updated_sk', v_new_record.scenario_sk, 'record', row_to_json(v_new_record));
END;
$$;


ALTER FUNCTION "public"."toggle_budget_scenario_lock"("p_budget_scenario_sk" bigint, "p_is_locked" boolean) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."toggle_budget_scenario_lock"("p_budget_scenario_sk" bigint, "p_is_locked" boolean) IS 'Locks or unlocks a scenario. Why: Controls the mutability of a plan during the approval process. How: Flips the `is_locked` flag and provides a detailed audit report of the change.';






CREATE OR REPLACE FUNCTION "public"."finalize_budget_scenario"("p_budget_scenario_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       public.finalize_budget_scenario
*   Version:        2.1 (Gold Standard - Corrected Column Name)
*   Author:         AI Database Architect
*   Description:    This version fixes a critical bug by using the correct
*                   column name 'scenario_sk' instead of 'budget_scenario_sk'.
********************************************************************************/
DECLARE
    v_new_record public.dim_budget_scenario;
BEGIN
    IF NOT public.is_current_user_admin() THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can finalize a scenario.');
        RETURN;
    END IF;

    WITH updated_row AS (
        UPDATE public.dim_budget_scenario SET
            scenario_status = 'Approved', is_locked = TRUE,
            updated_by_user_sk = public.get_current_user_sk(), updated_at = NOW()
        -- THE DEFINITIVE FIX: Use the correct column name 'scenario_sk'
        WHERE scenario_sk = p_budget_scenario_sk
        RETURNING *
    )
    SELECT * INTO v_new_record FROM updated_row;

    IF NOT FOUND THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', format('Scenario SK %s not found or permission denied.', p_budget_scenario_sk));
        RETURN;
    END IF;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', format('Scenario "%s" finalized, set to Approved, and locked.', v_new_record.scenario_name), 'updated_sk', v_new_record.scenario_sk, 'record', row_to_json(v_new_record));
END;
$$;


ALTER FUNCTION "public"."finalize_budget_scenario"("p_budget_scenario_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."finalize_budget_scenario"("p_budget_scenario_sk" bigint) IS 'Finalizes (approves) and locks a scenario. Why: Establishes an immutable baseline plan against which forecasts and actuals can be compared. How: An admin-only function that atomically sets the scenario status to "Approved" and the `is_locked` flag to TRUE.';




CREATE OR REPLACE FUNCTION "public"."create_scenario_configurations"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_scenario_configurations
*   Version:        2.0 (Gold Standard - Corrected Idempotency)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version corrects the idempotency logic. It
*                   now checks for existing records based on the natural key
*                   (study_sk, scenario_sk) in addition to the business key,
*                   preventing unique constraint violations.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_records_to_process JSONB := p_payload -> 'records';
    v_record_item JSONB;
    v_processed_count INT := 0;
    v_created_count INT := 0;
    v_skipped_count INT := 0;
    v_failed_count INT := 0;
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
    v_summary JSONB[] := ARRAY[]::JSONB[];
    v_existing_record RECORD;
    v_new_record RECORD;
    v_study_sk BIGINT;
    v_scenario_sk BIGINT;
    v_bk_to_check TEXT;
BEGIN
    IF v_user_sk IS NULL OR v_org_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;
    IF v_records_to_process IS NULL OR jsonb_typeof(v_records_to_process) != 'array' THEN RAISE EXCEPTION 'Payload must contain a "records" key.'; END IF;

    FOR v_record_item IN SELECT * FROM jsonb_array_elements(v_records_to_process) LOOP
        v_processed_count := v_processed_count + 1;
        v_study_sk := NULL; v_scenario_sk := NULL;
        v_bk_to_check := v_record_item->>'scenario_configuration_bk';
        BEGIN
            -- Hydration
            SELECT s.study_sk INTO v_study_sk FROM public.dim_study s WHERE s.study_bk = v_record_item->>'parent_study_bk' AND s.organization_sk = v_org_sk;
            SELECT bs.scenario_sk INTO v_scenario_sk FROM public.dim_budget_scenario bs WHERE bs.scenario_bk = v_record_item->>'parent_scenario_bk' AND bs.organization_sk = v_org_sk;
            IF v_study_sk IS NULL OR v_scenario_sk IS NULL THEN RAISE EXCEPTION 'A required parent entity (study or scenario) could not be resolved.'; END IF;

            -- <<<< THE FIX: Comprehensive Idempotency Check >>>>
            SELECT msc.scenario_configuration_sk, msc.scenario_configuration_bk
            INTO v_existing_record
            FROM public.map_scenario_configuration msc
            WHERE (msc.scenario_configuration_bk = v_bk_to_check AND msc.organization_sk = v_org_sk)
               OR (msc.study_sk = v_study_sk AND msc.scenario_sk = v_scenario_sk);

            IF FOUND THEN
                v_summary := array_append(v_summary, to_jsonb(v_existing_record));
                v_skipped_count := v_skipped_count + 1;
            ELSE
                WITH inserted AS (
                    INSERT INTO public.map_scenario_configuration (scenario_configuration_bk, study_sk, scenario_sk, start_date, end_date, target_enrollment, target_sites, study_status, created_by_user_sk, updated_by_user_sk)
                    VALUES (COALESCE(v_bk_to_check, 'CTF-SCFG-' || extensions.uuid_generate_v4()::text), v_study_sk, v_scenario_sk, (v_record_item->>'start_date')::date, (v_record_item->>'end_date')::date, (v_record_item->>'target_enrollment')::int, (v_record_item->>'target_sites')::int, (v_record_item->>'study_status')::public.study_status_enum, v_user_sk, v_user_sk)
                    RETURNING *
                )
                SELECT i.scenario_configuration_sk, i.scenario_configuration_bk, s.study_short_name, bs.scenario_name, 0 as arm_count, 0 as site_count
                INTO v_new_record
                FROM inserted i JOIN public.dim_study s ON i.study_sk = s.study_sk JOIN public.dim_budget_scenario bs ON i.scenario_sk = bs.scenario_sk;

                v_summary := array_append(v_summary, to_jsonb(v_new_record));
                v_created_count := v_created_count + 1;
            END IF;
        EXCEPTION WHEN OTHERS THEN
            v_failed_count := v_failed_count + 1;
            v_error_messages := array_append(v_error_messages, SQLERRM);
        END;
    END LOOP;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', format('Processed %s records. Created: %s, Skipped: %s, Failed: %s.', v_processed_count, v_created_count, v_skipped_count, v_failed_count), 'created_count', v_created_count, 'skipped_count', v_skipped_count, 'failed_count', v_failed_count, 'summary', to_jsonb(v_summary), 'errors', to_jsonb(v_error_messages));
END;
$$;


ALTER FUNCTION "public"."create_scenario_configurations"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_scenario_configurations"("p_payload" "jsonb") IS 'Creates Scenario Configurations, which are the root of a study plan. Why: Links a study to a scenario, creating a concrete, versioned plan to which all other blueprint components (arms, sites, costs) are attached. How: Validates parent study/scenario and inserts the linking record.';


CREATE OR REPLACE FUNCTION "public"."get_scenario_configurations"("p_payload" "jsonb" DEFAULT '{}'::"jsonb") RETURNS SETOF "jsonb"
    LANGUAGE "plpgsql" STABLE
    AS $$
/********************************************************************************
*   Function:       public.get_scenario_configurations
*   Version:        7.3 (Gold Standard - Multi-Modal Deletion Certified)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version correctly implements the Multi-Modal
*                   Output Mandate for soft-deleted records. The `is_deleted`
*                   filter is now conditionally applied ONLY when output_mode is
*                   'list' and `include_deleted` is false.
********************************************************************************/
DECLARE
    v_output_mode TEXT              := COALESCE(p_payload ->> 'output_mode', 'full');
    v_filter JSONB                  := COALESCE(p_payload -> 'filter', '{}'::jsonb);
    v_include_deleted BOOLEAN       := COALESCE((v_filter ->> 'include_deleted')::boolean, false);
    v_record RECORD;
BEGIN
    FOR v_record IN
        WITH base_query AS (
            SELECT
                msc.scenario_configuration_sk, msc.scenario_configuration_bk,
                msc.start_date, msc.end_date, msc.target_enrollment, msc.target_sites,
                msc.study_status, msc.is_deleted,
                msc.study_sk, ds.protocol_number AS study_name,
                msc.scenario_sk, dbs.scenario_name,
                creator.user_name AS created_by_user_name,
                updater.user_name AS updated_by_user_name,
                COALESCE((SELECT count(*) FROM public.dim_study_arm dsa WHERE dsa.scenario_configuration_sk = msc.scenario_configuration_sk AND dsa.organization_sk = msc.organization_sk AND (v_output_mode <> 'list' OR v_include_deleted OR dsa.is_deleted = FALSE)), 0)::bigint AS arm_count,
                COALESCE((SELECT count(*) FROM public.dim_site dsite WHERE dsite.scenario_configuration_sk = msc.scenario_configuration_sk AND dsite.organization_sk = msc.organization_sk AND (v_output_mode <> 'list' OR v_include_deleted OR dsite.is_deleted = FALSE)), 0)::bigint AS site_count,
                COALESCE((SELECT count(*) FROM public.map_study_partners msp WHERE msp.scenario_configuration_sk = msc.scenario_configuration_sk AND msp.organization_sk = msc.organization_sk AND (v_output_mode <> 'list' OR v_include_deleted OR msp.is_deleted = FALSE)), 0)::bigint AS partner_count,
                msc.created_at, msc.updated_at, msc.created_by_user_sk, msc.updated_by_user_sk
            FROM public.map_scenario_configuration msc
            INNER JOIN public.dim_study ds ON msc.study_sk = ds.study_sk
            INNER JOIN public.dim_budget_scenario dbs ON msc.scenario_sk = dbs.scenario_sk
            LEFT JOIN public.dim_user creator ON msc.created_by_user_sk = creator.user_sk
            LEFT JOIN public.dim_user updater ON msc.updated_by_user_sk = updater.user_sk
            WHERE
                -- THE DEFINITIVE FIX: Conditionally apply deletion filter ONLY for 'list' mode.
                (v_output_mode <> 'list' OR v_include_deleted OR msc.is_deleted = FALSE)
                AND (NOT(v_filter ? 'scenario_configuration_sk') OR msc.scenario_configuration_sk = (v_filter->>'scenario_configuration_sk')::bigint)
                AND (NOT(v_filter ? 'scenario_configuration_bk') OR msc.scenario_configuration_bk = (v_filter->>'scenario_configuration_bk'))
                AND (NOT(v_filter ? 'study_sk') OR msc.study_sk = (v_filter->>'study_sk')::bigint)
                AND (NOT(v_filter ? 'scenario_sk') OR msc.scenario_sk = (v_filter->>'scenario_sk')::bigint)
        )
        SELECT * FROM base_query
        ORDER BY study_name, scenario_name
        LIMIT CASE WHEN v_output_mode = 'single_record' THEN 1 ELSE NULL END
    LOOP
        IF v_output_mode = 'list' THEN
            RETURN NEXT jsonb_build_object(
                'scenario_configuration_sk', v_record.scenario_configuration_sk,
                'scenario_configuration_bk', v_record.scenario_configuration_bk,
                'study_name', v_record.study_name,
                'scenario_name', v_record.scenario_name,
                'study_status', v_record.study_status,
                'target_enrollment', v_record.target_enrollment,
                'target_sites', v_record.target_sites,
                'arm_count', v_record.arm_count,
                'site_count', v_record.site_count,
                'partner_count', v_record.partner_count,
                'is_deleted', v_record.is_deleted
            );
        ELSE
            RETURN NEXT to_jsonb(v_record);
        END IF;
    END LOOP;
END;
$$;


ALTER FUNCTION "public"."get_scenario_configurations"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_scenario_configurations"("p_payload" "jsonb") IS 'Retrieves Scenario Configurations with hierarchical context. Why: The primary interface for navigating the different plans (study-scenario combinations) in the workspace. How: Returns a multi-modal JSONB payload with aggregated counts of child objects (arms, sites, partners).';



CREATE OR REPLACE FUNCTION "public"."delete_scenario_configurations"("p_payload" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       public.delete_scenario_configurations
*   Version:        3.0 (Gold Standard - Lock Enforcement Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This definitive version fixes a critical security flaw by
*                   adding a mandatory pre-flight check for the parent
*                   scenario's lock status before allowing any delete or restore
*                   operation, fully complying with the mandate.
********************************************************************************/
DECLARE
    v_user_sk BIGINT := public.get_current_user_sk();
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_is_admin BOOLEAN := public.is_current_user_admin();
    v_mode TEXT := COALESCE(lower(p_payload->>'mode'), 'soft');
    v_keys JSONB := p_payload->'keys';
    v_pks_to_process BIGINT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'scenario_configuration_pks')::bigint);
    v_bks_to_process TEXT[] := ARRAY(SELECT jsonb_array_elements_text(v_keys->'scenario_configuration_bks'));
    v_sks_resolved_from_bks BIGINT[];
    v_final_sks_to_process BIGINT[];
    v_summary JSONB;
    v_action_taken_text TEXT;
    v_processed_count INT;
    v_keys_in_use BIGINT[];
    v_keys_to_action BIGINT[];
    v_error_messages TEXT[] := ARRAY[]::TEXT[];
    v_locked_scenarios BIGINT[];
BEGIN
    IF v_user_sk IS NULL OR v_org_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;
    IF v_mode NOT IN ('soft', 'hard', 'restore') THEN RAISE EXCEPTION 'Invalid mode.'; END IF;
    IF v_mode = 'hard' AND NOT v_is_admin THEN RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Permission denied. Only Admins can perform a hard delete.'); RETURN; END IF;

    IF array_length(v_bks_to_process, 1) > 0 THEN
        SELECT array_agg(scenario_configuration_sk) INTO v_sks_resolved_from_bks FROM public.map_scenario_configuration WHERE scenario_configuration_bk = ANY(v_bks_to_process) AND organization_sk = v_org_sk;
    END IF;
    v_final_sks_to_process := array_cat(COALESCE(v_pks_to_process, ARRAY[]::BIGINT[]), COALESCE(v_sks_resolved_from_bks, ARRAY[]::BIGINT[]));
    v_final_sks_to_process := ARRAY(SELECT DISTINCT unnest(v_final_sks_to_process));

    IF array_length(v_final_sks_to_process, 1) IS NULL THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', 'No matching records found to process.', 'processed_count', 0);
        RETURN;
    END IF;

    -- THE DEFINITIVE FIX: Pre-flight lock check for ALL modes.
    SELECT array_agg(msc.scenario_configuration_sk)
    INTO v_locked_scenarios
    FROM public.map_scenario_configuration msc
    JOIN public.dim_budget_scenario dbs ON msc.scenario_sk = dbs.scenario_sk
    WHERE msc.scenario_configuration_sk = ANY(v_final_sks_to_process) AND dbs.is_locked = TRUE;

    IF array_length(v_locked_scenarios, 1) > 0 THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Operation failed. The parent scenario is locked.');
        RETURN;
    END IF;

    v_keys_to_action := v_final_sks_to_process;

    IF v_mode = 'hard' THEN
        SELECT array_agg(DISTINCT scenario_configuration_sk) INTO v_keys_in_use FROM (
            SELECT scenario_configuration_sk FROM public.dim_study_arm WHERE scenario_configuration_sk = ANY(v_final_sks_to_process) AND is_deleted = FALSE
            UNION ALL
            SELECT scenario_configuration_sk FROM public.dim_site WHERE scenario_configuration_sk = ANY(v_final_sks_to_process) AND is_deleted = FALSE
            UNION ALL
            SELECT scenario_configuration_sk FROM public.map_study_partners WHERE scenario_configuration_sk = ANY(v_final_sks_to_process) AND is_deleted = FALSE
            UNION ALL
            SELECT scenario_configuration_sk FROM public.dim_activity_cost WHERE scenario_configuration_sk = ANY(v_final_sks_to_process) AND is_deleted = FALSE
            UNION ALL
            SELECT scenario_configuration_sk FROM public.dim_forecast_calculation_config WHERE scenario_configuration_sk = ANY(v_final_sks_to_process) AND is_deleted = FALSE
        ) as in_use;

        v_keys_in_use := COALESCE(v_keys_in_use, ARRAY[]::BIGINT[]);
        IF array_length(v_keys_in_use, 1) > 0 THEN
            v_error_messages := array_append(v_error_messages, format('Cannot hard-delete %s configurations because they are linked to active blueprint objects (arms, sites, partners, etc.).', array_length(v_keys_in_use, 1)));
            SELECT array_agg(k) INTO v_keys_to_action FROM unnest(v_final_sks_to_process) k WHERE k <> ALL(v_keys_in_use);
        END IF;
    END IF;

    IF array_length(v_keys_to_action, 1) IS NULL THEN
         RETURN QUERY SELECT jsonb_build_object(
            'status', CASE WHEN array_length(v_keys_in_use, 1) > 0 THEN 'error' ELSE 'success' END,
            'message', 'No records could be processed. All selected records have dependencies.',
            'processed_count', 0,
            'failed_count', COALESCE(array_length(v_keys_in_use, 1), 0),
            'summary', '[]'::jsonb,
            'errors', to_jsonb(v_error_messages)
        );
        RETURN;
    END IF;

    IF v_mode = 'hard' THEN
        v_action_taken_text := 'hard-deleted';
        WITH deleted_rows AS (DELETE FROM public.map_scenario_configuration WHERE scenario_configuration_sk = ANY(v_keys_to_action) RETURNING scenario_configuration_sk, scenario_configuration_bk)
        SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM deleted_rows t;
    ELSE
        DECLARE v_set_deleted_status BOOLEAN := (v_mode = 'soft');
        BEGIN
            v_action_taken_text := CASE WHEN v_set_deleted_status THEN 'soft-deleted' ELSE 'restored' END;
            WITH updated_rows AS (UPDATE public.map_scenario_configuration SET is_deleted = v_set_deleted_status, updated_by_user_sk = v_user_sk, updated_at = NOW() WHERE scenario_configuration_sk = ANY(v_keys_to_action) RETURNING scenario_configuration_sk, scenario_configuration_bk, is_deleted)
            SELECT jsonb_agg(t), count(*) INTO v_summary, v_processed_count FROM updated_rows t;
        END;
    END IF;

    RETURN QUERY SELECT jsonb_build_object(
        'status', CASE WHEN array_length(v_keys_in_use, 1) > 0 THEN 'partial_success' ELSE 'success' END,
        'message', format('%s records %s. %s records failed due to dependencies.', COALESCE(v_processed_count, 0), v_action_taken_text, COALESCE(array_length(v_keys_in_use, 1), 0)),
        'processed_count', COALESCE(v_processed_count, 0),
        'failed_count', COALESCE(array_length(v_keys_in_use, 1), 0),
        'summary', COALESCE(v_summary, '[]'::jsonb),
        'errors', to_jsonb(v_error_messages)
    );
END;
$$;


ALTER FUNCTION "public"."delete_scenario_configurations"("p_payload" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."delete_scenario_configurations"("p_payload" "jsonb") IS 'Archives, restores, or purges Scenario Configurations. Why: Manages the lifecycle of plan roots. How: Implements the "Safe Delete" pattern with dependency checks on all child blueprint objects.';



CREATE OR REPLACE FUNCTION "public"."update_scenario_configurations"("p_update" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.update_scenario_configurations
*   Version:        4.1 (Gold Standard - Payload & Lock Certified)
*   Author:         Senior AI QA Engineer
*   Description:    This definitive version fixes a critical payload parsing bug.
*                   It now correctly extracts the target SK from the top level
*                   of the payload and the fields to change from the nested
*                   'update_fields' object, fully complying with the Rich Audit
*                   mandate.
********************************************************************************/
DECLARE
    -- THE DEFINITIVE FIX: Correctly parse the payload according to the Rich Audit pattern.
    v_config_sk BIGINT := (p_update->>'scenario_configuration_sk')::bigint;
    v_update_data JSONB := p_update->'update_fields';
    v_updating_user_sk BIGINT := public.get_current_user_sk();
    v_old_record public.map_scenario_configuration;
    v_new_record public.map_scenario_configuration;
    v_changes jsonb := '{}'::jsonb;
    v_creator_name TEXT;
    v_updater_name TEXT;
    v_days_since_creation INT;
    v_human_readable_message TEXT;
    v_study_name TEXT;
    v_scenario_name TEXT;
    v_is_locked BOOLEAN;
BEGIN
    SELECT * INTO v_old_record FROM public.map_scenario_configuration WHERE scenario_configuration_sk = v_config_sk FOR UPDATE;
    IF NOT FOUND THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Update failed. Record not found or permission denied.');
        RETURN;
    END IF;

    SELECT dbs.is_locked INTO v_is_locked
    FROM public.dim_budget_scenario dbs
    WHERE dbs.scenario_sk = v_old_record.scenario_sk;

    IF v_is_locked THEN
        RETURN QUERY SELECT jsonb_build_object('status', 'error', 'message', 'Update failed. The parent scenario is locked.');
        RETURN;
    END IF;

    WITH updated_row AS (
        UPDATE public.map_scenario_configuration SET
            scenario_configuration_bk = CASE
                                            WHEN v_update_data ? 'scenario_configuration_bk' AND jsonb_typeof(v_update_data->'scenario_configuration_bk') = 'null' THEN 'CTF-SCFG-' || extensions.uuid_generate_v4()::text
                                            WHEN v_update_data ? 'scenario_configuration_bk' THEN v_update_data->>'scenario_configuration_bk'
                                            ELSE scenario_configuration_bk
                                        END,
            start_date                = CASE WHEN v_update_data ? 'start_date' AND jsonb_typeof(v_update_data->'start_date') = 'null' THEN NULL WHEN v_update_data ? 'start_date' THEN (v_update_data->>'start_date')::date ELSE start_date END,
            end_date                  = CASE WHEN v_update_data ? 'end_date' AND jsonb_typeof(v_update_data->'end_date') = 'null' THEN NULL WHEN v_update_data ? 'end_date' THEN (v_update_data->>'end_date')::date ELSE end_date END,
            target_enrollment         = CASE WHEN v_update_data ? 'target_enrollment' AND jsonb_typeof(v_update_data->'target_enrollment') = 'null' THEN NULL WHEN v_update_data ? 'target_enrollment' THEN (v_update_data->>'target_enrollment')::int ELSE target_enrollment END,
            target_sites              = CASE WHEN v_update_data ? 'target_sites' AND jsonb_typeof(v_update_data->'target_sites') = 'null' THEN NULL WHEN v_update_data ? 'target_sites' THEN (v_update_data->>'target_sites')::int ELSE target_sites END,
            study_status              = CASE WHEN v_update_data ? 'study_status' AND jsonb_typeof(v_update_data->'study_status') = 'null' THEN NULL WHEN v_update_data ? 'study_status' THEN (v_update_data->>'study_status')::public.study_status_enum ELSE study_status END,
            updated_by_user_sk        = v_updating_user_sk,
            updated_at                = NOW()
        WHERE scenario_configuration_sk = v_config_sk
        RETURNING *
    )
    SELECT * INTO v_new_record FROM updated_row;

    IF v_new_record.scenario_configuration_bk IS DISTINCT FROM v_old_record.scenario_configuration_bk THEN v_changes := v_changes || jsonb_build_object('scenario_configuration_bk', jsonb_build_object('old', v_old_record.scenario_configuration_bk, 'new', v_new_record.scenario_configuration_bk)); END IF;
    IF v_new_record.start_date IS DISTINCT FROM v_old_record.start_date THEN v_changes := v_changes || jsonb_build_object('start_date', jsonb_build_object('old', v_old_record.start_date, 'new', v_new_record.start_date)); END IF;
    IF v_new_record.end_date IS DISTINCT FROM v_old_record.end_date THEN v_changes := v_changes || jsonb_build_object('end_date', jsonb_build_object('old', v_old_record.end_date, 'new', v_new_record.end_date)); END IF;
    IF v_new_record.target_enrollment IS DISTINCT FROM v_old_record.target_enrollment THEN v_changes := v_changes || jsonb_build_object('target_enrollment', jsonb_build_object('old', v_old_record.target_enrollment, 'new', v_new_record.target_enrollment)); END IF;
    IF v_new_record.target_sites IS DISTINCT FROM v_old_record.target_sites THEN v_changes := v_changes || jsonb_build_object('target_sites', jsonb_build_object('old', v_old_record.target_sites, 'new', v_new_record.target_sites)); END IF;
    IF v_new_record.study_status IS DISTINCT FROM v_old_record.study_status THEN v_changes := v_changes || jsonb_build_object('study_status', jsonb_build_object('old', v_old_record.study_status, 'new', v_new_record.study_status)); END IF;

    SELECT s.protocol_number, b.scenario_name INTO v_study_name, v_scenario_name FROM public.map_scenario_configuration m JOIN public.dim_study s ON m.study_sk = s.study_sk JOIN public.dim_budget_scenario b ON m.scenario_sk = b.scenario_sk WHERE m.scenario_configuration_sk = v_new_record.scenario_configuration_sk;
    SELECT u.user_name INTO v_creator_name FROM public.dim_user u WHERE u.user_sk = v_old_record.created_by_user_sk;
    SELECT u.user_name INTO v_updater_name FROM public.dim_user u WHERE u.user_sk = v_updating_user_sk;
    v_days_since_creation := DATE_PART('day', v_new_record.updated_at - v_old_record.created_at);
    v_human_readable_message := format('Configuration for study "%s" / scenario "%s" updated by %s. This record was created %s days ago by %s.', v_study_name, v_scenario_name, COALESCE(v_updater_name, 'Unknown User'), v_days_since_creation, COALESCE(v_creator_name, 'Unknown User'));

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', v_human_readable_message, 'updated_sk', v_new_record.scenario_configuration_sk, 'record', row_to_json(v_new_record), 'changes', v_changes);
END;
$$;


ALTER FUNCTION "public"."update_scenario_configurations"("p_update" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."update_scenario_configurations"("p_update" "jsonb") IS 'Updates a single Scenario Configuration. Why: Allows for adjustment of high-level plan parameters like dates, targets, and status. How: A partial update function that implements the "Rich Audit" and "NULL-Safe" patterns.';




CREATE OR REPLACE FUNCTION "public"."get_template_metadata"() RETURNS TABLE("public_table_name" "text", "entry_description" "text", "business_insight" "text", "payload_analytics" "jsonb", "failures_report" "jsonb", "gaps_report" "jsonb", "business_logic_report" "jsonb", "last_refreshed_at" timestamp with time zone)
    LANGUAGE "sql" STABLE SECURITY DEFINER
    SET "search_path" TO 'public', 'private'
    AS $$
/********************************************************************************
*   Function:       public.get_template_metadata
*   Version:        3.2 (Gold Standard - Corrected Payload)
*   Author:         Senior AI Business Intelligence Architect
*   Description:    This definitive version fixes a critical bug where the
*                   public_table_name was not being returned, making it impossible
*                   for clients to map metadata to a specific entity.
********************************************************************************/
WITH ordered_metadata AS (
  SELECT
    *,
    stage_number AS stage_num,
    CASE d.template_table_name
        -- Stage 1 Order
        WHEN 'private.template_organizations' THEN 1 WHEN 'private.template_users' THEN 2 WHEN 'private.template_memberships' THEN 3
        WHEN 'private.template_reimbursement_types' THEN 4 WHEN 'private.template_activities' THEN 5 WHEN 'private.template_budget_categories' THEN 6
        -- Stage 2 Order
        WHEN 'private.template_studies' THEN 1 WHEN 'private.template_scenarios' THEN 2 WHEN 'private.template_sites' THEN 3
        WHEN 'private.template_amendments' THEN 4 WHEN 'private.template_study_arms' THEN 5 WHEN 'private.template_study_epochs' THEN 6
        WHEN 'private.template_study_visits' THEN 7 WHEN 'private.template_scenario_configurations' THEN 8 WHEN 'private.template_study_partners' THEN 9
        WHEN 'private.template_study_staff' THEN 10 WHEN 'private.template_forecast_calculation_configs' THEN 11
        -- Stage 3 Order
        WHEN 'private.template_soa_mappings' THEN 1 WHEN 'private.template_activity_costs' THEN 2
        -- Stage 4 & 5 Order
        WHEN 'private.template_fact_enrollment' THEN 1 WHEN 'private.template_fact_forecast_detail' THEN 2
        ELSE 99
    END AS intra_stage_order
  FROM private.template_metadata_dictionary d
)
SELECT
  -- THE FIX: Add the missing column to the function's output SELECT list.
  o.public_table_name,
  COALESCE(ROW_NUMBER() OVER (ORDER BY o.stage_num, o.intra_stage_order)::text, '?') || '. ' ||
    COALESCE(
        CASE o.template_table_name
            WHEN 'private.template_organizations' THEN 'Organizations: Populates fictitious organizations (Sponsors, Sites, CROs).'
            WHEN 'private.template_users' THEN 'Users: Populates fictitious users (PIs, Study Managers).'
            WHEN 'private.template_memberships' THEN 'User Memberships: Assigns users to organizations.'
            WHEN 'private.template_reimbursement_types' THEN 'Reimbursement Types: Populates financial reimbursement rules.'
            WHEN 'private.template_activities' THEN 'Activities: Populates the master list of billable trial activities.'
            WHEN 'private.template_budget_categories' THEN 'Budget Categories: Populates the hierarchical chart of accounts.'
            WHEN 'private.template_studies' THEN 'Studies: Populates the core study master list.'
            WHEN 'private.template_scenarios' THEN 'Scenarios: Populates portfolio-level scenarios (e.g., "2026 Plan").'
            WHEN 'private.template_sites' THEN 'Sites: Populates the master list of clinical sites available to studies.'
            WHEN 'private.template_amendments' THEN 'Amendments: Populates example protocol amendments for studies.'
            WHEN 'private.template_study_arms' THEN 'Study Arms: Populates treatment arms for each scenario.'
            WHEN 'private.template_study_epochs' THEN 'Study Epochs: Populates major time periods for each arm.'
            WHEN 'private.template_study_visits' THEN 'Study Visits: Populates the detailed visit schedules for each epoch.'
            WHEN 'private.template_scenario_configurations' THEN 'Scenario Configurations: The definitive bridge linking studies to scenarios.'
            WHEN 'private.template_study_partners' THEN 'Study Partners: Assigns partner roles (CRO, Lab) to a scenario.'
            WHEN 'private.template_forecast_calculation_configs' THEN 'Calculation Rules: Populates the rules that drive the forecast engine for a scenario.'
            WHEN 'private.template_study_staff' THEN 'Study Staff: Populates the study staff roles maps for a complete roster of personnel at a site.'
            WHEN 'private.template_soa_mappings' THEN 'Schedule of Activities (SoA): Defines which activities occur at each visit in a scenario.'
            WHEN 'private.template_activity_costs' THEN 'Activity Costs: Defines the price for each activity within a specific scenario.'
            WHEN 'private.template_fact_enrollment' THEN 'Enrollment Projections: Populates time-phased subject enrollment data.'
            WHEN 'private.template_fact_forecast_detail' THEN 'Financial Forecast Details: Populates the final, granular financial ledger.'
            ELSE o.public_table_name
        END,
        o.template_table_name
    )
    || ' Part of Stage ' || COALESCE(o.stage_num::text, '?') || '.'
    || ' Creates ~' || COALESCE((o.metadata->>'total_record_count')::bigint, 0) || ' records (~' || COALESCE(o.estimated_payload_size_kb::text, '0') || ' KB, ~' || COALESCE(o.estimated_token_count::text, '0') || ' tokens).'
    AS entry_description,

  private.summarize_validation_status(
      o.failures_report,
      o.gaps_report,
      o.business_logic_report,
      o.template_table_name,
      o.metadata
  ) AS business_insight,

  o.metadata AS payload_analytics,
  o.failures_report,
  o.gaps_report,
  o.business_logic_report,
  o.last_refreshed_at
FROM ordered_metadata o
ORDER BY o.stage_num, o.intra_stage_order;
$$;


ALTER FUNCTION "public"."get_template_metadata"() OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_template_metadata"() IS 'Returns the "IntelliSense" catalog of available template data. Why: The first step in the user onboarding and data seeding workflow, allowing users to discover what data is available. How: Reads from a materialized view to provide a fast, rich summary of each template table, including analytics and validation status.';





CREATE OR REPLACE FUNCTION "public"."clear_template_data"() RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql"
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.clear_template_data
*   Version:        36.0 (Definitive - Fact Anchor Certified)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version provides the final, correct fix for
*                   the "Phase 1: Mark" logic, ensuring that adopted fact records
*                   are correctly identified as protected anchor points.
*
*   FIX v36.0:      The previous version failed the "Adopt an Enrollment Fact"
*                   certification test. The root cause was the omission of a
*                   query in Phase 1 to identify adopted records in the
*                   `fact_enrollment` table. This version adds the missing
*                   query to populate `v_protected_enrollment_pks` with any
*                   non-template records, establishing them as valid anchors for
*                   the dependency traversal. This fix is identical in principle
*                   to the existing logic for `fact_forecast_detail`.
*
*                   The function is now fully certified against the entire
*                   comprehensive adoption test suite.
********************************************************************************/
DECLARE
    v_tpl_bk_prefix TEXT := 'TPL-%';
    v_report JSONB;
    v_current_org_sk BIGINT := public.get_current_organization_sk();

    -- SK Lists for Protected Entities
    v_protected_study_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_scenario_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_config_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_partner_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_org_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_activity_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_reimb_type_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_budget_cat_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_user_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_arm_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_epoch_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_visit_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_site_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_staff_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_cost_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_soa_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_rule_sks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_enrollment_pks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_forecast_pks BIGINT[] := ARRAY[]::BIGINT[];
    v_protected_amendment_sks BIGINT[] := ARRAY[]::BIGINT[];

    -- Report Variables
    v_summary_report JSONB := '{}'::jsonb;
    v_temp_deleted_count INT;
    v_temp_kept_count INT;
    v_temp_total_remaining INT;
    
    -- Loop control
    v_total_protected_before INT;
    v_total_protected_after INT;
    v_max_iterations INT := 20;
    v_current_iteration INT := 0;
BEGIN
    RAISE NOTICE '--- Executing Definitive Template Clearance (v36.0) for Org SK: % ---', v_current_org_sk;
    IF v_current_org_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;

    -- =========================================================================
    -- PHASE 1: MARK - Identify all initially adopted anchor points
    -- =========================================================================
    RAISE NOTICE '  -> Phase 1: Identifying all adopted anchor points...';
    v_protected_study_sks        := COALESCE(array_agg(study_sk), ARRAY[]::BIGINT[]) FROM public.dim_study WHERE organization_sk = v_current_org_sk AND study_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_org_sks          := COALESCE(array_agg(organization_sk), ARRAY[]::BIGINT[]) FROM public.dim_organization WHERE parent_organization_sk = v_current_org_sk AND is_clerk_managed = FALSE AND organization_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_user_sks         := COALESCE(array_agg(user_sk), ARRAY[]::BIGINT[]) FROM public.dim_user WHERE organization_sk = v_current_org_sk AND is_clerk_managed = FALSE AND user_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_activity_sks     := COALESCE(array_agg(activity_sk), ARRAY[]::BIGINT[]) FROM public.dim_activity WHERE organization_sk = v_current_org_sk AND activity_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_budget_cat_sks   := COALESCE(array_agg(budget_category_sk), ARRAY[]::BIGINT[]) FROM public.dim_budget_category WHERE organization_sk = v_current_org_sk AND budget_category_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_reimb_type_sks   := COALESCE(array_agg(reimbursement_type_sk), ARRAY[]::BIGINT[]) FROM public.dim_reimbursement_type WHERE organization_sk = v_current_org_sk AND reimbursement_type_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_config_sks       := COALESCE(array_agg(scenario_configuration_sk), ARRAY[]::BIGINT[]) FROM public.map_scenario_configuration WHERE organization_sk = v_current_org_sk AND scenario_configuration_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_arm_sks          := COALESCE(array_agg(arm_sk), ARRAY[]::BIGINT[]) FROM public.dim_study_arm WHERE organization_sk = v_current_org_sk AND arm_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_epoch_sks        := COALESCE(array_agg(epoch_sk), ARRAY[]::BIGINT[]) FROM public.dim_study_epochs WHERE organization_sk = v_current_org_sk AND epoch_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_visit_sks        := COALESCE(array_agg(visit_sk), ARRAY[]::BIGINT[]) FROM public.dim_study_visits WHERE organization_sk = v_current_org_sk AND visit_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_partner_sks      := COALESCE(array_agg(study_partner_sk), ARRAY[]::BIGINT[]) FROM public.map_study_partners WHERE organization_sk = v_current_org_sk AND study_partner_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_site_sks         := COALESCE(array_agg(site_sk), ARRAY[]::BIGINT[]) FROM public.dim_site WHERE organization_sk = v_current_org_sk AND site_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_staff_sks        := COALESCE(array_agg(study_staff_sk), ARRAY[]::BIGINT[]) FROM public.map_study_staff WHERE organization_sk = v_current_org_sk AND study_staff_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_cost_sks         := COALESCE(array_agg(activity_cost_sk), ARRAY[]::BIGINT[]) FROM public.dim_activity_cost WHERE organization_sk = v_current_org_sk AND activity_cost_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_soa_sks          := COALESCE(array_agg(map_soa_sk), ARRAY[]::BIGINT[]) FROM public.map_study_visit_activity WHERE organization_sk = v_current_org_sk AND map_soa_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_rule_sks         := COALESCE(array_agg(forecast_config_sk), ARRAY[]::BIGINT[]) FROM public.dim_forecast_calculation_config WHERE organization_sk = v_current_org_sk AND forecast_config_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_amendment_sks    := COALESCE(array_agg(amendment_sk), ARRAY[]::BIGINT[]) FROM public.dim_amendment WHERE organization_sk = v_current_org_sk AND amendment_bk NOT LIKE v_tpl_bk_prefix;
    
    -- THE DEFINITIVE FIX v36.0: Add missing anchor point identification for fact tables.
    v_protected_enrollment_pks   := COALESCE(array_agg(enrollment_pk), ARRAY[]::BIGINT[]) FROM public.fact_enrollment WHERE organization_sk = v_current_org_sk AND enrollment_bk NOT LIKE v_tpl_bk_prefix;
    v_protected_forecast_pks     := COALESCE(array_agg(forecast_detail_pk), ARRAY[]::BIGINT[]) FROM public.fact_forecast_detail WHERE organization_sk = v_current_org_sk AND forecast_detail_bk NOT LIKE v_tpl_bk_prefix;

    -- =========================================================================
    -- PHASE 2: MARK - Iteratively traverse dependencies
    -- =========================================================================
    RAISE NOTICE '  -> Phase 2: Iteratively traversing dependencies...';
    LOOP
        v_current_iteration := v_current_iteration + 1;
        IF v_current_iteration > v_max_iterations THEN RAISE EXCEPTION 'Max iterations reached in dependency traversal.'; END IF;

        v_total_protected_before := COALESCE(array_length(v_protected_study_sks, 1), 0) + COALESCE(array_length(v_protected_scenario_sks, 1), 0) + COALESCE(array_length(v_protected_config_sks, 1), 0) + COALESCE(array_length(v_protected_partner_sks, 1), 0) + COALESCE(array_length(v_protected_org_sks, 1), 0) + COALESCE(array_length(v_protected_activity_sks, 1), 0) + COALESCE(array_length(v_protected_reimb_type_sks, 1), 0) + COALESCE(array_length(v_protected_budget_cat_sks, 1), 0) + COALESCE(array_length(v_protected_user_sks, 1), 0) + COALESCE(array_length(v_protected_arm_sks, 1), 0) + COALESCE(array_length(v_protected_epoch_sks, 1), 0) + COALESCE(array_length(v_protected_visit_sks, 1), 0) + COALESCE(array_length(v_protected_site_sks, 1), 0) + COALESCE(array_length(v_protected_staff_sks, 1), 0) + COALESCE(array_length(v_protected_cost_sks, 1), 0) + COALESCE(array_length(v_protected_soa_sks, 1), 0) + COALESCE(array_length(v_protected_rule_sks, 1), 0) + COALESCE(array_length(v_protected_amendment_sks, 1), 0) + COALESCE(array_length(v_protected_enrollment_pks, 1), 0) + COALESCE(array_length(v_protected_forecast_pks, 1), 0);

        -- Upward Traversal (Child -> Parent)
        v_protected_study_sks      := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_study_sks, array_cat(
                                        ARRAY(SELECT study_sk FROM public.map_scenario_configuration WHERE scenario_configuration_sk = ANY(v_protected_config_sks)),
                                        ARRAY(SELECT study_sk FROM public.dim_amendment WHERE amendment_sk = ANY(v_protected_amendment_sks))
                                     ))));
        v_protected_scenario_sks   := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_scenario_sks,  ARRAY(SELECT scenario_sk FROM public.map_scenario_configuration WHERE scenario_configuration_sk = ANY(v_protected_config_sks)))));
        
        v_protected_config_sks     := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_config_sks, array_cat(
                                        ARRAY(SELECT scenario_configuration_sk FROM public.dim_study_arm WHERE arm_sk = ANY(v_protected_arm_sks)),
                                        array_cat(
                                            ARRAY(SELECT scenario_configuration_sk FROM public.map_study_partners WHERE study_partner_sk = ANY(v_protected_partner_sks)),
                                            array_cat(
                                                ARRAY(SELECT scenario_configuration_sk FROM public.dim_activity_cost WHERE activity_cost_sk = ANY(v_protected_cost_sks)),
                                                array_cat(
                                                    ARRAY(SELECT scenario_configuration_sk FROM public.dim_forecast_calculation_config WHERE forecast_config_sk = ANY(v_protected_rule_sks)),
                                                    array_cat(
                                                        ARRAY(SELECT scenario_configuration_sk FROM public.fact_enrollment WHERE enrollment_pk = ANY(v_protected_enrollment_pks)),
                                                        ARRAY(SELECT scenario_configuration_sk FROM public.fact_forecast_detail WHERE forecast_detail_pk = ANY(v_protected_forecast_pks))
                                                    )
                                                )
                                            )
                                        )
                                     ))));

        v_protected_arm_sks        := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_arm_sks,        ARRAY(SELECT arm_sk FROM public.dim_study_epochs WHERE epoch_sk = ANY(v_protected_epoch_sks)))));
        v_protected_epoch_sks      := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_epoch_sks,      ARRAY(SELECT epoch_sk FROM public.dim_study_visits WHERE visit_sk = ANY(v_protected_visit_sks)))));
        v_protected_visit_sks      := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_visit_sks,      ARRAY(SELECT visit_sk FROM public.map_study_visit_activity WHERE map_soa_sk = ANY(v_protected_soa_sks)))));
        v_protected_soa_sks        := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_soa_sks,        ARRAY(SELECT map_soa_sk FROM public.map_study_visit_activity WHERE activity_sk = ANY(v_protected_activity_sks)))));
        v_protected_activity_sks   := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_activity_sks,   array_cat(ARRAY(SELECT activity_sk FROM public.map_study_visit_activity WHERE map_soa_sk = ANY(v_protected_soa_sks)), ARRAY(SELECT activity_sk FROM public.dim_activity_cost WHERE activity_cost_sk = ANY(v_protected_cost_sks))))));
        v_protected_budget_cat_sks := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_budget_cat_sks, array_cat(ARRAY(SELECT budget_category_sk FROM public.dim_activity WHERE activity_sk = ANY(v_protected_activity_sks)), ARRAY(SELECT parent_category_sk FROM public.dim_budget_category WHERE budget_category_sk = ANY(v_protected_budget_cat_sks) AND parent_category_sk IS NOT NULL)))));
        v_protected_user_sks       := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_user_sks,       ARRAY(SELECT user_sk FROM public.map_study_staff WHERE study_staff_sk = ANY(v_protected_staff_sks)))));
        v_protected_org_sks        := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_org_sks, array_cat(
                                        ARRAY(SELECT partner_organization_sk FROM public.map_study_partners WHERE study_partner_sk = ANY(v_protected_partner_sks)),
                                        ARRAY(SELECT site_organization_sk FROM public.dim_site WHERE site_sk = ANY(v_protected_site_sks))
                                     ))));
        v_protected_reimb_type_sks := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_reimb_type_sks, array_cat(
                                        ARRAY(SELECT reimbursement_type_sk FROM public.dim_activity_cost WHERE activity_cost_sk = ANY(v_protected_cost_sks) AND reimbursement_type_sk IS NOT NULL),
                                        ARRAY(SELECT reimbursement_type_sk FROM public.dim_activity WHERE activity_sk = ANY(v_protected_activity_sks) AND reimbursement_type_sk IS NOT NULL)
                                     ))));

        -- Downward Traversal (Parent -> Children)
        v_protected_config_sks     := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_config_sks,     ARRAY(SELECT scenario_configuration_sk FROM public.map_scenario_configuration WHERE study_sk = ANY(v_protected_study_sks)))));
        v_protected_amendment_sks  := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_amendment_sks,  ARRAY(SELECT amendment_sk FROM public.dim_amendment WHERE study_sk = ANY(v_protected_study_sks)))));
        v_protected_arm_sks        := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_arm_sks,        ARRAY(SELECT arm_sk FROM public.dim_study_arm WHERE scenario_configuration_sk = ANY(v_protected_config_sks)))));
        v_protected_epoch_sks      := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_epoch_sks,      ARRAY(SELECT epoch_sk FROM public.dim_study_epochs WHERE arm_sk = ANY(v_protected_arm_sks)))));
        v_protected_visit_sks      := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_visit_sks,      ARRAY(SELECT visit_sk FROM public.dim_study_visits WHERE epoch_sk = ANY(v_protected_epoch_sks)))));
        v_protected_soa_sks        := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_soa_sks,        ARRAY(SELECT map_soa_sk FROM public.map_study_visit_activity WHERE visit_sk = ANY(v_protected_visit_sks)))));
        v_protected_staff_sks      := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_staff_sks,      ARRAY(SELECT study_staff_sk FROM public.map_study_staff WHERE study_partner_sk = ANY(v_protected_partner_sks)))));
        v_protected_cost_sks       := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_cost_sks,       ARRAY(SELECT activity_cost_sk FROM public.dim_activity_cost WHERE scenario_configuration_sk = ANY(v_protected_config_sks)))));
        v_protected_rule_sks       := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_rule_sks,       ARRAY(SELECT forecast_config_sk FROM public.dim_forecast_calculation_config WHERE scenario_configuration_sk = ANY(v_protected_config_sks)))));
        v_protected_partner_sks    := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_partner_sks, array_cat(
                                        ARRAY(SELECT study_partner_sk FROM public.map_study_staff WHERE study_staff_sk = ANY(v_protected_staff_sks)),
                                        array_cat(
                                            ARRAY(SELECT site_partner_sk FROM public.dim_site WHERE site_sk = ANY(v_protected_site_sks)),
                                            ARRAY(SELECT study_partner_sk FROM public.map_study_partners WHERE scenario_configuration_sk = ANY(v_protected_config_sks) OR partner_organization_sk = ANY(v_protected_org_sks))
                                        )
                                     ))));
        v_protected_site_sks       := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_site_sks, array_cat(
                                        ARRAY(SELECT site_sk FROM public.dim_site WHERE site_partner_sk = ANY(v_protected_partner_sks)),
                                        ARRAY(SELECT site_sk FROM public.dim_site WHERE site_organization_sk = ANY(v_protected_org_sks))
                                     ))));
        v_protected_enrollment_pks := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_enrollment_pks, ARRAY(SELECT enrollment_pk FROM public.fact_enrollment WHERE scenario_configuration_sk = ANY(v_protected_config_sks)))));
        v_protected_forecast_pks   := ARRAY(SELECT DISTINCT unnest(array_cat(v_protected_forecast_pks,   ARRAY(SELECT forecast_detail_pk FROM public.fact_forecast_detail WHERE scenario_configuration_sk = ANY(v_protected_config_sks)))));

        v_total_protected_after := COALESCE(array_length(v_protected_study_sks, 1), 0) + COALESCE(array_length(v_protected_scenario_sks, 1), 0) + COALESCE(array_length(v_protected_config_sks, 1), 0) + COALESCE(array_length(v_protected_partner_sks, 1), 0) + COALESCE(array_length(v_protected_org_sks, 1), 0) + COALESCE(array_length(v_protected_activity_sks, 1), 0) + COALESCE(array_length(v_protected_reimb_type_sks, 1), 0) + COALESCE(array_length(v_protected_budget_cat_sks, 1), 0) + COALESCE(array_length(v_protected_user_sks, 1), 0) + COALESCE(array_length(v_protected_arm_sks, 1), 0) + COALESCE(array_length(v_protected_epoch_sks, 1), 0) + COALESCE(array_length(v_protected_visit_sks, 1), 0) + COALESCE(array_length(v_protected_site_sks, 1), 0) + COALESCE(array_length(v_protected_staff_sks, 1), 0) + COALESCE(array_length(v_protected_cost_sks, 1), 0) + COALESCE(array_length(v_protected_soa_sks, 1), 0) + COALESCE(array_length(v_protected_rule_sks, 1), 0) + COALESCE(array_length(v_protected_amendment_sks, 1), 0) + COALESCE(array_length(v_protected_enrollment_pks, 1), 0) + COALESCE(array_length(v_protected_forecast_pks, 1), 0);

        IF v_total_protected_after = v_total_protected_before THEN
            EXIT;
        END IF;
    END LOOP;
    RAISE NOTICE '  -> Dependency traversal complete in % iterations.', v_current_iteration;

    -- =========================================================================
    -- PHASE 3: SWEEP - Execute surgical deletions and build the report.
    -- =========================================================================
    RAISE NOTICE '  -> Phase 3: Sweeping for un-adopted template entities and generating report...';
    
    WITH deleted AS (DELETE FROM public.fact_forecast_detail WHERE organization_sk = v_current_org_sk AND forecast_detail_bk LIKE v_tpl_bk_prefix AND forecast_detail_pk <> ALL(COALESCE(v_protected_forecast_pks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.fact_forecast_detail WHERE organization_sk = v_current_org_sk AND forecast_detail_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.fact_forecast_detail WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('forecast_details', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.fact_enrollment WHERE organization_sk = v_current_org_sk AND enrollment_bk LIKE v_tpl_bk_prefix AND enrollment_pk <> ALL(COALESCE(v_protected_enrollment_pks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.fact_enrollment WHERE organization_sk = v_current_org_sk AND enrollment_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.fact_enrollment WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('enrollment_facts', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.map_study_visit_activity WHERE organization_sk = v_current_org_sk AND map_soa_bk LIKE v_tpl_bk_prefix AND map_soa_sk <> ALL(COALESCE(v_protected_soa_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.map_study_visit_activity WHERE organization_sk = v_current_org_sk AND map_soa_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.map_study_visit_activity WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('soa_mappings', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_activity_cost WHERE organization_sk = v_current_org_sk AND activity_cost_bk LIKE v_tpl_bk_prefix AND activity_cost_sk <> ALL(COALESCE(v_protected_cost_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_activity_cost WHERE organization_sk = v_current_org_sk AND activity_cost_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_activity_cost WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('activity_costs', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.map_study_staff WHERE organization_sk = v_current_org_sk AND study_staff_bk LIKE v_tpl_bk_prefix AND study_staff_sk <> ALL(COALESCE(v_protected_staff_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.map_study_staff WHERE organization_sk = v_current_org_sk AND study_staff_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.map_study_staff WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('study_staff', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_site WHERE organization_sk = v_current_org_sk AND site_bk LIKE v_tpl_bk_prefix AND site_sk <> ALL(COALESCE(v_protected_site_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_site WHERE organization_sk = v_current_org_sk AND site_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_site WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('sites', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_forecast_calculation_config WHERE organization_sk = v_current_org_sk AND forecast_config_bk LIKE v_tpl_bk_prefix AND forecast_config_sk <> ALL(COALESCE(v_protected_rule_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_forecast_calculation_config WHERE organization_sk = v_current_org_sk AND forecast_config_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_forecast_calculation_config WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('forecast_rules', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_study_visits WHERE organization_sk = v_current_org_sk AND visit_bk LIKE v_tpl_bk_prefix AND visit_sk <> ALL(COALESCE(v_protected_visit_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_study_visits WHERE organization_sk = v_current_org_sk AND visit_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_study_visits WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('visits', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_study_epochs WHERE organization_sk = v_current_org_sk AND epoch_bk LIKE v_tpl_bk_prefix AND epoch_sk <> ALL(COALESCE(v_protected_epoch_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_study_epochs WHERE organization_sk = v_current_org_sk AND epoch_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_study_epochs WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('epochs', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_study_arm WHERE organization_sk = v_current_org_sk AND arm_bk LIKE v_tpl_bk_prefix AND arm_sk <> ALL(COALESCE(v_protected_arm_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_study_arm WHERE organization_sk = v_current_org_sk AND arm_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_study_arm WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('arms', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.map_study_partners WHERE organization_sk = v_current_org_sk AND study_partner_bk LIKE v_tpl_bk_prefix AND study_partner_sk <> ALL(COALESCE(v_protected_partner_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.map_study_partners WHERE organization_sk = v_current_org_sk AND study_partner_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.map_study_partners WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('partners', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.map_scenario_configuration WHERE organization_sk = v_current_org_sk AND scenario_configuration_bk LIKE v_tpl_bk_prefix AND scenario_configuration_sk <> ALL(COALESCE(v_protected_config_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.map_scenario_configuration WHERE organization_sk = v_current_org_sk AND scenario_configuration_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.map_scenario_configuration WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('configurations', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_amendment WHERE organization_sk = v_current_org_sk AND amendment_bk LIKE v_tpl_bk_prefix AND amendment_sk <> ALL(COALESCE(v_protected_amendment_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_amendment WHERE organization_sk = v_current_org_sk AND amendment_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_amendment WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('amendments', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_study WHERE organization_sk = v_current_org_sk AND study_bk LIKE v_tpl_bk_prefix AND study_sk <> ALL(COALESCE(v_protected_study_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_study WHERE organization_sk = v_current_org_sk AND study_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_study WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('studies', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_budget_scenario WHERE organization_sk = v_current_org_sk AND scenario_bk LIKE v_tpl_bk_prefix AND scenario_sk <> ALL(COALESCE(v_protected_scenario_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_budget_scenario WHERE organization_sk = v_current_org_sk AND scenario_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_budget_scenario WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('scenarios', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_activity WHERE organization_sk = v_current_org_sk AND activity_bk LIKE v_tpl_bk_prefix AND activity_sk <> ALL(COALESCE(v_protected_activity_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_activity WHERE organization_sk = v_current_org_sk AND activity_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_activity WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('activities', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_budget_category WHERE organization_sk = v_current_org_sk AND budget_category_bk LIKE v_tpl_bk_prefix AND budget_category_sk <> ALL(COALESCE(v_protected_budget_cat_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_budget_category WHERE organization_sk = v_current_org_sk AND budget_category_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_budget_category WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('budget_categories', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_reimbursement_type WHERE organization_sk = v_current_org_sk AND reimbursement_type_bk LIKE v_tpl_bk_prefix AND reimbursement_type_sk <> ALL(COALESCE(v_protected_reimb_type_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_reimbursement_type WHERE organization_sk = v_current_org_sk AND reimbursement_type_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_reimbursement_type WHERE organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('reimbursement_types', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_user WHERE organization_sk = v_current_org_sk AND user_bk LIKE v_tpl_bk_prefix AND user_sk <> ALL(COALESCE(v_protected_user_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_user WHERE organization_sk = v_current_org_sk AND user_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_user WHERE organization_sk = v_current_org_sk AND is_clerk_managed = false;
    v_summary_report := v_summary_report || jsonb_build_object('users', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));

    WITH deleted AS (DELETE FROM public.dim_organization WHERE parent_organization_sk = v_current_org_sk AND organization_bk LIKE v_tpl_bk_prefix AND organization_sk <> ALL(COALESCE(v_protected_org_sks, ARRAY[]::BIGINT[])) RETURNING 1) SELECT count(*) INTO v_temp_deleted_count FROM deleted;
    SELECT count(*) INTO v_temp_kept_count FROM public.dim_organization WHERE parent_organization_sk = v_current_org_sk AND organization_bk LIKE v_tpl_bk_prefix;
    SELECT count(*) INTO v_temp_total_remaining FROM public.dim_organization WHERE parent_organization_sk = v_current_org_sk;
    v_summary_report := v_summary_report || jsonb_build_object('organizations', jsonb_build_object('deleted', v_temp_deleted_count, 'kept_template_dependencies', v_temp_kept_count, 'total_remaining', v_temp_total_remaining));
    RAISE NOTICE '  -> Sweep complete.';
    
    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'message', 'Unused template data cleared successfully.', 'summary', v_summary_report);
END;
$$;


ALTER FUNCTION "public"."clear_template_data"() OWNER TO "postgres";


COMMENT ON FUNCTION "public"."clear_template_data"() IS 'Clears all unused template data from an organization''s sandbox while preserving any records that have been "adopted" by the user. Why: Keeps the user''s workspace tidy after they have explored the template data. How: Identifies adopted records and their dependencies, then deletes all other template-prefixed data.';







CREATE OR REPLACE FUNCTION "public"."create_activities_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_activities_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    A high-performance, SECURITY DEFINER worker for the seeder.
*                   This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_activity (
            organization_sk, activity_bk, activity_name, activity_type,
            activity_domain, activity_category, budget_category_sk,
            reimbursement_type_sk, standard_code, description, is_billable,
            created_by_user_sk, updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.activity_bk,
            x.activity_name,
            (x.activity_type)::public.activity_type_enum,
            x.activity_domain,
            x.activity_category,
            bc.budget_category_sk,
            rt.reimbursement_type_sk,
            x.standard_code,
            x.description,
            (x.is_billable)::boolean,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            activity_bk TEXT, activity_name TEXT, activity_type TEXT,
            activity_domain TEXT, activity_category TEXT, budget_category_bk TEXT,
            reimbursement_type_bk TEXT, standard_code TEXT, description TEXT, is_billable TEXT
        )
        JOIN public.dim_budget_category bc ON x.budget_category_bk = bc.budget_category_bk AND bc.organization_sk = p_calling_org_sk
        LEFT JOIN public.dim_reimbursement_type rt ON x.reimbursement_type_bk = rt.reimbursement_type_bk AND rt.organization_sk = p_calling_org_sk
        ON CONFLICT (organization_sk, activity_bk) DO NOTHING
        RETURNING activity_sk, activity_bk, activity_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'inserted_count', v_inserted_count, 'summary', v_summary);
END;
$$;


ALTER FUNCTION "public"."create_activities_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_activities_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed activity data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_activity_costs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_activity_costs_for_seeder
*   Version:        1.2 (Enum Cast Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version fixes a type mismatch error by explicitly
*                   casting the incoming text `cost_unit` to the new
*                   `public.cost_unit_enum` type.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_activity_cost (
            organization_sk,
            scenario_configuration_sk,
            activity_sk,
            cost_bearing_partner_sk,
            payer_partner_sk,
            reimbursement_type_sk,
            activity_cost_bk,
            cost_unit,
            unit_cost,
            currency_code,
            effective_date,
            end_date,
            notes,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            msc.scenario_configuration_sk,
            da.activity_sk,
            cbp.study_partner_sk,
            pp.study_partner_sk,
            drt.reimbursement_type_sk,
            x.activity_cost_bk,
            x.cost_unit::public.cost_unit_enum, -- THE DEFINITIVE FIX: Explicit cast
            (x.unit_cost)::numeric,
            x.currency_code,
            (x.effective_date)::date,
            (x.end_date)::date,
            x.notes,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            activity_cost_bk TEXT,
            parent_scenario_configuration_bk TEXT,
            activity_bk TEXT,
            cost_bearing_partner_bk TEXT,
            payer_partner_bk TEXT,
            reimbursement_type_bk TEXT,
            cost_unit TEXT,
            unit_cost TEXT,
            currency_code TEXT,
            effective_date TEXT,
            end_date TEXT,
            notes TEXT
        )
        JOIN public.map_scenario_configuration msc ON x.parent_scenario_configuration_bk = msc.scenario_configuration_bk AND msc.organization_sk = p_calling_org_sk
        JOIN public.dim_activity da ON x.activity_bk = da.activity_bk AND da.organization_sk = p_calling_org_sk
        JOIN public.map_study_partners cbp ON x.cost_bearing_partner_bk = cbp.study_partner_bk AND cbp.organization_sk = p_calling_org_sk
        LEFT JOIN public.map_study_partners pp ON x.payer_partner_bk = pp.study_partner_bk AND pp.organization_sk = p_calling_org_sk
        LEFT JOIN public.dim_reimbursement_type drt ON x.reimbursement_type_bk = drt.reimbursement_type_bk AND drt.organization_sk = p_calling_org_sk
        ON CONFLICT (organization_sk, activity_cost_bk) DO NOTHING
        RETURNING activity_cost_sk, activity_cost_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s activity costs.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_activity_costs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_activity_costs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed activity cost data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_amendments_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_amendments_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_amendment (
            organization_sk,
            study_sk,
            amendment_bk,
            amendment_version_id,
            approval_date,
            effective_date,
            summary,
            amendment_status,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            ds.study_sk,
            x.amendment_bk,
            x.amendment_version_id,
            (x.approval_date)::date,
            (x.effective_date)::date,
            x.summary,
            (x.amendment_status)::public.amendment_status_enum,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            amendment_bk TEXT,
            parent_study_bk TEXT,
            amendment_version_id TEXT,
            approval_date TEXT,
            effective_date TEXT,
            summary TEXT,
            amendment_status TEXT
        )
        JOIN public.dim_study ds ON x.parent_study_bk = ds.study_bk AND ds.organization_sk = p_calling_org_sk
        ON CONFLICT (study_sk, amendment_bk) DO NOTHING
        RETURNING amendment_sk, amendment_bk, amendment_version_id
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s amendments.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_amendments_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_amendments_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed amendment data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_budget_categories_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_budget_categories_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Principal Database Architect
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_budget_category (
            organization_sk,
            budget_category_bk,
            category_name,
            parent_category_bk,
            is_rollup_target,
            description,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.budget_category_bk,
            x.category_name,
            x.parent_category_bk,
            (x.is_rollup_target)::boolean,
            x.description,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            budget_category_bk TEXT,
            category_name TEXT,
            parent_category_bk TEXT,
            is_rollup_target TEXT,
            description TEXT
        )
        ON CONFLICT (organization_sk, budget_category_bk) DO NOTHING
        RETURNING budget_category_sk, budget_category_bk, category_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s records.', v_inserted_count),
        'inserted_count', v_inserted_count,
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_budget_categories_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_budget_categories_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed budget category data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_budget_scenarios_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_budget_scenarios_for_seeder
*   Version:        1.3 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_budget_scenario (
            organization_sk,
            scenario_bk,
            scenario_name,
            scenario_type,
            "version",
            scenario_status,
            description,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.scenario_bk,
            x.scenario_name,
            (x.scenario_type)::public.budget_scenario_type_enum,
            COALESCE((x.version)::integer, 1),
            COALESCE((x.scenario_status)::public.budget_scenario_status_enum, 'Draft'),
            x.description,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            scenario_bk TEXT,
            scenario_name TEXT,
            scenario_type TEXT,
            "version" TEXT,
            scenario_status TEXT,
            description TEXT
        )
        ON CONFLICT (organization_sk, scenario_bk) DO NOTHING
        RETURNING scenario_sk, scenario_bk, scenario_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s scenarios.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_budget_scenarios_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_budget_scenarios_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed budget scenario data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_fact_enrollments_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_fact_enrollments_for_seeder
*   Version:        3.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.fact_enrollment (
            organization_sk,
            enrollment_bk,
            snapshot_date_sk,
            projected_enrollment_count,
            actual_enrollment_count,
            site_bk,
            is_asserted_actual,
            last_edit_source,
            scenario_configuration_bk,
            study_bk,
            scenario_bk,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.enrollment_bk,
            (x.snapshot_date_sk)::date,
            (x.projected_enrollment_count)::integer,
            (x.actual_enrollment_count)::integer,
            x.site_bk,
            (x.is_asserted_actual)::boolean,
            x.last_edit_source,
            x.scenario_configuration_bk,
            x.study_bk,
            x.scenario_bk,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            enrollment_bk TEXT,
            snapshot_date_sk TEXT,
            projected_enrollment_count TEXT,
            actual_enrollment_count TEXT,
            site_bk TEXT,
            is_asserted_actual TEXT,
            last_edit_source TEXT,
            scenario_configuration_bk TEXT,
            study_bk TEXT,
            scenario_bk TEXT
        )
        RETURNING enrollment_pk, enrollment_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s enrollment records.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_fact_enrollments_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_fact_enrollments_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed enrollment fact data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';




CREATE OR REPLACE FUNCTION "public"."create_fact_forecast_details_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_fact_forecast_details_for_seeder
*   Version:        2.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.fact_forecast_detail (
            organization_sk,
            forecast_detail_bk,
            snapshot_date_sk,
            cost_unit,
            unit_cost,
            forecast_units,
            reimbursement_factor,
            last_edit_source,
            scenario_configuration_bk,
            study_bk,
            scenario_bk,
            activity_bk,
            site_bk,
            payer_partner_bk,
            payee_partner_bk,
            source_arm_bk,
            source_epoch_bk,
            source_visit_bk,
            source_enrollment_bk,
            source_activity_cost_bk,
            source_map_soa_bk,
            source_forecast_config_bk,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.forecast_detail_bk,
            (x.snapshot_date_sk)::date,
            x.cost_unit,
            (x.unit_cost)::numeric,
            (x.forecast_units)::numeric,
            (x.reimbursement_factor)::numeric,
            x.last_edit_source,
            x.scenario_configuration_bk,
            x.study_bk,
            x.scenario_bk,
            x.activity_bk,
            x.site_bk,
            x.payer_partner_bk,
            x.payee_partner_bk,
            x.source_arm_bk,
            x.source_epoch_bk,
            x.source_visit_bk,
            x.source_enrollment_bk,
            x.source_activity_cost_bk,
            x.source_map_soa_bk,
            x.source_forecast_config_bk,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            forecast_detail_bk TEXT,
            snapshot_date_sk TEXT,
            cost_unit TEXT,
            unit_cost TEXT,
            forecast_units TEXT,
            reimbursement_factor TEXT,
            last_edit_source TEXT,
            scenario_configuration_bk TEXT,
            study_bk TEXT,
            scenario_bk TEXT,
            activity_bk TEXT,
            site_bk TEXT,
            payer_partner_bk TEXT,
            payee_partner_bk TEXT,
            source_arm_bk TEXT,
            source_epoch_bk TEXT,
            source_visit_bk TEXT,
            source_enrollment_bk TEXT,
            source_activity_cost_bk TEXT,
            source_map_soa_bk TEXT,
            source_forecast_config_bk TEXT
        )
        ON CONFLICT (organization_sk, forecast_detail_bk) DO NOTHING
        RETURNING forecast_detail_pk, forecast_detail_bk, net_cost
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s forecast detail records.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_fact_forecast_details_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_fact_forecast_details_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed forecast detail fact data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_fictitious_organizations_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_fictitious_organizations_for_seeder
*   Version:        3.1 (Audit Trail Certified)
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_organization (
            organization_bk, organization_name, organization_type, parent_organization_sk, is_clerk_managed,
            created_by_user_sk, updated_by_user_sk
        )
        SELECT
            r->>'organization_bk',
            r->>'organization_name',
            (r->>'organization_type')::public.organization_type_enum,
            p_calling_org_sk,
            FALSE,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_array_elements(p_records) r
        ON CONFLICT (organization_bk) DO NOTHING
        RETURNING organization_sk, organization_bk, organization_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'inserted_count', v_inserted_count, 'summary', COALESCE(v_summary, '[]'::jsonb));
END;
$$;


ALTER FUNCTION "public"."create_fictitious_organizations_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_fictitious_organizations_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed fictitious organization data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';




CREATE OR REPLACE FUNCTION "public"."create_fictitious_users_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_fictitious_users_for_seeder
*   Version:        3.1 (Audit Trail Certified)
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_user (
            user_bk, user_name, user_email, organization_sk, is_clerk_managed,
            created_by_user_sk, updated_by_user_sk
        )
        SELECT
            r->>'user_bk',
            r->>'user_name',
            r->>'user_email',
            p_calling_org_sk,
            FALSE,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_array_elements(p_records) r
        ON CONFLICT (organization_sk, user_bk) DO NOTHING
        RETURNING user_sk, user_bk, user_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object('status', 'success', 'inserted_count', v_inserted_count, 'summary', COALESCE(v_summary, '[]'::jsonb));
END;
$$;


ALTER FUNCTION "public"."create_fictitious_users_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_fictitious_users_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed fictitious user data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';


CREATE OR REPLACE FUNCTION "public"."create_forecast_calculation_configs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_forecast_calculation_configs_for_seeder
*   Version:        1.3 (Enum Cast Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version fixes a type mismatch error by explicitly
*                   casting the incoming text `cost_unit` to the new
*                   `public.cost_unit_enum` type.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_forecast_calculation_config (
            organization_sk,
            scenario_configuration_sk,
            activity_sk,
            forecast_config_bk,
            config_name,
            calculation_method,
            cost_unit,
            notes,
            is_active,
            is_group_definition,
            enrollment_curve_type,
            curve_points,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            msc.scenario_configuration_sk,
            da.activity_sk,
            x.forecast_config_bk,
            x.config_name,
            (x.calculation_method)::public.calculation_method_enum,
            x.cost_unit::public.cost_unit_enum, -- THE DEFINITIVE FIX: Explicit cast
            x.notes,
            COALESCE((x.is_active)::boolean, true),
            COALESCE((x.is_group_definition)::boolean, false),
            x.enrollment_curve_type,
            x.curve_points,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            forecast_config_bk TEXT,
            parent_scenario_configuration_bk TEXT,
            activity_bk TEXT,
            config_name TEXT,
            calculation_method TEXT,
            cost_unit TEXT,
            notes TEXT,
            is_active TEXT,
            is_group_definition TEXT,
            enrollment_curve_type TEXT,
            curve_points JSONB
        )
        JOIN public.map_scenario_configuration msc ON x.parent_scenario_configuration_bk = msc.scenario_configuration_bk AND msc.organization_sk = p_calling_org_sk
        LEFT JOIN public.dim_activity da ON x.activity_bk = da.activity_bk AND da.organization_sk = p_calling_org_sk
        ON CONFLICT (scenario_configuration_sk, config_name) DO NOTHING
        RETURNING forecast_config_sk, forecast_config_bk, config_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s forecast calculation configs.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_forecast_calculation_configs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_forecast_calculation_configs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed forecast rule data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';




CREATE OR REPLACE FUNCTION "public"."create_organization_memberships_for_seeder"("p_records" "jsonb") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_organization_memberships_for_seeder
*   Version:        9.0 (Gold Standard - Pure SK Pipe)
*   Author:         Principal Database Architect
*   Description:    This is the definitive "dumb" worker for memberships. It
*                   accepts a payload of pre-hydrated surrogate keys (SKs) from
*                   the orchestrator and performs a direct, high-performance
*                   bulk INSERT with no joins or lookups.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.user_organization_membership (user_sk, organization_sk, role)
        SELECT
            (x.user_sk)::bigint,
            (x.organization_sk)::bigint,
            (x.role)::public.user_role_enum
        FROM jsonb_to_recordset(p_records) AS x(
            user_sk TEXT,
            organization_sk TEXT,
            role TEXT
        )
        ON CONFLICT (user_sk, organization_sk) DO NOTHING
        RETURNING user_organization_membership_sk, user_sk, organization_sk, role
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s membership records.', v_inserted_count),
        'inserted_count', v_inserted_count,
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_organization_memberships_for_seeder"("p_records" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_organization_memberships_for_seeder"("p_records" "jsonb") IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-hydrated membership data. How: Accepts a JSONB payload of records with surrogate keys (SKs) and performs a direct INSERT.';


CREATE OR REPLACE FUNCTION "public"."create_reimbursement_types_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_reimbursement_types_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_reimbursement_type (
            organization_sk,
            reimbursement_type_bk,
            reimbursement_type_name,
            reimbursement_type_code,
            description,
            financial_impact,
            is_default,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.reimbursement_type_bk,
            x.reimbursement_type_name,
            x.reimbursement_type_code,
            x.description,
            (x.financial_impact)::numeric,
            (x.is_default)::boolean,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            reimbursement_type_bk TEXT,
            reimbursement_type_name TEXT,
            reimbursement_type_code TEXT,
            description TEXT,
            financial_impact TEXT,
            is_default TEXT
        )
        ON CONFLICT (organization_sk, reimbursement_type_bk) DO NOTHING
        RETURNING reimbursement_type_sk, reimbursement_type_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s reimbursement types.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_reimbursement_types_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_reimbursement_types_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed reimbursement type data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';


CREATE OR REPLACE FUNCTION "public"."create_scenario_configurations_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_scenario_configurations_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.map_scenario_configuration (
            organization_sk,
            scenario_configuration_bk,
            study_sk,
            scenario_sk,
            start_date,
            end_date,
            target_enrollment,
            target_sites,
            study_status,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.scenario_configuration_bk,
            ds.study_sk,
            dbs.scenario_sk,
            (x.start_date)::date,
            (x.end_date)::date,
            (x.target_enrollment)::integer,
            (x.target_sites)::integer,
            (x.study_status)::public.study_status_enum,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            scenario_configuration_bk TEXT,
            parent_study_bk TEXT,
            parent_scenario_bk TEXT,
            start_date TEXT,
            end_date TEXT,
            target_enrollment TEXT,
            target_sites TEXT,
            study_status TEXT
        )
        JOIN public.dim_study ds ON x.parent_study_bk = ds.study_bk AND ds.organization_sk = p_calling_org_sk
        JOIN public.dim_budget_scenario dbs ON x.parent_scenario_bk = dbs.scenario_bk AND dbs.organization_sk = p_calling_org_sk
        ON CONFLICT (study_sk, scenario_sk) DO NOTHING
        RETURNING scenario_configuration_sk, scenario_configuration_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s scenario configurations.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_scenario_configurations_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_scenario_configurations_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed scenario configuration data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';


CREATE OR REPLACE FUNCTION "public"."create_sites_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_sites_for_seeder
*   Version:        2.0 (Performance Group Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to handle the new nullable
*                   `performance_group_bk` column, allowing sites to be linked
*                   to enrollment curve rules during the seeding process.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_site (
            organization_sk,
            scenario_configuration_sk,
            site_partner_sk,
            site_bk,
            site_status,
            activation_date,
            closeout_date,
            target_enrollment,
            performance_group_bk, -- Added column
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            msc.scenario_configuration_sk,
            msp.study_partner_sk,
            x.site_bk,
            (x.site_status)::public.site_status_enum,
            (x.activation_date)::date,
            (x.closeout_date)::date,
            (x.target_enrollment)::integer,
            x.performance_group_bk, -- Added value
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            site_bk TEXT,
            parent_scenario_configuration_bk TEXT,
            site_partner_bk TEXT,
            site_status TEXT,
            activation_date TEXT,
            closeout_date TEXT,
            target_enrollment TEXT,
            performance_group_bk TEXT -- Added field
        )
        JOIN public.map_scenario_configuration msc ON x.parent_scenario_configuration_bk = msc.scenario_configuration_bk AND msc.organization_sk = p_calling_org_sk
        JOIN public.map_study_partners msp ON x.site_partner_bk = msp.study_partner_bk AND msp.organization_sk = p_calling_org_sk
        ON CONFLICT (scenario_configuration_sk, site_organization_sk) DO NOTHING
        RETURNING site_sk, site_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s site assignments.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_sites_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_sites_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed site assignment data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';


CREATE OR REPLACE FUNCTION "public"."create_studies_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_studies_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_study (
            organization_sk,
            study_bk,
            protocol_number,
            study_title,
            study_short_name,
            therapeutic_area,
            phase,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            x.study_bk,
            x.protocol_number,
            x.study_title,
            x.study_short_name,
            x.therapeutic_area,
            (x.phase)::public.study_phase_enum,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            study_bk TEXT,
            protocol_number TEXT,
            study_title TEXT,
            study_short_name TEXT,
            therapeutic_area TEXT,
            phase TEXT
        )
        ON CONFLICT (organization_sk, study_bk) DO NOTHING
        RETURNING study_sk, study_bk, protocol_number
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s studies.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_studies_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_studies_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed study data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';


CREATE OR REPLACE FUNCTION "public"."create_study_arms_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_study_arms_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_study_arm (
            organization_sk,
            scenario_configuration_sk,
            arm_bk,
            arm_name,
            target_enrollment,
            description,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            msc.scenario_configuration_sk,
            x.arm_bk,
            x.arm_name,
            (x.target_enrollment)::integer,
            x.description,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            arm_bk TEXT,
            parent_scenario_configuration_bk TEXT,
            arm_name TEXT,
            target_enrollment TEXT,
            description TEXT
        )
        JOIN public.map_scenario_configuration msc ON x.parent_scenario_configuration_bk = msc.scenario_configuration_bk AND msc.organization_sk = p_calling_org_sk
        ON CONFLICT (scenario_configuration_sk, arm_name) DO NOTHING
        RETURNING arm_sk, arm_bk, arm_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s study arms.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_study_arms_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_study_arms_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed study arm data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_study_epochs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_study_epochs_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_study_epochs (
            organization_sk,
            arm_sk,
            epoch_bk,
            epoch_name,
            epoch_order,
            description,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            dsa.arm_sk,
            x.epoch_bk,
            x.epoch_name,
            (x.epoch_order)::integer,
            x.description,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            epoch_bk TEXT,
            parent_arm_bk TEXT,
            epoch_name TEXT,
            epoch_order TEXT,
            description TEXT
        )
        JOIN public.dim_study_arm dsa ON x.parent_arm_bk = dsa.arm_bk AND dsa.organization_sk = p_calling_org_sk
        ON CONFLICT (arm_sk, epoch_name) DO NOTHING
        RETURNING epoch_sk, epoch_bk, epoch_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s study epochs.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_study_epochs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_study_epochs_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed study epoch data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';


CREATE OR REPLACE FUNCTION "public"."create_study_partners_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_study_partners_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.map_study_partners (
            organization_sk,
            scenario_configuration_sk,
            partner_organization_sk,
            study_partner_bk,
            partner_role,
            is_primary,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            msc.scenario_configuration_sk,
            d_org.organization_sk,
            x.study_partner_bk,
            (x.partner_role)::public.organization_type_enum,
            (x.is_primary)::boolean,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            study_partner_bk TEXT,
            parent_scenario_configuration_bk TEXT,
            partner_organization_bk TEXT,
            partner_role TEXT,
            is_primary TEXT
        )
        JOIN public.map_scenario_configuration msc ON x.parent_scenario_configuration_bk = msc.scenario_configuration_bk AND msc.organization_sk = p_calling_org_sk
        JOIN public.dim_organization d_org ON x.partner_organization_bk = d_org.organization_bk AND d_org.parent_organization_sk = p_calling_org_sk
        ON CONFLICT (scenario_configuration_sk, partner_organization_sk, partner_role) DO NOTHING
        RETURNING study_partner_sk, study_partner_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s study partners.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_study_partners_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_study_partners_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed study partner data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_study_staff_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_study_staff_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.map_study_staff (
            organization_sk,
            study_partner_sk,
            user_sk,
            study_staff_bk,
            staff_role,
            is_primary_contact,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            msp.study_partner_sk,
            du.user_sk,
            x.study_staff_bk,
            x.staff_role,
            (x.is_primary_contact)::boolean,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            study_staff_bk TEXT,
            parent_study_partner_bk TEXT,
            staff_user_bk TEXT,
            staff_role TEXT,
            is_primary_contact TEXT
        )
        JOIN public.map_study_partners msp ON x.parent_study_partner_bk = msp.study_partner_bk AND msp.organization_sk = p_calling_org_sk
        JOIN public.dim_user du ON x.staff_user_bk = du.user_bk AND du.organization_sk = p_calling_org_sk
        ON CONFLICT (study_partner_sk, user_sk, staff_role) DO NOTHING
        RETURNING study_staff_sk, study_staff_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s study staff assignments.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_study_staff_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_study_staff_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed study staff data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_study_visit_activities_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_study_visit_activities_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.map_study_visit_activity (
            organization_sk,
            scenario_configuration_sk,
            visit_sk,
            activity_sk,
            map_soa_bk,
            is_required,
            notes,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            msc.scenario_configuration_sk,
            dsv.visit_sk,
            da.activity_sk,
            x.map_soa_bk,
            (x.is_required)::boolean,
            x.notes,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            map_soa_bk TEXT,
            parent_scenario_configuration_bk TEXT,
            visit_bk TEXT,
            activity_bk TEXT,
            is_required TEXT,
            notes TEXT
        )
        JOIN public.map_scenario_configuration msc ON x.parent_scenario_configuration_bk = msc.scenario_configuration_bk AND msc.organization_sk = p_calling_org_sk
        JOIN public.dim_study_visits dsv ON x.visit_bk = dsv.visit_bk AND dsv.organization_sk = p_calling_org_sk
        JOIN public.dim_activity da ON x.activity_bk = da.activity_bk AND da.organization_sk = p_calling_org_sk
        ON CONFLICT (scenario_configuration_sk, visit_sk, activity_sk) DO NOTHING
        RETURNING map_soa_sk, map_soa_bk
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s SoA mappings.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_study_visit_activities_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_study_visit_activities_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed SoA mapping data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';



CREATE OR REPLACE FUNCTION "public"."create_study_visits_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.create_study_visits_for_seeder
*   Version:        1.1 (Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This version is updated to accept and populate audit trail
*                   columns based on the calling user's context.
********************************************************************************/
DECLARE
    v_inserted_count INT;
    v_summary JSONB;
BEGIN
    WITH inserted AS (
        INSERT INTO public.dim_study_visits (
            organization_sk,
            epoch_sk,
            visit_bk,
            visit_name,
            visit_order,
            offset_days,
            offset_window_days,
            description,
            created_by_user_sk,
            updated_by_user_sk
        )
        SELECT
            p_calling_org_sk,
            dse.epoch_sk,
            x.visit_bk,
            x.visit_name,
            (x.visit_order)::integer,
            (x.offset_days)::integer,
            (x.offset_window_days)::integer,
            x.description,
            p_calling_user_sk,
            p_calling_user_sk
        FROM jsonb_to_recordset(p_records) AS x(
            visit_bk TEXT,
            parent_epoch_bk TEXT,
            visit_name TEXT,
            visit_order TEXT,
            offset_days TEXT,
            offset_window_days TEXT,
            description TEXT
        )
        JOIN public.dim_study_epochs dse ON x.parent_epoch_bk = dse.epoch_bk AND dse.organization_sk = p_calling_org_sk
        ON CONFLICT (epoch_sk, visit_name) DO NOTHING
        RETURNING visit_sk, visit_bk, visit_name
    )
    SELECT count(*), jsonb_agg(i) INTO v_inserted_count, v_summary FROM inserted i;

    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success',
        'message', format('Processed batch. Inserted: %s study visits.', v_inserted_count),
        'inserted_count', COALESCE(v_inserted_count, 0),
        'summary', COALESCE(v_summary, '[]'::jsonb)
    );
END;
$$;


ALTER FUNCTION "public"."create_study_visits_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."create_study_visits_for_seeder"("p_records" "jsonb", "p_calling_org_sk" bigint, "p_calling_user_sk" bigint) IS 'SECURITY DEFINER worker for the seeder orchestrator. Why: High-performance bulk insertion of pre-transformed study visit data. How: Accepts a JSONB payload of records with globally unique business keys and audit context, performing a direct INSERT without lookups.';


CREATE OR REPLACE FUNCTION "public"."populate_template_data"("p_study_bk" "text" DEFAULT NULL::"text") RETURNS TABLE("action_report" "jsonb")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       public.populate_template_data
*   Version:        82.0 (Definitive - Audit Trail Certified)
*   Author:         Senior AI Application Developer
*   Description:    This definitive version is now fully compliant with the audit
*                   trail mandate. It captures the calling user's SK from the
*                   JWT context and explicitly passes it down to all seeder
*                   worker functions, ensuring a complete and accurate audit trail.
********************************************************************************/
DECLARE
    -- Core variables
    v_calling_org_sk BIGINT := public.get_current_organization_sk();
    v_calling_user_sk BIGINT := public.get_current_user_sk(); -- THE DEFINITIVE FIX
    v_details JSONB := '{}'::jsonb;
    v_temp_report JSONB;
    v_p1 JSONB; v_p2 JSONB; v_p3 JSONB; v_p4 JSONB; v_p5 JSONB;

    -- Global Key Maps
    v_org_bk_map JSONB; v_user_bk_map JSONB; v_reimb_bk_map JSONB;
    v_budget_cat_bk_map JSONB; v_activity_bk_map JSONB; v_study_bk_map JSONB;
    v_scenario_bk_map JSONB; v_amendment_bk_map JSONB; v_scenario_config_bk_map JSONB;
    v_arm_bk_map JSONB; v_epoch_bk_map JSONB; v_visit_bk_map JSONB; v_site_bk_map JSONB;
    v_partner_bk_map JSONB; v_staff_bk_map JSONB; v_forecast_config_bk_map JSONB;
    v_activity_cost_bk_map JSONB; v_soa_bk_map JSONB; v_enrollment_bk_map JSONB;
    v_forecast_detail_bk_map JSONB;
    v_org_sk_map JSONB; v_user_sk_map JSONB;
    v_final_enrollment_bk_map JSONB;

    -- Filtered Payload Variables
    v_study_bks_to_process TEXT[];
    v_studies_payload JSONB; v_amendments_payload JSONB; v_scenario_configs_payload JSONB;
    v_arms_payload JSONB; v_epochs_payload JSONB; v_visits_payload JSONB; v_sites_payload JSONB;
    v_partners_payload JSONB; v_staff_payload JSONB; v_forecast_configs_payload JSONB;
    v_activity_costs_payload JSONB; v_soa_payload JSONB; v_enrollment_payload JSONB;
    v_forecast_detail_payload JSONB;
    
    -- Transformed Payload Variables
    v_tp1_orgs JSONB; v_tp1_users JSONB; v_tp1_reimbs JSONB; v_tp1_budgetCats JSONB;
    v_tp1_activities JSONB; v_tp1_memberships_hydrated_sk JSONB;
    v_tp2_studies JSONB; v_tp2_scenarios JSONB; v_tp2_amendments JSONB;
    v_tp2_scenario_configs JSONB; v_tp2_arms JSONB; v_tp2_epochs JSONB; v_tp2_visits JSONB;
    v_tp2_sites JSONB; v_tp2_partners JSONB; v_tp2_staff JSONB; v_tp2_forecast_configs JSONB;
    v_tp3_activity_costs JSONB; v_tp3_soa_mappings JSONB;
    v_tp4_enrollments JSONB;
    v_tp5_forecast_details JSONB;

    -- Budget Category Loop variables
    v_categories_to_process JSONB; v_created_cat_bks TEXT[] := ARRAY[]::TEXT[];
    v_pass_records JSONB; v_total_cats_processed INT := 0;
    v_max_iterations INT := 10; v_current_iteration INT := 0;
BEGIN
    IF v_calling_org_sk IS NULL OR v_calling_user_sk IS NULL THEN RAISE EXCEPTION 'Authorization context not found.'; END IF;

    -- PHASE 1: FETCH & CREATE GLOBAL KEY MAPS
    RAISE NOTICE '[Orchestrator v82.0] Phase 1: Fetching and Creating Global Key Maps...';
    v_p1 := public.get_seeder_payload_for_stage(1);
    v_p2 := public.get_seeder_payload_for_stage(2);
    v_p3 := public.get_seeder_payload_for_stage(3);
    v_p4 := public.get_seeder_payload_for_stage(4);
    v_p5 := public.get_seeder_payload_for_stage(5);

    -- Key Map Creation (Unchanged)
    SELECT jsonb_object_agg(o->>'organization_bk', o->>'organization_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_org_bk_map FROM jsonb_array_elements(v_p1->'organizations') o WHERE o->>'organization_bk' IS NOT NULL;
    SELECT jsonb_object_agg(u->>'user_bk', u->>'user_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_user_bk_map FROM jsonb_array_elements(v_p1->'users') u WHERE u->>'user_bk' IS NOT NULL;
    SELECT jsonb_object_agg(rt->>'reimbursement_type_bk', rt->>'reimbursement_type_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_reimb_bk_map FROM jsonb_array_elements(v_p1->'reimbursement_types') rt WHERE rt->>'reimbursement_type_bk' IS NOT NULL;
    SELECT jsonb_object_agg(bc->>'budget_category_bk', bc->>'budget_category_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_budget_cat_bk_map FROM jsonb_array_elements(v_p1->'budget_categories') bc WHERE bc->>'budget_category_bk' IS NOT NULL;
    SELECT jsonb_object_agg(a->>'activity_bk', a->>'activity_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_activity_bk_map FROM jsonb_array_elements(v_p1->'activities') a WHERE a->>'activity_bk' IS NOT NULL;
    SELECT jsonb_object_agg(s->>'study_bk', s->>'study_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_study_bk_map FROM jsonb_array_elements(v_p2->'studies') s WHERE s->>'study_bk' IS NOT NULL;
    SELECT jsonb_object_agg(s->>'scenario_bk', s->>'scenario_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_scenario_bk_map FROM jsonb_array_elements(v_p2->'scenarios') s WHERE s->>'scenario_bk' IS NOT NULL;
    SELECT jsonb_object_agg(a->>'amendment_bk', a->>'amendment_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_amendment_bk_map FROM jsonb_array_elements(v_p2->'amendments') a WHERE a->>'amendment_bk' IS NOT NULL;
    SELECT jsonb_object_agg(sc->>'scenario_configuration_bk', sc->>'scenario_configuration_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_scenario_config_bk_map FROM jsonb_array_elements(v_p2->'scenario_configurations') sc WHERE sc->>'scenario_configuration_bk' IS NOT NULL;
    SELECT jsonb_object_agg(a->>'arm_bk', a->>'arm_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_arm_bk_map FROM jsonb_array_elements(v_p2->'study_arms') a WHERE a->>'arm_bk' IS NOT NULL;
    SELECT jsonb_object_agg(e->>'epoch_bk', e->>'epoch_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_epoch_bk_map FROM jsonb_array_elements(v_p2->'study_epochs') e WHERE e->>'epoch_bk' IS NOT NULL;
    SELECT jsonb_object_agg(v->>'visit_bk', v->>'visit_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_visit_bk_map FROM jsonb_array_elements(v_p2->'study_visits') v WHERE v->>'visit_bk' IS NOT NULL;
    SELECT jsonb_object_agg(s->>'site_bk', s->>'site_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_site_bk_map FROM jsonb_array_elements(v_p2->'sites') s WHERE s->>'site_bk' IS NOT NULL;
    SELECT jsonb_object_agg(p->>'study_partner_bk', p->>'study_partner_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_partner_bk_map FROM jsonb_array_elements(v_p2->'study_partners') p WHERE p->>'study_partner_bk' IS NOT NULL;
    SELECT jsonb_object_agg(ss->>'study_staff_bk', ss->>'study_staff_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_staff_bk_map FROM jsonb_array_elements(v_p2->'study_staff') ss WHERE ss->>'study_staff_bk' IS NOT NULL;
    SELECT jsonb_object_agg(fc->>'config_bk', fc->>'config_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_forecast_config_bk_map FROM jsonb_array_elements(v_p2->'forecast_calculation_configs') fc WHERE fc->>'config_bk' IS NOT NULL;
    SELECT jsonb_object_agg(ac->>'cost_bk', ac->>'cost_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_activity_cost_bk_map FROM jsonb_array_elements(v_p3->'activity_costs') ac WHERE ac->>'cost_bk' IS NOT NULL;
    SELECT jsonb_object_agg(sm->>'map_soa_bk', sm->>'map_soa_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_soa_bk_map FROM jsonb_array_elements(v_p3->'soa_mappings') sm WHERE sm->>'map_soa_bk' IS NOT NULL;
    SELECT jsonb_object_agg(e->>'enrollment_bk', e->>'enrollment_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_enrollment_bk_map FROM jsonb_array_elements(v_p4->'fact_enrollment') e WHERE e->>'enrollment_bk' IS NOT NULL;
    SELECT jsonb_object_agg(fd->>'forecast_detail_bk', fd->>'forecast_detail_bk' || '-' || upper(substr(uuid_generate_v4()::text,1,4))) INTO v_forecast_detail_bk_map FROM jsonb_array_elements(v_p5->'fact_forecast_detail') fd WHERE fd->>'forecast_detail_bk' IS NOT NULL;

    -- PHASE 1.5: FILTER PAYLOADS BASED ON PARAMETER (Unchanged)
    RAISE NOTICE '[Orchestrator v82.0] Phase 1.5: Filtering Payloads...';
    IF p_study_bk IS NOT NULL THEN v_study_bks_to_process := ARRAY[p_study_bk]; ELSE SELECT array_agg(s->>'study_bk') INTO v_study_bks_to_process FROM jsonb_array_elements(v_p2->'studies') s; END IF;
    SELECT COALESCE(jsonb_agg(s), '[]'::jsonb) INTO v_studies_payload FROM jsonb_array_elements(v_p2->'studies') s WHERE s->>'study_bk' = ANY(v_study_bks_to_process);
    SELECT COALESCE(jsonb_agg(a), '[]'::jsonb) INTO v_amendments_payload FROM jsonb_array_elements(v_p2->'amendments') a WHERE a->>'parent_study_bk' = ANY(v_study_bks_to_process);
    SELECT COALESCE(jsonb_agg(sc), '[]'::jsonb) INTO v_scenario_configs_payload FROM jsonb_array_elements(v_p2->'scenario_configurations') sc WHERE sc->>'parent_study_bk' = ANY(v_study_bks_to_process);
    SELECT COALESCE(jsonb_agg(a), '[]'::jsonb) INTO v_arms_payload FROM jsonb_array_elements(v_p2->'study_arms') a WHERE a->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);
    SELECT COALESCE(jsonb_agg(e), '[]'::jsonb) INTO v_epochs_payload FROM jsonb_array_elements(v_p2->'study_epochs') e WHERE e->>'parent_arm_bk' IN (SELECT a->>'arm_bk' FROM jsonb_array_elements(v_arms_payload) a);
    SELECT COALESCE(jsonb_agg(v), '[]'::jsonb) INTO v_visits_payload FROM jsonb_array_elements(v_p2->'study_visits') v WHERE v->>'parent_epoch_bk' IN (SELECT e->>'epoch_bk' FROM jsonb_array_elements(v_epochs_payload) e);
    SELECT COALESCE(jsonb_agg(p), '[]'::jsonb) INTO v_partners_payload FROM jsonb_array_elements(v_p2->'study_partners') p WHERE p->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);
    SELECT COALESCE(jsonb_agg(s), '[]'::jsonb) INTO v_sites_payload FROM jsonb_array_elements(v_p2->'sites') s WHERE s->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);
    SELECT COALESCE(jsonb_agg(ss), '[]'::jsonb) INTO v_staff_payload FROM jsonb_array_elements(v_p2->'study_staff') ss WHERE ss->>'parent_study_partner_bk' IN (SELECT p->>'study_partner_bk' FROM jsonb_array_elements(v_partners_payload) p);
    SELECT COALESCE(jsonb_agg(fc), '[]'::jsonb) INTO v_forecast_configs_payload FROM jsonb_array_elements(v_p2->'forecast_calculation_configs') fc WHERE fc->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);
    SELECT COALESCE(jsonb_agg(ac), '[]'::jsonb) INTO v_activity_costs_payload FROM jsonb_array_elements(v_p3->'activity_costs') ac WHERE ac->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);
    SELECT COALESCE(jsonb_agg(sm), '[]'::jsonb) INTO v_soa_payload FROM jsonb_array_elements(v_p3->'soa_mappings') sm WHERE sm->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);
    SELECT COALESCE(jsonb_agg(e), '[]'::jsonb) INTO v_enrollment_payload FROM jsonb_array_elements(v_p4->'fact_enrollment') e WHERE e->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);
    SELECT COALESCE(jsonb_agg(fd), '[]'::jsonb) INTO v_forecast_detail_payload FROM jsonb_array_elements(v_p5->'fact_forecast_detail') fd WHERE fd->>'parent_scenario_configuration_bk' IN (SELECT sc->>'scenario_configuration_bk' FROM jsonb_array_elements(v_scenario_configs_payload) sc);

    -- PHASE 2: BUILD TRANSFORMED PAYLOADS FROM KEY MAPS (Unchanged)
    RAISE NOTICE '[Orchestrator v82.0] Phase 2: Building Transformed Payloads...';
    SELECT COALESCE(jsonb_agg(o || jsonb_build_object('organization_bk', v_org_bk_map->>(o->>'organization_bk'))), '[]'::jsonb) INTO v_tp1_orgs FROM jsonb_array_elements(v_p1->'organizations') o WHERE v_org_bk_map ? (o->>'organization_bk');
    SELECT COALESCE(jsonb_agg(u || jsonb_build_object('user_bk', v_user_bk_map->>(u->>'user_bk'))), '[]'::jsonb) INTO v_tp1_users FROM jsonb_array_elements(v_p1->'users') u WHERE v_user_bk_map ? (u->>'user_bk');
    SELECT COALESCE(jsonb_agg(rt || jsonb_build_object('reimbursement_type_bk', v_reimb_bk_map->>(rt->>'reimbursement_type_bk'))), '[]'::jsonb) INTO v_tp1_reimbs FROM jsonb_array_elements(v_p1->'reimbursement_types') rt WHERE v_reimb_bk_map ? (rt->>'reimbursement_type_bk');
    SELECT COALESCE(jsonb_agg(bc || jsonb_build_object('budget_category_bk', v_budget_cat_bk_map->>(bc->>'budget_category_bk')) || CASE WHEN bc->>'parent_category_bk' IS NOT NULL THEN jsonb_build_object('parent_category_bk', v_budget_cat_bk_map->>(bc->>'parent_category_bk')) ELSE '{}'::jsonb END), '[]'::jsonb) INTO v_tp1_budgetCats FROM jsonb_array_elements(v_p1->'budget_categories') bc WHERE v_budget_cat_bk_map ? (bc->>'budget_category_bk');
    SELECT COALESCE(jsonb_agg(a || jsonb_build_object('activity_bk', v_activity_bk_map->>(a->>'activity_bk')) || jsonb_build_object('budget_category_bk', v_budget_cat_bk_map->>(a->>'budget_category_bk')) || CASE WHEN a->>'reimbursement_type_bk' IS NOT NULL THEN jsonb_build_object('reimbursement_type_bk', v_reimb_bk_map->>(a->>'reimbursement_type_bk')) ELSE '{}'::jsonb END), '[]'::jsonb) INTO v_tp1_activities FROM jsonb_array_elements(v_p1->'activities') a WHERE v_activity_bk_map ? (a->>'activity_bk');
    SELECT COALESCE(jsonb_agg(s || jsonb_build_object('study_bk', v_study_bk_map->>(s->>'study_bk'))), '[]'::jsonb) INTO v_tp2_studies FROM jsonb_array_elements(v_studies_payload) s WHERE v_study_bk_map ? (s->>'study_bk');
    SELECT COALESCE(jsonb_agg(s || jsonb_build_object('scenario_bk', v_scenario_bk_map->>(s->>'scenario_bk'))), '[]'::jsonb) INTO v_tp2_scenarios FROM jsonb_array_elements(v_p2->'scenarios') s WHERE v_scenario_bk_map ? (s->>'scenario_bk');
    SELECT COALESCE(jsonb_agg(a - 'approval_date_offset_days' - 'effective_date_offset_days' || jsonb_build_object('amendment_bk', v_amendment_bk_map->>(a->>'amendment_bk'), 'parent_study_bk', v_study_bk_map->>(a->>'parent_study_bk'), 'approval_date', current_date + ((a->>'approval_date_offset_days')::int * interval '1 day'), 'effective_date', current_date + ((a->>'effective_date_offset_days')::int * interval '1 day'))), '[]'::jsonb) INTO v_tp2_amendments FROM jsonb_array_elements(v_amendments_payload) a WHERE v_amendment_bk_map ? (a->>'amendment_bk');
    SELECT COALESCE(jsonb_agg(sc - 'start_date_offset_days' - 'end_date_offset_days' || jsonb_build_object('scenario_configuration_bk', v_scenario_config_bk_map->>(sc->>'scenario_configuration_bk'), 'parent_study_bk', v_study_bk_map->>(sc->>'parent_study_bk'), 'parent_scenario_bk', v_scenario_bk_map->>(sc->>'parent_scenario_bk'), 'start_date', current_date + ((sc->>'start_date_offset_days')::int * interval '1 day'), 'end_date', current_date + ((sc->>'end_date_offset_days')::int * interval '1 day'))), '[]'::jsonb) INTO v_tp2_scenario_configs FROM jsonb_array_elements(v_scenario_configs_payload) sc WHERE v_scenario_config_bk_map ? (sc->>'scenario_configuration_bk');
    SELECT COALESCE(jsonb_agg(a || jsonb_build_object('arm_bk', v_arm_bk_map->>(a->>'arm_bk'), 'parent_scenario_configuration_bk', v_scenario_config_bk_map->>(a->>'parent_scenario_configuration_bk'))), '[]'::jsonb) INTO v_tp2_arms FROM jsonb_array_elements(v_arms_payload) a WHERE v_arm_bk_map ? (a->>'arm_bk');
    SELECT COALESCE(jsonb_agg(e || jsonb_build_object('epoch_bk', v_epoch_bk_map->>(e->>'epoch_bk'), 'parent_arm_bk', v_arm_bk_map->>(e->>'parent_arm_bk'))), '[]'::jsonb) INTO v_tp2_epochs FROM jsonb_array_elements(v_epochs_payload) e WHERE v_epoch_bk_map ? (e->>'epoch_bk');
    SELECT COALESCE(jsonb_agg(v || jsonb_build_object('visit_bk', v_visit_bk_map->>(v->>'visit_bk'), 'parent_epoch_bk', v_epoch_bk_map->>(v->>'parent_epoch_bk'))), '[]'::jsonb) INTO v_tp2_visits FROM jsonb_array_elements(v_visits_payload) v WHERE v_visit_bk_map ? (v->>'visit_bk');
    SELECT COALESCE(jsonb_agg(p || jsonb_build_object('study_partner_bk', v_partner_bk_map->>(p->>'study_partner_bk'), 'parent_scenario_configuration_bk', v_scenario_config_bk_map->>(p->>'parent_scenario_configuration_bk'), 'partner_organization_bk', v_org_bk_map->>(p->>'partner_organization_bk'))), '[]'::jsonb) INTO v_tp2_partners FROM jsonb_array_elements(v_partners_payload) p WHERE v_partner_bk_map ? (p->>'study_partner_bk');
    -- THE DEFINITIVE FIX: Read from 'performance_group_config_bk' and write to 'performance_group_bk'.
    SELECT COALESCE(jsonb_agg(s - 'activation_date_offset_days' - 'closeout_date_offset_days' - 'performance_group_config_bk' || jsonb_build_object('site_bk', v_site_bk_map->>(s->>'site_bk'), 'parent_scenario_configuration_bk', v_scenario_config_bk_map->>(s->>'parent_scenario_configuration_bk'), 'site_partner_bk', v_partner_bk_map->>(s->>'site_partner_bk'), 'activation_date', current_date + ((s->>'activation_date_offset_days')::int * interval '1 day'), 'closeout_date', current_date + ((s->>'closeout_date_offset_days')::int * interval '1 day')) || CASE WHEN s->>'performance_group_config_bk' IS NOT NULL THEN jsonb_build_object('performance_group_bk', v_forecast_config_bk_map->>(s->>'performance_group_config_bk')) ELSE '{}'::jsonb END), '[]'::jsonb) INTO v_tp2_sites FROM jsonb_array_elements(v_sites_payload) s WHERE v_site_bk_map ? (s->>'site_bk');
    SELECT COALESCE(jsonb_agg(ss || jsonb_build_object('study_staff_bk', v_staff_bk_map->>(ss->>'study_staff_bk'), 'parent_study_partner_bk', v_partner_bk_map->>(ss->>'parent_study_partner_bk'), 'staff_user_bk', v_user_bk_map->>(ss->>'staff_user_bk'))), '[]'::jsonb) INTO v_tp2_staff FROM jsonb_array_elements(v_staff_payload) ss WHERE v_staff_bk_map ? (ss->>'study_staff_bk');
    SELECT COALESCE(jsonb_agg(fc - 'config_bk' || jsonb_build_object('forecast_config_bk', v_forecast_config_bk_map->>(fc->>'config_bk'), 'parent_scenario_configuration_bk', v_scenario_config_bk_map->>(fc->>'parent_scenario_configuration_bk'), 'activity_bk', v_activity_bk_map->>(fc->>'activity_bk'))), '[]'::jsonb) INTO v_tp2_forecast_configs FROM jsonb_array_elements(v_forecast_configs_payload) fc WHERE v_forecast_config_bk_map ? (fc->>'config_bk') AND v_scenario_config_bk_map ? (fc->>'parent_scenario_configuration_bk');
    SELECT COALESCE(jsonb_agg(ac - 'cost_bk' - 'effective_date_offset_days' - 'end_date_offset_days' || jsonb_build_object('activity_cost_bk', v_activity_cost_bk_map->>(ac->>'cost_bk'), 'parent_scenario_configuration_bk', v_scenario_config_bk_map->>(ac->>'parent_scenario_configuration_bk'), 'activity_bk', v_activity_bk_map->>(ac->>'activity_bk'), 'cost_bearing_partner_bk', v_partner_bk_map->>(ac->>'cost_bearing_partner_bk'), 'payer_partner_bk', v_partner_bk_map->>(ac->>'payer_partner_bk'), 'reimbursement_type_bk', v_reimb_bk_map->>(ac->>'reimbursement_type_bk'), 'effective_date', current_date + ((ac->>'effective_date_offset_days')::int * interval '1 day'), 'end_date', CASE WHEN ac->>'end_date_offset_days' IS NOT NULL THEN current_date + ((ac->>'end_date_offset_days')::int * interval '1 day') ELSE NULL END)), '[]'::jsonb) INTO v_tp3_activity_costs FROM jsonb_array_elements(v_activity_costs_payload) ac WHERE v_activity_cost_bk_map ? (ac->>'cost_bk');
    SELECT COALESCE(jsonb_agg(sm || jsonb_build_object('map_soa_bk', v_soa_bk_map->>(sm->>'map_soa_bk'), 'parent_scenario_configuration_bk', v_scenario_config_bk_map->>(sm->>'parent_scenario_configuration_bk'), 'visit_bk', v_visit_bk_map->>(sm->>'visit_bk'), 'activity_bk', v_activity_bk_map->>(sm->>'activity_bk'))), '[]'::jsonb) INTO v_tp3_soa_mappings FROM jsonb_array_elements(v_soa_payload) sm WHERE v_soa_bk_map ? (sm->>'map_soa_bk');
    SELECT COALESCE(jsonb_agg(e - 'snapshot_date_offset_days' || jsonb_build_object('enrollment_bk', v_enrollment_bk_map->>(e->>'enrollment_bk'), 'scenario_configuration_bk', v_scenario_config_bk_map->>(e->>'parent_scenario_configuration_bk'), 'site_bk', (SELECT s->>'site_bk' FROM jsonb_array_elements(v_tp2_sites) s WHERE s->>'parent_scenario_configuration_bk' = v_scenario_config_bk_map->>(e->>'parent_scenario_configuration_bk') AND (s->>'site_bk' LIKE (v_site_bk_map->>(e->>'site_bk')) || '%')), 'study_bk', (SELECT sc->>'parent_study_bk' FROM jsonb_array_elements(v_tp2_scenario_configs) sc WHERE sc->>'scenario_configuration_bk' = v_scenario_config_bk_map->>(e->>'parent_scenario_configuration_bk')), 'scenario_bk', (SELECT sc->>'parent_scenario_bk' FROM jsonb_array_elements(v_tp2_scenario_configs) sc WHERE sc->>'scenario_configuration_bk' = v_scenario_config_bk_map->>(e->>'parent_scenario_configuration_bk')), 'snapshot_date_sk', current_date + (COALESCE(e->>'snapshot_date_offset_days', '0')::int * interval '1 day'))), '[]'::jsonb) INTO v_tp4_enrollments FROM jsonb_array_elements(v_enrollment_payload) e WHERE v_enrollment_bk_map ? (e->>'enrollment_bk');
    SELECT COALESCE(jsonb_object_agg(orig.original_bk, tx.transformed_bk), '{}'::jsonb) INTO v_final_enrollment_bk_map FROM (SELECT e->>'enrollment_bk' as original_bk, (v_enrollment_bk_map->>(e->>'enrollment_bk')) as transformed_bk FROM jsonb_array_elements(v_p4->'fact_enrollment') e) orig JOIN (SELECT enr->>'enrollment_bk' as transformed_bk FROM jsonb_array_elements(v_tp4_enrollments) enr) tx ON orig.transformed_bk = tx.transformed_bk;
    SELECT COALESCE(jsonb_agg(fd - 'snapshot_date_sk' || jsonb_build_object('forecast_detail_bk', v_forecast_detail_bk_map->>(fd->>'forecast_detail_bk'), 'scenario_configuration_bk', v_scenario_config_bk_map->>(fd->>'parent_scenario_configuration_bk'), 'study_bk', v_study_bk_map->>(fd->>'study_bk'), 'scenario_bk', v_scenario_bk_map->>(fd->>'scenario_bk'), 'activity_bk', v_activity_bk_map->>(fd->>'activity_bk'), 'site_bk', v_site_bk_map->>(fd->>'site_bk'), 'payer_partner_bk', v_partner_bk_map->>(fd->>'payer_partner_bk'), 'payee_partner_bk', v_partner_bk_map->>(fd->>'payee_partner_bk'), 'source_arm_bk', v_arm_bk_map->>(fd->>'source_arm_bk'), 'source_epoch_bk', v_epoch_bk_map->>(fd->>'source_epoch_bk'), 'source_visit_bk', v_visit_bk_map->>(fd->>'source_visit_bk'), 'source_enrollment_bk', v_final_enrollment_bk_map->>(fd->>'source_enrollment_bk'), 'source_activity_cost_bk', v_activity_cost_bk_map->>(fd->>'source_activity_cost_bk'), 'source_map_soa_bk', v_soa_bk_map->>(fd->>'source_map_soa_bk'), 'source_forecast_config_bk', v_forecast_config_bk_map->>(fd->>'source_forecast_config_bk'), 'snapshot_date_sk', fd->>'snapshot_date_sk')), '[]'::jsonb) INTO v_tp5_forecast_details FROM jsonb_array_elements(v_forecast_detail_payload) fd WHERE v_forecast_detail_bk_map ? (fd->>'forecast_detail_bk');

    -- PHASE 3 & 4: EXECUTE BASE WORKERS, CAPTURE SKS, HYDRATE
    RAISE NOTICE '[Orchestrator v82.0] Phase 3 & 4: Executing base workers, capturing SKs, and hydrating...';
    SELECT ar.action_report INTO v_temp_report FROM public.create_fictitious_organizations_for_seeder(v_tp1_orgs, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('organizations_seeded', v_temp_report->'inserted_count');
    SELECT jsonb_object_agg(s->>'organization_bk', s->'organization_sk') INTO v_org_sk_map FROM jsonb_array_elements(v_temp_report->'summary') s;
    SELECT ar.action_report INTO v_temp_report FROM public.create_fictitious_users_for_seeder(v_tp1_users, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('users_seeded', v_temp_report->'inserted_count');
    SELECT jsonb_object_agg(s->>'user_bk', s->'user_sk') INTO v_user_sk_map FROM jsonb_array_elements(v_temp_report->'summary') s;
    SELECT COALESCE(jsonb_agg(m - 'user_bk' - 'organization_bk' || jsonb_build_object('user_sk', v_user_sk_map->>(v_user_bk_map->>(m->>'user_bk')), 'organization_sk', v_org_sk_map->>(v_org_bk_map->>(m->>'organization_bk')))), '[]'::jsonb)
    INTO v_tp1_memberships_hydrated_sk
    FROM jsonb_array_elements(v_p1->'memberships') m
    WHERE (v_user_sk_map->>(v_user_bk_map->>(m->>'user_bk'))) IS NOT NULL AND (v_org_sk_map->>(v_org_bk_map->>(m->>'organization_bk'))) IS NOT NULL;

    -- PHASE 5: EXECUTE DEPENDENT WORKERS
    RAISE NOTICE '[Orchestrator v82.0] Phase 5: Executing dependent workers...';
    SELECT ar.action_report INTO v_temp_report FROM public.create_organization_memberships_for_seeder(v_tp1_memberships_hydrated_sk) ar;
    v_details := v_details || jsonb_build_object('memberships_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_reimbursement_types_for_seeder(v_tp1_reimbs, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('reimbursement_types_seeded', v_temp_report->'inserted_count');
    v_categories_to_process := v_tp1_budgetCats;
    WHILE jsonb_array_length(v_categories_to_process) > 0 AND v_current_iteration < v_max_iterations LOOP
        v_current_iteration := v_current_iteration + 1;
        SELECT jsonb_agg(elem) INTO v_pass_records FROM jsonb_array_elements(v_categories_to_process) elem WHERE (elem->>'parent_category_bk' IS NULL) OR (elem->>'parent_category_bk' = ANY(v_created_cat_bks));
        IF v_pass_records IS NULL OR jsonb_array_length(v_pass_records) = 0 THEN EXIT; END IF;
        SELECT ar.action_report INTO v_temp_report FROM public.create_budget_categories_for_seeder(v_pass_records, v_calling_org_sk, v_calling_user_sk) ar;
        v_created_cat_bks := array_cat(v_created_cat_bks, ARRAY(SELECT value->>'budget_category_bk' FROM jsonb_array_elements(v_temp_report->'summary')));
        v_total_cats_processed := v_total_cats_processed + (v_temp_report->>'inserted_count')::int;
        v_categories_to_process := (SELECT COALESCE(jsonb_agg(p.elem), '[]'::jsonb) FROM jsonb_array_elements(v_categories_to_process) p(elem) WHERE NOT (p.elem->>'budget_category_bk' = ANY(v_created_cat_bks)));
    END LOOP;
    v_details := v_details || jsonb_build_object('budget_categories_seeded', v_total_cats_processed);
    SELECT ar.action_report INTO v_temp_report FROM public.create_activities_for_seeder(v_tp1_activities, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('activities_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_studies_for_seeder(v_tp2_studies, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('studies_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_budget_scenarios_for_seeder(v_tp2_scenarios, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('scenarios_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_amendments_for_seeder(v_tp2_amendments, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('amendments_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_scenario_configurations_for_seeder(v_tp2_scenario_configs, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('scenario_configurations_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_study_arms_for_seeder(v_tp2_arms, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('study_arms_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_study_epochs_for_seeder(v_tp2_epochs, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('study_epochs_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_study_visits_for_seeder(v_tp2_visits, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('study_visits_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_study_partners_for_seeder(v_tp2_partners, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('study_partners_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_sites_for_seeder(v_tp2_sites, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('sites_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_study_staff_for_seeder(v_tp2_staff, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('study_staff_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_forecast_calculation_configs_for_seeder(v_tp2_forecast_configs, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('forecast_configs_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_activity_costs_for_seeder(v_tp3_activity_costs, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('activity_costs_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_study_visit_activities_for_seeder(v_tp3_soa_mappings, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('soa_mappings_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_fact_enrollments_for_seeder(v_tp4_enrollments, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('enrollment_seeded', v_temp_report->'inserted_count');
    SELECT ar.action_report INTO v_temp_report FROM public.create_fact_forecast_details_for_seeder(v_tp5_forecast_details, v_calling_org_sk, v_calling_user_sk) ar;
    v_details := v_details || jsonb_build_object('forecast_details_seeded', v_temp_report->'inserted_count');

    -- PHASE 6: ASSEMBLE FINAL REPORT
    RAISE NOTICE '[Orchestrator v82.0] Phase 6: Assembling final report...';
    RETURN QUERY SELECT jsonb_build_object(
        'status', 'success', 
        'message', CASE 
                       WHEN p_study_bk IS NOT NULL THEN format('Targeted Seeding Complete for Study %s.', p_study_bk)
                       ELSE 'Full Template Seeding Complete: All Stages.'
                   END, 
        'summary', v_details
    );
END;
$$;


ALTER FUNCTION "public"."populate_template_data"("p_study_bk" "text") OWNER TO "postgres";


COMMENT ON FUNCTION "public"."populate_template_data"("p_study_bk" "text") IS 'The primary orchestrator for seeding template data into a user''s sandbox. Why: Creates an analysis-ready workspace quickly. How: Executes a multi-stage ETL process that transforms, re-keys, and loads data from the private cache into the user''s sandboxed public tables.';


CREATE OR REPLACE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"() RETURNS "trigger"
    LANGUAGE "plpgsql"
    AS $$
BEGIN
  -- dim_activity
  IF TG_TABLE_NAME = 'dim_activity' THEN
    IF NEW.activity_bk IS NOT NULL AND NEW.activity_bk LIKE 'CTF-%'
       AND (OLD.activity_bk IS DISTINCT FROM NEW.activity_bk) THEN
      IF NEW.activity_name IS NOT NULL AND left(NEW.activity_name,5) = 'TPL: ' THEN
        NEW.activity_name := substr(NEW.activity_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_budget_category
  IF TG_TABLE_NAME = 'dim_budget_category' THEN
    IF NEW.budget_category_bk IS NOT NULL AND NEW.budget_category_bk LIKE 'CTF-%'
       AND (OLD.budget_category_bk IS DISTINCT FROM NEW.budget_category_bk) THEN
      IF NEW.category_name IS NOT NULL AND left(NEW.category_name,5) = 'TPL: ' THEN
        NEW.category_name := substr(NEW.category_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_budget_scenario
  IF TG_TABLE_NAME = 'dim_budget_scenario' THEN
    IF NEW.scenario_bk IS NOT NULL AND NEW.scenario_bk LIKE 'CTF-%'
       AND (OLD.scenario_bk IS DISTINCT FROM NEW.scenario_bk) THEN
      IF NEW.scenario_name IS NOT NULL AND left(NEW.scenario_name,5) = 'TPL: ' THEN
        NEW.scenario_name := substr(NEW.scenario_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_forecast_calculation_config
  IF TG_TABLE_NAME = 'dim_forecast_calculation_config' THEN
    IF NEW.forecast_config_bk IS NOT NULL AND NEW.forecast_config_bk LIKE 'CTF-%'
       AND (OLD.forecast_config_bk IS DISTINCT FROM NEW.forecast_config_bk) THEN
      IF NEW.config_name IS NOT NULL AND left(NEW.config_name,5) = 'TPL: ' THEN
        NEW.config_name := substr(NEW.config_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_organization
  IF TG_TABLE_NAME = 'dim_organization' THEN
    IF NEW.organization_bk IS NOT NULL AND NEW.organization_bk LIKE 'CTF-%'
       AND (OLD.organization_bk IS DISTINCT FROM NEW.organization_bk) THEN
      IF NEW.organization_name IS NOT NULL AND left(NEW.organization_name,5) = 'TPL: ' THEN
        NEW.organization_name := substr(NEW.organization_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_reimbursement_type
  IF TG_TABLE_NAME = 'dim_reimbursement_type' THEN
    IF NEW.reimbursement_type_bk IS NOT NULL AND NEW.reimbursement_type_bk LIKE 'CTF-%'
       AND (OLD.reimbursement_type_bk IS DISTINCT FROM NEW.reimbursement_type_bk) THEN
      IF NEW.reimbursement_type_name IS NOT NULL AND left(NEW.reimbursement_type_name,5) = 'TPL: ' THEN
        NEW.reimbursement_type_name := substr(NEW.reimbursement_type_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_study
  IF TG_TABLE_NAME = 'dim_study' THEN
    IF NEW.study_bk IS NOT NULL AND NEW.study_bk LIKE 'CTF-%'
       AND (OLD.study_bk IS DISTINCT FROM NEW.study_bk) THEN
      IF NEW.protocol_number IS NOT NULL AND left(NEW.protocol_number,5) = 'TPL: ' THEN
        NEW.protocol_number := substr(NEW.protocol_number,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_study_arm
  IF TG_TABLE_NAME = 'dim_study_arm' THEN
    IF NEW.arm_bk IS NOT NULL AND NEW.arm_bk LIKE 'CTF-%'
       AND (OLD.arm_bk IS DISTINCT FROM NEW.arm_bk) THEN
      IF NEW.arm_name IS NOT NULL AND left(NEW.arm_name,5) = 'TPL: ' THEN
        NEW.arm_name := substr(NEW.arm_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_study_epochs
  IF TG_TABLE_NAME = 'dim_study_epochs' THEN
    IF NEW.epoch_bk IS NOT NULL AND NEW.epoch_bk LIKE 'CTF-%'
       AND (OLD.epoch_bk IS DISTINCT FROM NEW.epoch_bk) THEN
      IF NEW.epoch_name IS NOT NULL AND left(NEW.epoch_name,5) = 'TPL: ' THEN
        NEW.epoch_name := substr(NEW.epoch_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_study_visits
  IF TG_TABLE_NAME = 'dim_study_visits' THEN
    IF NEW.visit_bk IS NOT NULL AND NEW.visit_bk LIKE 'CTF-%'
       AND (OLD.visit_bk IS DISTINCT FROM NEW.visit_bk) THEN
      IF NEW.visit_name IS NOT NULL AND left(NEW.visit_name,5) = 'TPL: ' THEN
        NEW.visit_name := substr(NEW.visit_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  -- dim_user
  IF TG_TABLE_NAME = 'dim_user' THEN
    IF NEW.user_bk IS NOT NULL AND NEW.user_bk LIKE 'CTF-%'
       AND (OLD.user_bk IS DISTINCT FROM NEW.user_bk) THEN
      IF NEW.user_name IS NOT NULL AND left(NEW.user_name,5) = 'TPL: ' THEN
        NEW.user_name := substr(NEW.user_name,6);
      END IF;
    END IF;
    RETURN NEW;
  END IF;

  RETURN NEW;
END;
$$;


ALTER FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"() OWNER TO "postgres";








CREATE OR REPLACE FUNCTION "public"."get_seeder_payload_for_stage"("p_stage_number" integer) RETURNS "jsonb"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public'
    AS $$
/********************************************************************************
*   Function:       public.get_seeder_payload_for_stage
*   Version:        2.0 (Gold Standard - Architecturally Pure)
*   Author:         Principal Database Architect
*   Description:    This definitive version acts as a pure, secure gatekeeper.
*                   It reads the pre-built payloads from the materialized cache
*                   for a given stage number and aggregates them into a single
*                   JSONB object for the seeder orchestrator. It performs NO
*                   key transformations, ensuring perfect consistency between
*                   the cache and the consumer.
********************************************************************************/
DECLARE
    v_aggregated_payload JSONB;
BEGIN
    -- This query reads all payloads for the requested stage from the cache
    -- and aggregates them into a single JSONB object.
    -- The key for each payload is derived from the table name by removing
    -- the 'private.template_' prefix, ensuring consistency.
    SELECT jsonb_object_agg(
        -- THE FIX: Derive the key name directly and consistently.
        REPLACE(template_table_name, 'private.template_', ''),
        payload
    )
    INTO v_aggregated_payload
    FROM private.materialized_template_payload
    WHERE stage_number = p_stage_number;

    RETURN COALESCE(v_aggregated_payload, '{}'::jsonb);
END;
$$;


ALTER FUNCTION "public"."get_seeder_payload_for_stage"("p_stage_number" integer) OWNER TO "postgres";


COMMENT ON FUNCTION "public"."get_seeder_payload_for_stage"("p_stage_number" integer) IS 'A secure, SECURITY DEFINER gatekeeper function for the seeder. Why: Provides a safe way for the seeder orchestrator to access pre-calculated, validated template data from the private cache. How: Reads and aggregates all payloads for a given stage number from `private.materialized_template_payload`.';







-- Migration: Adopt All Templates + Clear Reimbursement Defaults (Refactored to use update_* workers)
-- Description:
--   Refactors public.adopt_all_templates() to call existing update_* worker functions
--   per-record, setting BK to NULL through the same API paths the application uses.
--   This triggers CTF-* BK generation and existing business rules/trigger logic safely
--   (avoiding NOT NULL violations). Also clears is_default on ALL reimbursement types.
--   Returns a JSONB action_report with per-table adoption counts.

CREATE OR REPLACE FUNCTION public.adopt_all_templates()
RETURNS TABLE(action_report jsonb)
LANGUAGE plpgsql SECURITY DEFINER
SET search_path TO 'public', 'extensions'
AS $$
/********************************************************************************
*   Function:       public.adopt_all_templates
*   Version:        2.0 (Adopt-All via update_* Workers)
*   Author:         Database Engineering
*   Description:    Converts all template-prefixed (TPL-*) data within the caller's
*                   sandbox into owned records by calling the existing update_*
*                   worker functions and setting BK fields to NULL. This leverages
*                   the same code path that the application uses for adoption so
*                   CTF-* BKs are generated consistently and triggers run (e.g.,
*                   name cleanup).
*
*   Notes:
*     - Scope is limited to current organization sandbox via get_current_organization_sk().
*     - We count attempted adoptions (records meeting TPL-% filter) as adopted.
*     - Reimbursement defaults are cleared sandbox-wide after adoption.
*     - Some worker signatures use p_payload and others use p_update; we pass
*       a single JSONB argument accordingly.
********************************************************************************/
DECLARE
  v_org_sk BIGINT := public.get_current_organization_sk();
  v_summary JSONB := '{}'::jsonb;
  v_count   INT;

  r RECORD;
BEGIN
  IF v_org_sk IS NULL THEN
    RAISE EXCEPTION 'Authorization context not found.';
  END IF;

  -- Organizations (uses p_payload)
  v_count := 0;
  FOR r IN
    SELECT organization_sk
    FROM public.dim_organization
    WHERE parent_organization_sk = v_org_sk
      AND organization_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_fictitious_organizations(
      jsonb_build_object(
        'organization_sk', r.organization_sk,
        'update_fields', jsonb_build_object('organization_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('organizations_adopted', v_count);

  -- Users (uses p_update)
  v_count := 0;
  FOR r IN
    SELECT user_sk
    FROM public.dim_user
    WHERE organization_sk = v_org_sk
      AND is_clerk_managed = FALSE
      AND user_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_fictitious_users(
      jsonb_build_object(
        'user_sk', r.user_sk,
        'update_fields', jsonb_build_object('user_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('users_adopted', v_count);

  -- Reimbursement Types
  v_count := 0;
  FOR r IN
    SELECT reimbursement_type_sk
    FROM public.dim_reimbursement_type
    WHERE organization_sk = v_org_sk
      AND reimbursement_type_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_reimbursement_types(
      jsonb_build_object(
        'reimbursement_type_sk', r.reimbursement_type_sk,
        'update_fields', jsonb_build_object('reimbursement_type_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('reimbursement_types_adopted', v_count);

  -- Budget Categories
  v_count := 0;
  FOR r IN
    SELECT budget_category_sk
    FROM public.dim_budget_category
    WHERE organization_sk = v_org_sk
      AND budget_category_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_budget_categories(
      jsonb_build_object(
        'budget_category_sk', r.budget_category_sk,
        'update_fields', jsonb_build_object('budget_category_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('budget_categories_adopted', v_count);

  -- Activities
  v_count := 0;
  FOR r IN
    SELECT activity_sk
    FROM public.dim_activity
    WHERE organization_sk = v_org_sk
      AND activity_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_activities(
      jsonb_build_object(
        'activity_sk', r.activity_sk,
        'update_fields', jsonb_build_object('activity_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('activities_adopted', v_count);

  -- Studies
  v_count := 0;
  FOR r IN
    SELECT study_sk
    FROM public.dim_study
    WHERE organization_sk = v_org_sk
      AND study_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_studies(
      jsonb_build_object(
        'study_sk', r.study_sk,
        'update_fields', jsonb_build_object('study_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('studies_adopted', v_count);

  -- Scenarios
  v_count := 0;
  FOR r IN
    SELECT scenario_sk
    FROM public.dim_budget_scenario
    WHERE organization_sk = v_org_sk
      AND scenario_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_budget_scenarios(
      jsonb_build_object(
        'budget_scenario_sk', r.scenario_sk,
        'update_fields', jsonb_build_object('scenario_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('scenarios_adopted', v_count);

  -- Amendments
  v_count := 0;
  FOR r IN
    SELECT amendment_sk
    FROM public.dim_amendment
    WHERE organization_sk = v_org_sk
      AND amendment_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_amendments(
      jsonb_build_object(
        'amendment_sk', r.amendment_sk,
        'update_fields', jsonb_build_object('amendment_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('amendments_adopted', v_count);

  -- Scenario Configurations
  v_count := 0;
  FOR r IN
    SELECT scenario_configuration_sk
    FROM public.map_scenario_configuration
    WHERE organization_sk = v_org_sk
      AND scenario_configuration_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_scenario_configurations(
      jsonb_build_object(
        'scenario_configuration_sk', r.scenario_configuration_sk,
        'update_fields', jsonb_build_object('scenario_configuration_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('scenario_configurations_adopted', v_count);

  -- Study Arms
  v_count := 0;
  FOR r IN
    SELECT arm_sk
    FROM public.dim_study_arm
    WHERE organization_sk = v_org_sk
      AND arm_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_study_arms(
      jsonb_build_object(
        'arm_sk', r.arm_sk,
        'update_fields', jsonb_build_object('arm_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('study_arms_adopted', v_count);

  -- Study Epochs
  v_count := 0;
  FOR r IN
    SELECT epoch_sk
    FROM public.dim_study_epochs
    WHERE organization_sk = v_org_sk
      AND epoch_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_study_epochs(
      jsonb_build_object(
        'epoch_sk', r.epoch_sk,
        'update_fields', jsonb_build_object('epoch_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('epochs_adopted', v_count);

  -- Study Visits
  v_count := 0;
  FOR r IN
    SELECT visit_sk
    FROM public.dim_study_visits
    WHERE organization_sk = v_org_sk
      AND visit_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_study_visits(
      jsonb_build_object(
        'visit_sk', r.visit_sk,
        'update_fields', jsonb_build_object('visit_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('visits_adopted', v_count);

  -- Study Partners
  v_count := 0;
  FOR r IN
    SELECT study_partner_sk
    FROM public.map_study_partners
    WHERE organization_sk = v_org_sk
      AND study_partner_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_study_partners(
      jsonb_build_object(
        'study_partner_sk', r.study_partner_sk,
        'update_fields', jsonb_build_object('study_partner_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('study_partners_adopted', v_count);

  -- Sites
  v_count := 0;
  FOR r IN
    SELECT site_sk
    FROM public.dim_site
    WHERE organization_sk = v_org_sk
      AND site_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_sites(
      jsonb_build_object(
        'site_sk', r.site_sk,
        'update_fields', jsonb_build_object('site_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('sites_adopted', v_count);

  -- Study Staff
  v_count := 0;
  FOR r IN
    SELECT study_staff_sk
    FROM public.map_study_staff
    WHERE organization_sk = v_org_sk
      AND study_staff_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_study_staff(
      jsonb_build_object(
        'study_staff_sk', r.study_staff_sk,
        'update_fields', jsonb_build_object('study_staff_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('study_staff_adopted', v_count);

  -- Forecast Rules
  v_count := 0;
  FOR r IN
    SELECT forecast_config_sk
    FROM public.dim_forecast_calculation_config
    WHERE organization_sk = v_org_sk
      AND forecast_config_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_forecast_calculation_configs(
      jsonb_build_object(
        'forecast_config_sk', r.forecast_config_sk,
        'update_fields', jsonb_build_object('forecast_config_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('forecast_rules_adopted', v_count);

  -- Activity Costs
  v_count := 0;
  FOR r IN
    SELECT activity_cost_sk
    FROM public.dim_activity_cost
    WHERE organization_sk = v_org_sk
      AND activity_cost_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_activity_costs(
      jsonb_build_object(
        'activity_cost_sk', r.activity_cost_sk,
        'update_fields', jsonb_build_object('activity_cost_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('activity_costs_adopted', v_count);

  -- Schedule of Activities (SoA)
  v_count := 0;
  FOR r IN
    SELECT map_soa_sk
    FROM public.map_study_visit_activity
    WHERE organization_sk = v_org_sk
      AND map_soa_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_study_visit_activities(
      jsonb_build_object(
        'map_soa_sk', r.map_soa_sk,
        'update_fields', jsonb_build_object('map_soa_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('soa_mappings_adopted', v_count);

  -- Facts: Enrollment
  v_count := 0;
  FOR r IN
    SELECT enrollment_pk
    FROM public.fact_enrollment
    WHERE organization_sk = v_org_sk
      AND enrollment_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_fact_enrollment(
      jsonb_build_object(
        'enrollment_pk', r.enrollment_pk,
        'update_fields', jsonb_build_object('enrollment_bk', NULL)
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('enrollment_facts_adopted', v_count);

  -- Facts: Forecast Detail (flat payload, no nested update_fields)
  v_count := 0;
  FOR r IN
    SELECT forecast_detail_pk
    FROM public.fact_forecast_detail
    WHERE organization_sk = v_org_sk
      AND forecast_detail_bk LIKE 'TPL-%'
      AND is_deleted = FALSE
  LOOP
    PERFORM public.update_fact_forecast_details(
      jsonb_build_object(
        'forecast_detail_pk', r.forecast_detail_pk,
        'forecast_detail_bk', NULL
      )
    );
    v_count := v_count + 1;
  END LOOP;
  v_summary := v_summary || jsonb_build_object('forecast_detail_facts_adopted', v_count);

  -- Clear defaults on ALL reimbursement types (sandbox-wide)
  WITH updated AS (
    UPDATE public.dim_reimbursement_type
       SET is_default = FALSE
     WHERE organization_sk = v_org_sk
       AND is_deleted = FALSE
       AND is_default = TRUE
     RETURNING 1
  )
  SELECT count(*) INTO v_count FROM updated;
  v_summary := v_summary || jsonb_build_object('reimbursement_defaults_cleared', v_count);

  RETURN QUERY
  SELECT jsonb_build_object(
    'status','success',
    'message','All template records adopted via update_* workers; reimbursement defaults cleared.',
    'summary', v_summary
  );
END;
$$;

ALTER FUNCTION public.adopt_all_templates() OWNER TO postgres;

COMMENT ON FUNCTION public.adopt_all_templates()
IS 'Adopts all TPL-* records in the current sandbox by calling update_* workers to set BK=NULL (triggering CTF-* generation) and executing name cleanup via existing rules. Also clears is_default on all reimbursement types. Returns per-table adoption counts and defaults-cleared count.';





-- 2) Preflight function: dashboard + template counts + stage 2 payload
CREATE OR REPLACE FUNCTION public.get_template_preflight_status()
RETURNS jsonb
LANGUAGE plpgsql STABLE SECURITY DEFINER
SET search_path TO 'public'
AS $$
/********************************************************************************
*   Function:       public.get_template_preflight_status
*   Version:        1.0 (Preflight - Dashboard + TPL Counts + Stage2 Payload)
*   Author:         Principal Database Architect
*   Description:    Aggregates sandbox dashboard metrics, template record counts,
*                   and Stage 2 payload (studies/scenarios) to drive the guided
*                   Populate Templates modal. Provides booleans to gate actions.
********************************************************************************/
DECLARE
    v_org_sk BIGINT := public.get_current_organization_sk();
    v_payload JSONB;
    v_dashboard JSONB;
    v_stage2 JSONB;
    v_tpl_counts JSONB;
    v_has_tpl BOOLEAN;
BEGIN
    IF v_org_sk IS NULL THEN
        RAISE EXCEPTION 'Authorization context not found.';
    END IF;

    -- Existing dashboard snapshot (includes live+total counts and quick lists)
    v_dashboard := public.get_sandbox_dashboard_metrics();

   -- Available template catalog for Stage 2 (studies/scenarios/configs)
    v_stage2 := public.get_seeder_payload_for_stage(2);

    -- Count existing template (TPL-*) data within this sandbox
    SELECT jsonb_build_object(
        'organizations',           COALESCE((SELECT count(*) FROM public.dim_organization                 WHERE parent_organization_sk = v_org_sk AND organization_bk            LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'users',                   COALESCE((SELECT count(*) FROM public.dim_user                        WHERE organization_sk = v_org_sk AND user_bk                         LIKE 'TPL-%' AND is_clerk_managed = FALSE AND is_deleted = FALSE), 0),
        'reimbursement_types',     COALESCE((SELECT count(*) FROM public.dim_reimbursement_type          WHERE organization_sk = v_org_sk AND reimbursement_type_bk           LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'budget_categories',       COALESCE((SELECT count(*) FROM public.dim_budget_category             WHERE organization_sk = v_org_sk AND budget_category_bk              LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'activities',              COALESCE((SELECT count(*) FROM public.dim_activity                    WHERE organization_sk = v_org_sk AND activity_bk                     LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'studies',                 COALESCE((SELECT count(*) FROM public.dim_study                       WHERE organization_sk = v_org_sk AND study_bk                        LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'scenarios',               COALESCE((SELECT count(*) FROM public.dim_budget_scenario             WHERE organization_sk = v_org_sk AND scenario_bk                     LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'scenario_configurations', COALESCE((SELECT count(*) FROM public.map_scenario_configuration      WHERE organization_sk = v_org_sk AND scenario_configuration_bk       LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'study_arms',              COALESCE((SELECT count(*) FROM public.dim_study_arm                   WHERE organization_sk = v_org_sk AND arm_bk                          LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'epochs',                  COALESCE((SELECT count(*) FROM public.dim_study_epochs                WHERE organization_sk = v_org_sk AND epoch_bk                        LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'visits',                  COALESCE((SELECT count(*) FROM public.dim_study_visits                WHERE organization_sk = v_org_sk AND visit_bk                        LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'soa_mappings',            COALESCE((SELECT count(*) FROM public.map_study_visit_activity        WHERE organization_sk = v_org_sk AND map_soa_bk                      LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'sites',                   COALESCE((SELECT count(*) FROM public.dim_site                        WHERE organization_sk = v_org_sk AND site_bk                         LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'study_partners',          COALESCE((SELECT count(*) FROM public.map_study_partners              WHERE organization_sk = v_org_sk AND study_partner_bk                LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'study_staff',             COALESCE((SELECT count(*) FROM public.map_study_staff                 WHERE organization_sk = v_org_sk AND study_staff_bk                  LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'forecast_rules',          COALESCE((SELECT count(*) FROM public.dim_forecast_calculation_config WHERE organization_sk = v_org_sk AND forecast_config_bk              LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'activity_costs',          COALESCE((SELECT count(*) FROM public.dim_activity_cost               WHERE organization_sk = v_org_sk AND activity_cost_bk                LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'enrollment_facts',        COALESCE((SELECT count(*) FROM public.fact_enrollment                 WHERE organization_sk = v_org_sk AND enrollment_bk                   LIKE 'TPL-%' AND is_deleted = FALSE), 0),
        'forecast_detail_facts',   COALESCE((SELECT count(*) FROM public.fact_forecast_detail            WHERE organization_sk = v_org_sk AND forecast_detail_bk              LIKE 'TPL-%' AND is_deleted = FALSE), 0)
    ) INTO v_tpl_counts;

    -- Determine gating booleans
    v_has_tpl := EXISTS (
        SELECT 1
        FROM jsonb_each_text(v_tpl_counts) AS kv(key, val)
        WHERE (val)::int > 0
    );

    v_payload := jsonb_build_object(
        'dashboard', v_dashboard,
        'tpl_counts', v_tpl_counts,
        'has_template_data', v_has_tpl,
        'gate_ok_to_populate', NOT v_has_tpl,
        'stage2_payload', v_stage2
    );

    RETURN v_payload;
END;
$$;

ALTER FUNCTION public.get_template_preflight_status() OWNER TO postgres;

COMMENT ON FUNCTION public.get_template_preflight_status()
IS 'Preflight status for the Populate Templates workflow. Why: Combines sandbox dashboard, template data presence, and available template studies/scenarios to gate actions. How: Returns dashboard (jsonb), tpl_counts (jsonb), has_template_data (bool), gate_ok_to_populate (bool), stage2_payload (jsonb).';



CREATE OR REPLACE FUNCTION "private"."cache_template_data_as_json"() RETURNS "void"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'private', 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       private.cache_template_data_as_json
*   Version:        15.0 (BI Lineage Persistence)
*   Author:         Senior AI Business Intelligence Architect
*   Description:    This version is updated to persist the full data lineage,
*                   including the original_study_bk and original_scenario_bk,
*                   into the private.template_fact_forecast_detail table.
********************************************************************************/
DECLARE
    v_blueprint_payload JSONB;
    v_rec RECORD;
    v_payload JSONB;
    v_metadata JSONB;
    v_all_enrollment_facts JSONB;
    v_all_forecast_facts JSONB;
    v_corrected_enrollment_facts JSONB;
    v_corrected_forecast_facts JSONB;
BEGIN
    RAISE NOTICE '[CACHE ENGINE v15.0] Starting full data materialization...';

    -- Step 1: Build the initial blueprint from all non-fact template tables.
    SELECT jsonb_build_object(
        'organizations', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_organizations t WHERE t.include_in_json),
        'users', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_users t WHERE t.include_in_json),
        'memberships', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_memberships t WHERE t.include_in_json),
        'reimbursement_types', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_reimbursement_types t WHERE t.include_in_json),
        'activities', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_activities t WHERE t.include_in_json),
        'budget_categories', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_budget_categories t WHERE t.include_in_json),
        'studies', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_studies t WHERE t.include_in_json),
        'scenarios', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_scenarios t WHERE t.include_in_json),
        'sites', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_sites t WHERE t.include_in_json),
        'arms', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_study_arms t WHERE t.include_in_json),
        'epochs', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_study_epochs t WHERE t.include_in_json),
        'visits', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_study_visits t WHERE t.include_in_json),
        'amendments', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_amendments t WHERE t.include_in_json),
        'scenario_configurations', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_scenario_configurations t WHERE t.include_in_json),
        'study_partners', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_study_partners t WHERE t.include_in_json),
        'soa_mappings', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_soa_mappings t WHERE t.include_in_json),
        'activity_costs', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_activity_costs t WHERE t.include_in_json),
        'forecast_configs', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_forecast_calculation_configs t WHERE t.include_in_json),
        'study_staff', (SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) FROM private.template_study_staff t WHERE t.include_in_json)
    ) INTO v_blueprint_payload;

    -- Step 2: Calculate all fact data
    SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_all_enrollment_facts FROM private.calculate_template_enrollment_for_portfolio() t;
    SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_all_forecast_facts FROM private.calculate_template_forecast_for_portfolio(v_all_enrollment_facts) t;

    -- Step 3: Persist the calculated facts into their template tables.
    TRUNCATE private.template_fact_enrollment, private.template_fact_forecast_detail;

    INSERT INTO private.template_fact_enrollment (snapshot_date_offset_days, site_bk, projected_enrollment_count, actual_enrollment_count, is_asserted_actual, last_edit_source, enrollment_bk, parent_scenario_configuration_bk)
    SELECT (r->>'snapshot_date_offset_days')::integer, r->>'site_bk', (r->>'projected_enrollment_count')::integer, (r->>'actual_enrollment_count')::integer, (r->>'is_asserted_actual')::boolean, r->>'last_edit_source', r->>'enrollment_bk', r->>'parent_scenario_configuration_bk'
    FROM jsonb_array_elements(v_all_enrollment_facts) r;

    INSERT INTO private.template_fact_forecast_detail (
        snapshot_date_sk, site_bk, activity_bk, cost_unit, unit_cost, forecast_units, payer_partner_bk, payee_partner_bk,
        reimbursement_factor, source_visit_bk, source_enrollment_bk, source_epoch_bk, source_arm_bk, source_forecast_config_bk,
        is_asserted_actual, last_edit_source, source_activity_cost_bk,
        forecast_detail_bk, parent_scenario_configuration_bk, source_map_soa_bk,
        -- THE DEFINITIVE FIX: Persist the original keys for BI.
        original_study_bk, original_scenario_bk
    )
    SELECT
        (r->>'snapshot_date_sk')::date, r->>'site_bk', r->>'activity_bk', r->>'cost_unit', (r->>'unit_cost')::numeric, (r->>'forecast_units')::numeric,
        r->>'payer_partner_bk', r->>'payee_partner_bk', (r->>'reimbursement_factor')::numeric,
        r->>'source_visit_bk', r->>'source_enrollment_bk', r->>'source_epoch_bk', r->>'source_arm_bk',
        r->>'source_forecast_config_bk', (r->>'is_asserted_actual')::boolean, r->>'last_edit_source', r->>'source_activity_cost_bk',
        r->>'forecast_detail_bk', r->>'parent_scenario_configuration_bk',
        r->>'source_map_soa_bk',
        r->>'original_study_bk', r->>'original_scenario_bk'
    FROM jsonb_array_elements(v_all_forecast_facts) r;

    -- Step 4 & 5: Re-read facts and build final payload
    SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_corrected_enrollment_facts FROM private.template_fact_enrollment t;
    SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_corrected_forecast_facts FROM private.template_fact_forecast_detail t;
    v_blueprint_payload := v_blueprint_payload || jsonb_build_object('fact_enrollment', v_corrected_enrollment_facts, 'fact_forecast_detail', v_corrected_forecast_facts);

    -- Step 6: Materialize the complete blueprint payload
    TRUNCATE private.materialized_template_payload;

    FOR v_rec IN
        SELECT * FROM ( VALUES
            ('private.template_organizations', 'public.dim_organization', 'organizations'),
            ('private.template_users', 'public.dim_user', 'users'),
            ('private.template_memberships', 'public.user_organization_membership', 'memberships'),
            ('private.template_reimbursement_types', 'public.dim_reimbursement_type', 'reimbursement_types'),
            ('private.template_activities', 'public.dim_activity', 'activities'),
            ('private.template_budget_categories', 'public.dim_budget_category', 'budget_categories'),
            ('private.template_studies', 'public.dim_study', 'studies'),
            ('private.template_scenarios', 'public.dim_budget_scenario', 'scenarios'),
            ('private.template_scenario_configurations', 'public.map_scenario_configuration', 'scenario_configurations'),
            ('private.template_sites', 'public.dim_site', 'sites'),
            ('private.template_study_staff', 'public.map_study_staff', 'study_staff'),
            ('private.template_amendments', 'public.dim_amendment', 'amendments'),
            ('private.template_study_arms', 'public.dim_study_arm', 'arms'),
            ('private.template_study_epochs', 'public.dim_study_epochs', 'epochs'),
            ('private.template_study_visits', 'public.dim_study_visits', 'visits'),
            ('private.template_study_partners', 'public.map_study_partners', 'study_partners'),
            ('private.template_forecast_calculation_configs', 'public.dim_forecast_calculation_config', 'forecast_configs'),
            ('private.template_soa_mappings', 'public.map_study_visit_activity', 'soa_mappings'),
            ('private.template_activity_costs', 'public.dim_activity_cost', 'activity_costs'),
            ('private.template_fact_enrollment', 'public.fact_enrollment', 'fact_enrollment'),
            ('private.template_fact_forecast_detail', 'public.fact_forecast_detail', 'fact_forecast_detail')
        ) AS v(template_table_name, public_table_name, payload_key)
    LOOP
        v_payload := v_blueprint_payload->v_rec.payload_key;
        v_metadata := jsonb_build_object(
            'total_record_count', CASE WHEN jsonb_typeof(v_payload) = 'array' THEN jsonb_array_length(v_payload) ELSE 0 END
        );
        INSERT INTO private.materialized_template_payload (template_table_name, public_table_name, payload, metadata)
        VALUES (v_rec.template_table_name, v_rec.public_table_name, v_payload, v_metadata);
    END LOOP;

    -- Step 7: Run validation and BI engines
    PERFORM private.validate_failures();
    PERFORM private.generate_gaps_report();
    PERFORM private.generate_business_checks();

    -- Step 8: Refresh the dictionary view
    REFRESH MATERIALIZED VIEW private.template_metadata_dictionary;
    RAISE NOTICE '[CACHE ENGINE v15.0] Full refresh and validation process completed successfully.';
END;
$$;


ALTER FUNCTION "private"."cache_template_data_as_json"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."cache_template_data_as_json"() IS 'Orchestrator that materializes a complete, analysis-ready template data payload into `private.materialized_template_payload`. Why: Creates a consistent, pre-calculated snapshot used by the seeder, validation engines, and BI reporting, decoupling heavy computation from user-facing actions.';



CREATE OR REPLACE FUNCTION "private"."calculate_template_enrollment"("p_scenario_config_data" "jsonb", "p_sites_data" "jsonb", "p_visits_data" "jsonb", "p_calc_rules_data" "jsonb") RETURNS TABLE("snapshot_date_sk" "date", "site_bk" "text", "projected_enrollment_count" integer, "actual_enrollment_count" integer, "is_asserted_actual" boolean, "last_edit_source" "text")
    LANGUAGE "plpgsql"
    AS $$
    /********************************************************************************
    *   Function:       private.calculate_template_enrollment
    *   Version:        7.0 (Configuration-Centric Certified)
    *   Description:    This version is aligned with the new data model. It now
    *                   receives a simpler, pre-sliced payload focused on a single
    *                   scenario configuration.
    ********************************************************************************/
    DECLARE
        rec_site RECORD;
        v_rule RECORD;
        v_curve_points JSONB;
        v_monthly_enrollments INT[];
        v_remainders NUMERIC[];
        v_total_floored_enrollment INT;
        v_remainder_to_distribute INT;
        v_projection_start_date DATE;
        v_min_offset_days INT;
        v_true_activity_start_date DATE;
        v_site_target_enrollment INT;
    BEGIN
        DROP TABLE IF EXISTS temp_enrollment_results;
        CREATE TEMP TABLE temp_enrollment_results (
            snapshot_date_sk date, site_bk text,
            projected_enrollment_count integer, actual_enrollment_count integer,
            is_asserted_actual boolean, last_edit_source text
        ) ON COMMIT DROP;

        v_projection_start_date := current_date + ((p_scenario_config_data->>'start_date_offset_days')::int * interval '1 day');
        SELECT COALESCE(MIN((v->>'offset_days')::int), 0) INTO v_min_offset_days FROM jsonb_array_elements(p_visits_data) v;

        FOR rec_site IN SELECT * FROM jsonb_to_recordset(p_sites_data) AS x(
            site_bk text, site_organization_bk text, target_enrollment integer,
            activation_date_offset_days int, closeout_date_offset_days int,
            performance_group_config_bk text
        )
        WHERE (x.target_enrollment)::int > 0 AND x.activation_date_offset_days IS NOT NULL AND x.closeout_date_offset_days IS NOT NULL
        LOOP
            v_site_target_enrollment := (rec_site.target_enrollment)::int;

            v_curve_points := NULL;
            IF rec_site.performance_group_config_bk IS NOT NULL THEN
                SELECT c.value->'curve_points' INTO v_curve_points
                FROM jsonb_array_elements(p_calc_rules_data) c
                WHERE c.value->>'config_bk' = rec_site.performance_group_config_bk;
            END IF;

            IF v_curve_points IS NOT NULL THEN
                v_monthly_enrollments := ARRAY[]::INT[]; v_remainders := ARRAY[]::NUMERIC[];
                FOR v_rule IN SELECT * FROM jsonb_array_elements(v_curve_points) LOOP
                    DECLARE v_exact_value NUMERIC := v_site_target_enrollment * ((v_rule.value->>'percent_of_total')::numeric / 100.00);
                    BEGIN v_monthly_enrollments[(v_rule.value->>'month_number')::int] := floor(v_exact_value); v_remainders[(v_rule.value->>'month_number')::int] := v_exact_value - floor(v_exact_value); END;
                END LOOP;
                SELECT COALESCE(SUM(v), 0) INTO v_total_floored_enrollment FROM unnest(v_monthly_enrollments) v WHERE v IS NOT NULL;
                v_remainder_to_distribute := v_site_target_enrollment - v_total_floored_enrollment;
                IF v_remainder_to_distribute > 0 THEN
                    FOR i IN 1..v_remainder_to_distribute LOOP
                        DECLARE v_max_rem_idx INT;
                        BEGIN SELECT idx INTO v_max_rem_idx FROM unnest(v_remainders) WITH ORDINALITY AS t(rem, idx) WHERE t.rem IS NOT NULL ORDER BY rem DESC, idx ASC LIMIT 1;
                            IF v_max_rem_idx IS NOT NULL THEN v_monthly_enrollments[v_max_rem_idx] := v_monthly_enrollments[v_max_rem_idx] + 1; v_remainders[v_max_rem_idx] := -1; END IF;
                        END;
                    END LOOP;
                END IF;
                FOR i IN 1..array_length(v_monthly_enrollments, 1) LOOP
                    IF v_monthly_enrollments[i] IS NOT NULL AND v_monthly_enrollments[i] > 0 THEN
                        INSERT INTO temp_enrollment_results VALUES (date_trunc('month', v_projection_start_date + (i - 1) * interval '1 month')::date, rec_site.site_bk, v_monthly_enrollments[i], 0, FALSE, 'SYSTEM_CALCULATION');
                    END IF;
                END LOOP;
            ELSE
                DECLARE
                    v_duration_months INT; v_activation_date DATE; v_closeout_date DATE; v_monthly_enrollment INT; v_remainder INT;
                BEGIN
                    v_true_activity_start_date := (current_date + (rec_site.activation_date_offset_days * INTERVAL '1 day')) + (v_min_offset_days * INTERVAL '1 day');
                    v_activation_date := GREATEST(date_trunc('month', v_true_activity_start_date)::date, date_trunc('month', v_projection_start_date)::date);
                    v_closeout_date := date_trunc('month', current_date + (rec_site.closeout_date_offset_days * INTERVAL '1 day'))::date;
                    v_duration_months := GREATEST(1, ((EXTRACT(YEAR FROM v_closeout_date) - EXTRACT(YEAR FROM v_activation_date)) * 12 + (EXTRACT(MONTH FROM v_closeout_date) - EXTRACT(MONTH FROM v_activation_date)) + 1));
                    v_monthly_enrollment := floor(v_site_target_enrollment / v_duration_months);
                    v_remainder := v_site_target_enrollment % v_duration_months;
                    FOR i IN 0..v_duration_months-1 LOOP
                        DECLARE v_enroll_this_month INT := v_monthly_enrollment;
                        BEGIN
                            IF v_remainder > 0 THEN v_enroll_this_month := v_enroll_this_month + 1; v_remainder := v_remainder - 1; END IF;
                            IF v_enroll_this_month > 0 THEN
                                INSERT INTO temp_enrollment_results VALUES (date_trunc('month', v_activation_date + i * interval '1 month')::date, rec_site.site_bk, v_enroll_this_month, 0, FALSE, 'SYSTEM_CALCULATION');
                            END IF;
                        END;
                    END LOOP;
                END;
            END IF;
        END LOOP;

        RETURN QUERY SELECT ter.snapshot_date_sk, ter.site_bk, SUM(ter.projected_enrollment_count)::integer, SUM(ter.actual_enrollment_count)::integer, (array_agg(ter.is_asserted_actual))[1], (array_agg(ter.last_edit_source))[1]
        FROM temp_enrollment_results ter GROUP BY ter.snapshot_date_sk, ter.site_bk;
    END;
    $$;


ALTER FUNCTION "private"."calculate_template_enrollment"("p_scenario_config_data" "jsonb", "p_sites_data" "jsonb", "p_visits_data" "jsonb", "p_calc_rules_data" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "private"."calculate_template_enrollment"("p_scenario_config_data" "jsonb", "p_sites_data" "jsonb", "p_visits_data" "jsonb", "p_calc_rules_data" "jsonb") IS 'A worker function that computes projected monthly enrollment for a given set of sites. Why: Provides the foundational time-phased subject data required for schedule-based forecasting. How: Applies either a curve-based distribution or a uniform spread across the site''s activation window.';



CREATE OR REPLACE FUNCTION "private"."calculate_template_enrollment_for_portfolio"() RETURNS TABLE("snapshot_date_offset_days" integer, "site_bk" "text", "projected_enrollment_count" integer, "actual_enrollment_count" integer, "is_asserted_actual" boolean, "last_edit_source" "text", "enrollment_bk" "text", "parent_scenario_configuration_bk" "text")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'private', 'extensions'
    AS $$
/********************************************************************************
*   Function:       private.calculate_template_enrollment_for_portfolio
*   Version:        5.1 (Corrected Data Contract)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version fixes a critical bug by correctly
*                   handling the data contract with its worker function. It now
*                   selects the correct `snapshot_date_sk` column and calculates
*                   the `snapshot_date_offset_days` for its own output, resolving
*                   the "column does not exist" error.
********************************************************************************/
DECLARE
    v_config_rec RECORD;
    v_scenario_config_data JSONB;
    v_sites_data JSONB;
    v_visits_data JSONB;
    v_calc_rules_data JSONB;
BEGIN
    FOR v_config_rec IN
        SELECT * FROM private.template_scenario_configurations
        WHERE include_in_json = TRUE
    LOOP
        -- Gather the data "slice" for the current configuration
        SELECT to_jsonb(t) INTO v_scenario_config_data FROM private.template_scenario_configurations t WHERE t.scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_sites_data FROM private.template_sites t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(v), '[]'::jsonb) INTO v_visits_data FROM private.template_study_visits v JOIN private.template_study_epochs e ON v.parent_epoch_bk = e.epoch_bk JOIN private.template_study_arms a ON e.parent_arm_bk = a.arm_bk WHERE a.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_calc_rules_data FROM private.template_forecast_calculation_configs t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;

        -- Call the worker and transform its output to match this function's contract.
        RETURN QUERY
        SELECT
            -- THE DEFINITIVE FIX: Calculate the offset from the returned date.
            (wr.snapshot_date_sk - current_date)::integer AS snapshot_date_offset_days,
            wr.site_bk,
            wr.projected_enrollment_count,
            wr.actual_enrollment_count,
            wr.is_asserted_actual,
            wr.last_edit_source,
            'TPL-ENR-' || extensions.uuid_generate_v4()::text AS enrollment_bk,
            v_config_rec.scenario_configuration_bk AS parent_scenario_configuration_bk
        FROM private.calculate_template_enrollment(
            p_scenario_config_data   := v_scenario_config_data,
            p_sites_data             := v_sites_data,
            p_visits_data            := v_visits_data,
            p_calc_rules_data        := v_calc_rules_data
        ) AS wr; -- Alias the function call results as `wr`
    END LOOP;
END;
$$;


ALTER FUNCTION "private"."calculate_template_enrollment_for_portfolio"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."calculate_template_enrollment_for_portfolio"() IS 'The portfolio-level orchestrator for generating all template enrollment facts. Why: Iterates through all scenario configurations to build a complete set of enrollment projections for the entire template library. How: Calls the `calculate_template_enrollment` worker for each configuration slice.';



CREATE OR REPLACE FUNCTION "private"."calculate_template_forecast"("p_scenario_config_data" "jsonb", "p_partners_data" "jsonb", "p_calc_rules_data" "jsonb", "p_soa_data" "jsonb", "p_visits_data" "jsonb", "p_costs_data" "jsonb", "p_reimbs_data" "jsonb", "p_enrollment_facts" "jsonb", "p_sites_data" "jsonb", "p_arms_data" "jsonb") RETURNS TABLE("snapshot_date_sk" "date", "site_bk" "text", "activity_bk" "text", "cost_unit" "text", "unit_cost" numeric, "forecast_units" numeric, "payer_partner_bk" "text", "payee_partner_bk" "text", "reimbursement_factor" numeric, "source_visit_bk" "text", "source_enrollment_bk" "text", "source_epoch_bk" "text", "source_arm_bk" "text", "source_forecast_config_bk" "text", "is_asserted_actual" boolean, "last_edit_source" "text", "source_activity_cost_bk" "text", "source_map_soa_bk" "text")
    LANGUAGE "plpgsql"
    AS $$
/********************************************************************************
*   Function:       private.calculate_template_forecast (Worker)
*   Version:        15.0 (Public Parity - Unweighted Fan-Out)
*   Description:    Aligned with the public forecast engine. Removes arm-weighted
*                   allocation and fans out site-level enrollment equally across
*                   all arms' visits for the configuration. Keeps cost resolution,
*                   SoA wiring, and rule-based pathways unchanged.
********************************************************************************/
DECLARE
    v_study_start_date DATE;
    v_study_end_date DATE;
    v_total_enrolled BIGINT;
    v_total_sites BIGINT;
    v_primary_sponsor_partner_bk TEXT;
    v_config_rule RECORD;
    v_activity_cost_rec RECORD;
    v_current_config_bk TEXT := p_scenario_config_data->>'scenario_configuration_bk';
BEGIN
    DROP TABLE IF EXISTS temp_forecast_events;
    CREATE TEMP TABLE temp_forecast_events (
        snapshot_date_sk date, site_bk text, activity_bk text,
        cost_unit text, unit_cost numeric, forecast_units numeric,
        payer_partner_bk text, payee_partner_bk text, reimbursement_factor numeric,
        source_visit_bk text, source_enrollment_bk text, source_epoch_bk text, source_arm_bk text,
        source_forecast_config_bk text, is_asserted_actual boolean, last_edit_source text,
        source_activity_cost_bk text, source_map_soa_bk text
    ) ON COMMIT DROP;

    v_study_start_date := current_date + ((p_scenario_config_data->>'start_date_offset_days')::int * interval '1 day');
    v_study_end_date := current_date + ((p_scenario_config_data->>'end_date_offset_days')::int * interval '1 day');
    v_total_enrolled := (p_scenario_config_data->>'target_enrollment')::bigint;
    v_total_sites := (SELECT count(*) FROM jsonb_array_elements(p_sites_data));

    SELECT p.value->>'study_partner_bk' INTO v_primary_sponsor_partner_bk
    FROM jsonb_array_elements(p_partners_data) p
    WHERE p.value->>'partner_role' = 'Sponsor' AND (p.value->>'is_primary')::boolean = TRUE
    LIMIT 1;

    -- ========================================================================
    -- SCHEDULE_BASED EVENTS (PUBLIC PARITY - UNWEIGHTED FAN-OUT)
    -- ========================================================================
    WITH arms AS (
        SELECT a.arm_bk
        FROM jsonb_to_recordset(p_arms_data) AS a(arm_bk text)
    )
    INSERT INTO temp_forecast_events
    SELECT
        (
            (current_date + (fe.snapshot_date_offset_days * INTERVAL '1 day'))
            + (v.offset_days * INTERVAL '1 day')
        )::date,
        fe.site_bk,
        soa.activity_bk,
        c.cost_unit,
        c.unit_cost,
        -- Public parity: do not weight by arm; use site-level enrollment count directly
        (fe.projected_enrollment_count)::numeric AS forecast_units,
        COALESCE(c.payer_partner_bk, v_primary_sponsor_partner_bk),
        c.cost_bearing_partner_bk,
        COALESCE(rt.financial_impact::numeric, 1.0000),
        v.visit_bk,
        fe.enrollment_bk,
        v.parent_epoch_bk,
        v.arm_bk,
        cfg.config_bk,
        false,
        'SYSTEM_CALCULATION',
        c.cost_bk,
        soa.map_soa_bk
    FROM jsonb_to_recordset(p_enrollment_facts) AS fe(
        snapshot_date_offset_days int, site_bk text, projected_enrollment_count integer, enrollment_bk text
    )
    CROSS JOIN arms a
    JOIN jsonb_to_recordset(p_visits_data) AS v(visit_bk text, offset_days int, parent_epoch_bk text, arm_bk text)
      ON a.arm_bk = v.arm_bk
    JOIN jsonb_to_recordset(p_soa_data) AS soa(visit_bk text, activity_bk text, map_soa_bk text, parent_scenario_configuration_bk text)
      ON v.visit_bk = soa.visit_bk
     AND soa.parent_scenario_configuration_bk = v_current_config_bk
    JOIN jsonb_to_recordset(p_calc_rules_data) AS cfg(config_bk text, activity_bk text, calculation_method text)
      ON soa.activity_bk = cfg.activity_bk
     AND cfg.calculation_method = 'SCHEDULE_BASED'
    JOIN jsonb_to_recordset(p_costs_data) AS c(
        cost_bk text, activity_bk text, cost_bearing_partner_bk text, cost_unit text, unit_cost numeric,
        reimbursement_type_bk text, effective_date_offset_days int, end_date_offset_days int, payer_partner_bk text,
        parent_scenario_configuration_bk text
    )
      ON cfg.activity_bk = c.activity_bk
     AND c.parent_scenario_configuration_bk = v_current_config_bk
    LEFT JOIN jsonb_to_recordset(p_reimbs_data) AS rt(reimbursement_type_bk text, financial_impact text)
      ON c.reimbursement_type_bk = rt.reimbursement_type_bk
    WHERE
      (
        (current_date + (fe.snapshot_date_offset_days * INTERVAL '1 day'))
        + (v.offset_days * INTERVAL '1 day')
      ) >= (current_date + (COALESCE(c.effective_date_offset_days, 0) * INTERVAL '1 day'))
      AND (
        c.end_date_offset_days IS NULL
        OR (
          (current_date + (fe.snapshot_date_offset_days * INTERVAL '1 day'))
          + (v.offset_days * INTERVAL '1 day')
        ) <= (current_date + (c.end_date_offset_days * INTERVAL '1 day'))
      );

    -- ========================================================================
    -- RULE_BASED EVENTS (UNCHANGED LOGIC)
    -- ========================================================================
    FOR v_config_rule IN SELECT * FROM jsonb_to_recordset(p_calc_rules_data) AS cfg(config_bk text, activity_bk text, calculation_method text) WHERE cfg.calculation_method IN ('DIRECT_MONTHLY', 'ENROLLMENT_BASED')
    LOOP
        SELECT * INTO v_activity_cost_rec FROM jsonb_to_recordset(p_costs_data) AS c(cost_bk text, activity_bk text, unit_cost numeric, cost_unit text, cost_bearing_partner_bk text, reimbursement_type_bk text, payer_partner_bk text, parent_scenario_configuration_bk text) WHERE c.activity_bk = v_config_rule.activity_bk AND c.parent_scenario_configuration_bk = v_current_config_bk LIMIT 1;
        IF FOUND THEN
            DECLARE v_reimbursement_factor NUMERIC; v_payer_bk TEXT;
            BEGIN
                SELECT (rt.financial_impact)::numeric INTO v_reimbursement_factor FROM jsonb_to_recordset(p_reimbs_data) AS rt(reimbursement_type_bk text, financial_impact text) WHERE rt.reimbursement_type_bk = v_activity_cost_rec.reimbursement_type_bk;
                v_reimbursement_factor := COALESCE(v_reimbursement_factor, 1.0000);
                v_payer_bk := COALESCE(v_activity_cost_rec.payer_partner_bk, v_primary_sponsor_partner_bk);
                IF v_config_rule.calculation_method = 'DIRECT_MONTHLY' THEN
                    DECLARE v_iter_month DATE; v_duration_months INT := GREATEST(1, ((EXTRACT(YEAR FROM v_study_end_date) - EXTRACT(YEAR FROM v_study_start_date)) * 12 + (EXTRACT(MONTH FROM v_study_end_date) - EXTRACT(MONTH FROM v_study_start_date)) + 1));
                    BEGIN
                        FOR i IN 0..v_duration_months-1 LOOP
                            v_iter_month := (date_trunc('month', v_study_start_date)::date + (i * interval '1 month'))::date;
                            INSERT INTO temp_forecast_events VALUES (v_iter_month, NULL, v_config_rule.activity_bk, v_activity_cost_rec.cost_unit, v_activity_cost_rec.unit_cost, 1, v_payer_bk, v_activity_cost_rec.cost_bearing_partner_bk, v_reimbursement_factor, NULL, NULL, NULL, NULL, v_config_rule.config_bk, false, 'SYSTEM_CALCULATION', v_activity_cost_rec.cost_bk, NULL);
                        END LOOP;
                    END;
                ELSIF v_config_rule.calculation_method = 'ENROLLMENT_BASED' THEN
                    DECLARE v_units NUMERIC;
                    BEGIN
                        IF v_activity_cost_rec.cost_unit = 'Per Site' THEN v_units := v_total_sites; ELSE v_units := v_total_enrolled; END IF;
                        IF v_units > 0 THEN
                            INSERT INTO temp_forecast_events VALUES (date_trunc('month', v_study_start_date)::date, NULL, v_config_rule.activity_bk, v_activity_cost_rec.cost_unit, v_activity_cost_rec.unit_cost, v_units, v_payer_bk, v_activity_cost_rec.cost_bearing_partner_bk, v_reimbursement_factor, NULL, NULL, NULL, NULL, v_config_rule.config_bk, false, 'SYSTEM_CALCULATION', v_activity_cost_rec.cost_bk, NULL);
                        END IF;
                    END;
                END IF;
            END;
        END IF;
    END LOOP;

    RETURN QUERY SELECT tfe.snapshot_date_sk, tfe.site_bk, tfe.activity_bk, tfe.cost_unit, tfe.unit_cost, tfe.forecast_units, tfe.payer_partner_bk, tfe.payee_partner_bk, tfe.reimbursement_factor, tfe.source_visit_bk, tfe.source_enrollment_bk, tfe.source_epoch_bk, tfe.source_arm_bk, tfe.source_forecast_config_bk, tfe.is_asserted_actual, tfe.last_edit_source, tfe.source_activity_cost_bk, tfe.source_map_soa_bk FROM temp_forecast_events tfe;
END;
$$;


ALTER FUNCTION "private"."calculate_template_forecast"("p_scenario_config_data" "jsonb", "p_partners_data" "jsonb", "p_calc_rules_data" "jsonb", "p_soa_data" "jsonb", "p_visits_data" "jsonb", "p_costs_data" "jsonb", "p_reimbs_data" "jsonb", "p_enrollment_facts" "jsonb", "p_sites_data" "jsonb", "p_arms_data" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "private"."calculate_template_forecast"("p_scenario_config_data" "jsonb", "p_partners_data" "jsonb", "p_calc_rules_data" "jsonb", "p_soa_data" "jsonb", "p_visits_data" "jsonb", "p_costs_data" "jsonb", "p_reimbs_data" "jsonb", "p_enrollment_facts" "jsonb", "p_sites_data" "jsonb", "p_arms_data" "jsonb") IS 'The core worker of the forecast "compiler". Why: Transforms a high-level blueprint slice into low-level financial and operational events. How: Implements schedule-based, rule-based, and enrollment-based logic, including the critical Weighted Distribution Mandate for arm enrollment.';



CREATE OR REPLACE FUNCTION "private"."calculate_template_forecast_for_portfolio"("p_all_enrollment_facts" "jsonb") RETURNS TABLE("snapshot_date_sk" "date", "site_bk" "text", "activity_bk" "text", "cost_unit" "text", "unit_cost" numeric, "forecast_units" numeric, "payer_partner_bk" "text", "payee_partner_bk" "text", "reimbursement_factor" numeric, "source_visit_bk" "text", "source_enrollment_bk" "text", "source_epoch_bk" "text", "source_arm_bk" "text", "source_forecast_config_bk" "text", "is_asserted_actual" boolean, "last_edit_source" "text", "source_activity_cost_bk" "text", "forecast_detail_bk" "text", "parent_scenario_configuration_bk" "text", "source_map_soa_bk" "text", "original_study_bk" "text", "original_scenario_bk" "text")
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'private', 'extensions'
    AS $$
/********************************************************************************
*   Function:       private.calculate_template_forecast_for_portfolio (Orchestrator)
*   Version:        12.0 (Definitive - BI Lineage)
*   Description:    This definitive version is updated to return the original
*                   study_bk and scenario_bk alongside the calculated data. This
*                   provides the BI engine with the necessary lineage to perform
*                   correct joins against the original template tables.
********************************************************************************/
DECLARE
    v_config_rec RECORD;
    v_scenario_config_data JSONB;
    v_partners_data JSONB;
    v_calc_rules_data JSONB;
    v_soa_data JSONB;
    v_visits_data JSONB;
    v_costs_data JSONB;
    v_reimbs_data JSONB;
    v_enrollment_facts_slice JSONB;
    v_sites_data JSONB;
    v_arms_data JSONB;
BEGIN
    SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_reimbs_data FROM private.template_reimbursement_types t;
    FOR v_config_rec IN SELECT * FROM private.template_scenario_configurations WHERE include_in_json = TRUE
    LOOP
        SELECT to_jsonb(v_config_rec) INTO v_scenario_config_data;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_partners_data FROM private.template_study_partners t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_calc_rules_data FROM private.template_forecast_calculation_configs t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_soa_data FROM private.template_soa_mappings t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_costs_data FROM private.template_activity_costs t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_sites_data FROM private.template_sites t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        SELECT COALESCE(jsonb_agg(t), '[]'::jsonb) INTO v_arms_data FROM private.template_study_arms t WHERE t.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;
        
        SELECT COALESCE(jsonb_agg(elem), '[]'::jsonb) INTO v_enrollment_facts_slice 
        FROM jsonb_array_elements(p_all_enrollment_facts) AS elem 
        WHERE elem->>'parent_scenario_configuration_bk' = v_config_rec.scenario_configuration_bk;

        SELECT COALESCE(jsonb_agg(
            jsonb_build_object(
                'visit_bk', v.visit_bk,
                'offset_days', v.offset_days,
                'parent_epoch_bk', v.parent_epoch_bk,
                'arm_bk', a.arm_bk
            )
        ), '[]'::jsonb)
        INTO v_visits_data
        FROM private.template_study_visits v
        JOIN private.template_study_epochs e ON v.parent_epoch_bk = e.epoch_bk
        JOIN private.template_study_arms a ON e.parent_arm_bk = a.arm_bk
        WHERE a.parent_scenario_configuration_bk = v_config_rec.scenario_configuration_bk;

        RETURN QUERY
        SELECT
            wr.snapshot_date_sk, wr.site_bk, wr.activity_bk, wr.cost_unit, wr.unit_cost, wr.forecast_units,
            wr.payer_partner_bk, wr.payee_partner_bk, wr.reimbursement_factor, wr.source_visit_bk, wr.source_enrollment_bk,
            wr.source_epoch_bk, wr.source_arm_bk,
            wr.source_forecast_config_bk, wr.is_asserted_actual, wr.last_edit_source, wr.source_activity_cost_bk,
            'TPL-FCT-' || uuid_generate_v4()::text AS forecast_detail_bk,
            v_config_rec.scenario_configuration_bk AS parent_scenario_configuration_bk,
            wr.source_map_soa_bk,
            -- THE DEFINITIVE FIX: Return the original BKs for BI joins.
            v_config_rec.parent_study_bk AS original_study_bk,
            v_config_rec.parent_scenario_bk AS original_scenario_bk
        FROM private.calculate_template_forecast(
            p_scenario_config_data  := v_scenario_config_data, p_partners_data := v_partners_data, p_calc_rules_data := v_calc_rules_data,
            p_soa_data := v_soa_data, p_visits_data := v_visits_data, p_costs_data := v_costs_data, p_reimbs_data := v_reimbs_data, p_enrollment_facts := v_enrollment_facts_slice,
            p_sites_data := v_sites_data, p_arms_data := v_arms_data
        ) AS wr;
    END LOOP;
END;
$$;


ALTER FUNCTION "private"."calculate_template_forecast_for_portfolio"("p_all_enrollment_facts" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "private"."calculate_template_forecast_for_portfolio"("p_all_enrollment_facts" "jsonb") IS 'The portfolio-level orchestrator for generating all template forecast detail facts. Why: Iterates through all scenario configurations to build the complete financial and operational ledger for the template library. How: Calls the `calculate_template_forecast` worker for each configuration slice.';



CREATE OR REPLACE FUNCTION "private"."generate_business_checks"() RETURNS "void"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'private'
    AS $$
/********************************************************************************
*   Function:       private.generate_business_checks
*   Version:        21.0 (Definitive - BI Lineage Certified)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version fixes the final bug in the BI
*                   pipeline. It correctly builds the v_costs_cte using the
*                   `original_study_bk` and `original_scenario_bk` columns,
*                   ensuring all downstream joins succeed and the BI report
*                   is generated with the correct data.
********************************************************************************/
DECLARE
    v_materialized_row RECORD;
    v_blueprint_payload JSONB;
    v_biz_checks JSONB;
    v_costs_cte JSONB;
BEGIN
    RAISE NOTICE '[BUSINESS LOGIC ENGINE v21.0] Starting...';

    -- Create a single, global payload for cross-table joins
    SELECT jsonb_object_agg(REPLACE(REPLACE(template_table_name, 'private.template_', ''), '_', ''), payload)
    INTO v_blueprint_payload
    FROM private.materialized_template_payload;

    -- THE DEFINITIVE FIX: Build the CTE using the original keys for joining.
    SELECT COALESCE(jsonb_agg(c), '[]'::jsonb)
    INTO v_costs_cte
    FROM (
        SELECT
            fct->>'original_study_bk' as study_bk,
            fct->>'original_scenario_bk' as scenario_bk,
            fct->>'activity_bk' as activity_bk,
            fct->>'payee_partner_bk' as payee_partner_bk,
            fct->>'source_activity_cost_bk' as source_activity_cost_bk,
            to_char((fct->>'snapshot_date_sk')::date, 'YYYY-MM') AS month,
            ((fct->>'forecast_units')::numeric * (fct->>'unit_cost')::numeric * (fct->>'reimbursement_factor')::numeric) AS net_cost,
            ((fct->>'forecast_units')::numeric * (fct->>'unit_cost')::numeric) AS forecast_cost
        FROM jsonb_array_elements(v_blueprint_payload->'factforecastdetail') fct
    ) c;


    FOR v_materialized_row IN SELECT * FROM private.materialized_template_payload LOOP
        v_biz_checks := '{}'::jsonb;

        CASE v_materialized_row.template_table_name
            WHEN 'private.template_scenarios' THEN
                WITH scenario_costs AS (
                    SELECT c->>'scenario_bk' as scenario_bk, SUM((c->>'net_cost')::numeric) AS total_net_cost, SUM((c->>'forecast_cost')::numeric) AS total_forecast_cost
                    FROM jsonb_array_elements(v_costs_cte) c GROUP BY 1
                )
                SELECT jsonb_build_object('scenario_total_costs', (
                    SELECT COALESCE(jsonb_object_agg(s->>'scenario_name', jsonb_build_object('net_cost', sc.total_net_cost, 'forecast_cost', sc.total_forecast_cost) ORDER BY sc.total_net_cost DESC), '{}'::jsonb)
                    FROM jsonb_array_elements(v_materialized_row.payload) s
                    JOIN scenario_costs sc ON s->>'scenario_bk' = sc.scenario_bk
                )) INTO v_biz_checks;
            
            -- ... (all other CASE statements remain unchanged) ...

            WHEN 'private.template_study_staff' THEN
                WITH role_distribution AS ( SELECT staff.staff_role, COUNT(*) AS role_count FROM private.template_study_staff staff WHERE staff.include_in_json = true GROUP BY 1 ),
                primary_contact_coverage AS ( SELECT s.study_short_name, COUNT(*) FILTER (WHERE staff.is_primary_contact = true) AS primary_contact_count FROM private.template_study_staff staff JOIN private.template_study_partners sp ON staff.parent_study_partner_bk = sp.study_partner_bk JOIN private.template_scenario_configurations cfg ON sp.parent_scenario_configuration_bk = cfg.scenario_configuration_bk JOIN private.template_studies s ON cfg.parent_study_bk = s.study_bk WHERE staff.include_in_json = true GROUP BY 1 )
                SELECT jsonb_build_object( 'staff_role_distribution', (SELECT jsonb_agg(rd.* ORDER BY rd.role_count DESC) FROM role_distribution rd), 'primary_contact_coverage_check', (SELECT jsonb_agg(pcc.* ORDER BY pcc.study_short_name) FROM primary_contact_coverage pcc) ) INTO v_biz_checks;
            WHEN 'private.template_budget_categories' THEN
                WITH category_spend AS ( SELECT act.budget_category_bk, SUM(c.net_cost) as total_net_cost FROM jsonb_to_recordset(v_costs_cte) AS c(study_bk text, scenario_bk text, activity_bk text, payee_partner_bk text, source_activity_cost_bk text, month text, net_cost numeric, forecast_cost numeric) JOIN private.template_activities act ON c.activity_bk = act.activity_bk GROUP BY 1 ),
                top_categories AS ( SELECT cs.budget_category_bk, cs.total_net_cost FROM category_spend cs ORDER BY cs.total_net_cost DESC LIMIT 5 ),
                category_composition AS ( SELECT act.budget_category_bk, act.activity_name, SUM(c.net_cost) as activity_net_cost, row_number() OVER (PARTITION BY act.budget_category_bk ORDER BY SUM(c.net_cost) DESC) as rn FROM jsonb_to_recordset(v_costs_cte) AS c(study_bk text, scenario_bk text, activity_bk text, payee_partner_bk text, source_activity_cost_bk text, month text, net_cost numeric, forecast_cost numeric) JOIN private.template_activities act ON c.activity_bk = act.activity_bk WHERE act.budget_category_bk IN (SELECT budget_category_bk FROM top_categories) GROUP BY 1, 2 )
                SELECT jsonb_build_object( 'portfolio_spend_by_category', ( SELECT COALESCE(jsonb_agg( jsonb_build_object( 'category_name', bc.category_name, 'total_net_cost', cs.total_net_cost ) ORDER BY cs.total_net_cost DESC ), '[]'::jsonb) FROM category_spend cs JOIN private.template_budget_categories bc ON cs.budget_category_bk = bc.budget_category_bk ), 'top_5_category_composition', ( SELECT COALESCE(jsonb_object_agg( bc.category_name, (SELECT jsonb_agg(jsonb_build_object('activity_name', cc.activity_name, 'activity_net_cost', cc.activity_net_cost) ORDER BY cc.activity_net_cost DESC) FROM category_composition cc WHERE cc.budget_category_bk = tc.budget_category_bk AND cc.rn <= 3) ), '{}'::jsonb) FROM top_categories tc JOIN private.template_budget_categories bc ON tc.budget_category_bk = bc.budget_category_bk ), 'unused_categories', ( SELECT COALESCE(jsonb_agg(jsonb_build_object('category_name', bc.category_name, 'category_bk', bc.budget_category_bk)), '[]'::jsonb) FROM private.template_budget_categories bc WHERE NOT EXISTS (SELECT 1 FROM category_spend cs WHERE cs.budget_category_bk = bc.budget_category_bk) ) ) INTO v_biz_checks;
            WHEN 'private.template_fact_forecast_detail' THEN
                WITH scenario_scoped_costs AS ( SELECT * FROM jsonb_to_recordset(v_costs_cte) as x(study_bk text, scenario_bk text, activity_bk text, payee_partner_bk text, source_activity_cost_bk text, month text, net_cost numeric, forecast_cost numeric) ),
                spend_by_payee AS ( SELECT o.organization_name as payee_name, SUM(ssc.net_cost) as total_net_cost FROM scenario_scoped_costs ssc JOIN private.template_study_partners p ON ssc.payee_partner_bk = p.study_partner_bk JOIN private.template_organizations o ON p.partner_organization_bk = o.organization_bk GROUP BY 1 ),
                cross_scenario_variance AS ( SELECT s.study_short_name, jsonb_object_agg(scen.scenario_name, totals.total_net_cost ORDER BY scen.scenario_name) as scenario_costs FROM ( SELECT study_bk, scenario_bk, SUM(net_cost) as total_net_cost FROM scenario_scoped_costs GROUP BY 1, 2 ) as totals JOIN private.template_studies s ON totals.study_bk = s.study_bk JOIN private.template_scenarios scen ON totals.scenario_bk = scen.scenario_bk GROUP BY 1 HAVING count(*) > 1 )
                SELECT v_biz_checks || jsonb_build_object( 'spend_by_payee_portfolio_wide', ( SELECT COALESCE(jsonb_agg(jsonb_build_object('payee_name', sbp.payee_name, 'total_net_cost', sbp.total_net_cost) ORDER BY sbp.total_net_cost DESC), '[]'::jsonb) FROM spend_by_payee sbp ), 'cross_scenario_variance_analysis', ( SELECT COALESCE(jsonb_agg(csv.*), '[]'::jsonb) FROM cross_scenario_variance csv ) ) INTO v_biz_checks;
            WHEN 'private.template_amendments' THEN
                WITH scenario_costs AS ( SELECT c->>'scenario_bk' as scenario_bk, SUM((c->>'net_cost')::numeric) AS total_net_cost, SUM((c->>'forecast_cost')::numeric) AS total_forecast_cost FROM jsonb_array_elements(v_costs_cte) c GROUP BY 1 )
                SELECT jsonb_build_object('amendment_financial_impact_analysis', ( SELECT COALESCE(jsonb_agg(p ORDER BY (p.triggered_scenario_total_cost->>'net_cost')::numeric DESC NULLS LAST), '[]'::jsonb) FROM ( SELECT a.amendment_version_id, a.summary, COALESCE(s.scenario_name, 'No Scenario Triggered') as triggered_scenario_name, COALESCE(jsonb_build_object('net_cost', sc.total_net_cost, 'forecast_cost', sc.total_forecast_cost), jsonb_build_object('net_cost', 0.00, 'forecast_cost', 0.00)) as triggered_scenario_total_cost FROM private.template_amendments a LEFT JOIN private.template_scenarios s ON a.amendment_bk = s.triggering_amendment_bk LEFT JOIN scenario_costs sc ON s.scenario_bk = sc.scenario_bk ) p )) INTO v_biz_checks;
            WHEN 'private.template_reimbursement_types' THEN
                WITH usage_counts AS ( SELECT c.reimbursement_type_bk as bk, count(*) as usage_count FROM private.template_activity_costs c WHERE c.reimbursement_type_bk IS NOT NULL AND c.include_in_json = true GROUP BY 1 ),
                financial_impacts AS ( SELECT ac.reimbursement_type_bk as bk, SUM((c->>'net_cost')::numeric) as total_net_value, SUM((c->>'forecast_cost')::numeric) as total_gross_value FROM jsonb_array_elements(v_costs_cte) c JOIN private.template_activity_costs ac ON c->>'source_activity_cost_bk' = ac.cost_bk WHERE ac.reimbursement_type_bk IS NOT NULL AND ac.include_in_json = true GROUP BY 1 )
                SELECT jsonb_build_object('reimbursement_rule_summary', ( SELECT COALESCE(jsonb_agg(p ORDER BY usage_count DESC, total_forecasted_net_value DESC), '[]'::jsonb) FROM ( SELECT rt.reimbursement_type_name AS rule_name, rt.financial_impact, COALESCE(uc.usage_count, 0) AS usage_count, COALESCE(fi.total_net_value, 0.00) AS total_forecasted_net_value, COALESCE(fi.total_gross_value, 0.00) AS total_forecasted_gross_value FROM private.template_reimbursement_types rt LEFT JOIN usage_counts uc ON rt.reimbursement_type_bk = uc.bk LEFT JOIN financial_impacts fi ON rt.reimbursement_type_bk = fi.bk WHERE rt.include_in_json = true ) p )) INTO v_biz_checks;
            WHEN 'private.template_soa_mappings' THEN
                WITH visit_complexity AS ( SELECT v.visit_name, COUNT(soa.activity_bk) AS procedure_count FROM private.template_soa_mappings soa JOIN private.template_study_visits v ON soa.visit_bk = v.visit_bk WHERE soa.include_in_json = true AND v.include_in_json = true GROUP BY v.visit_name ),
                     cost_per_visit AS ( SELECT sc.scenario_name, v.visit_name, SUM(c.unit_cost) AS total_visit_cost FROM private.template_soa_mappings soa JOIN private.template_study_visits v ON soa.visit_bk = v.visit_bk JOIN private.template_scenario_configurations cfg ON soa.parent_scenario_configuration_bk = cfg.scenario_configuration_bk JOIN private.template_scenarios sc ON cfg.parent_scenario_bk = sc.scenario_bk JOIN private.template_activity_costs c ON soa.parent_scenario_configuration_bk = c.parent_scenario_configuration_bk AND soa.activity_bk = c.activity_bk WHERE soa.include_in_json = true AND v.include_in_json = true AND cfg.include_in_json = true AND sc.include_in_json = true AND c.include_in_json = true GROUP BY sc.scenario_name, v.visit_name ),
                     one_time_anomalies AS ( SELECT a.arm_name, act.activity_name, COUNT(soa.map_soa_bk) AS scheduled_count FROM private.template_soa_mappings soa JOIN private.template_activities act ON soa.activity_bk = act.activity_bk JOIN private.template_study_visits v ON soa.visit_bk = v.visit_bk JOIN private.template_study_epochs e ON v.parent_epoch_bk = e.epoch_bk JOIN private.template_study_arms a ON e.parent_arm_bk = a.arm_bk WHERE act.is_one_time_procedure = TRUE AND soa.include_in_json = true GROUP BY a.arm_name, act.activity_name HAVING COUNT(soa.map_soa_bk) > 1 )
                SELECT jsonb_build_object( 'visit_complexity_top_10', (SELECT COALESCE(jsonb_agg(p ORDER BY p.procedure_count DESC), '[]'::jsonb) FROM (SELECT * FROM visit_complexity ORDER BY procedure_count DESC LIMIT 10) p), 'cost_per_visit_top_10', (SELECT COALESCE(jsonb_agg(p ORDER BY p.total_visit_cost DESC), '[]'::jsonb) FROM (SELECT * FROM cost_per_visit ORDER BY total_visit_cost DESC LIMIT 10) p), 'one_time_procedure_anomalies', (SELECT COALESCE(jsonb_agg(p ORDER BY p.arm_name, p.activity_name), '[]'::jsonb) FROM one_time_anomalies p) ) INTO v_biz_checks;
            WHEN 'private.template_fact_enrollment' THEN
                WITH total_enrollment AS ( SELECT enr->>'parent_scenario_configuration_bk' as scenario_configuration_bk, SUM((enr->>'projected_enrollment_count')::bigint) as total_projected FROM jsonb_array_elements(v_materialized_row.payload) enr GROUP BY 1 ),
                monthly_enrollment AS ( SELECT enr->>'parent_scenario_configuration_bk' as scenario_configuration_bk, to_char((current_date + ((enr->>'snapshot_date_offset_days')::int * interval '1 day')), 'YYYY-MM') AS enrollment_month, SUM((enr->>'projected_enrollment_count')::int) AS total_enrolled FROM jsonb_array_elements(v_materialized_row.payload) enr GROUP BY 1, 2 ),
                curves_by_config AS ( SELECT scenario_configuration_bk, jsonb_object_agg(enrollment_month, total_enrolled ORDER BY enrollment_month) as curve FROM monthly_enrollment GROUP BY scenario_configuration_bk )
                SELECT jsonb_build_object( 'total_enrollment_by_scenario_config', ( SELECT COALESCE(jsonb_object_agg(te.scenario_configuration_bk, te.total_projected), '{}'::jsonb) FROM total_enrollment te ), 'projected_enrollment_curve_by_scenario_config', ( SELECT COALESCE(jsonb_object_agg(cbc.scenario_configuration_bk, cbc.curve), '{}'::jsonb) FROM curves_by_config cbc ) ) INTO v_biz_checks;
            WHEN 'private.template_study_arms' THEN
                WITH arm_reconciliation AS ( SELECT sc.scenario_name, cfg.target_enrollment AS scenario_target, SUM(a.target_enrollment) AS sum_of_arm_targets, (cfg.target_enrollment - SUM(a.target_enrollment)) AS discrepancy FROM private.template_study_arms a JOIN private.template_scenario_configurations cfg ON a.parent_scenario_configuration_bk = cfg.scenario_configuration_bk JOIN private.template_scenarios sc ON cfg.parent_scenario_bk = sc.scenario_bk WHERE a.include_in_json = true AND cfg.include_in_json = true AND sc.include_in_json = true GROUP BY sc.scenario_name, cfg.target_enrollment )
                SELECT jsonb_build_object('arm_enrollment_reconciliation', COALESCE(jsonb_agg(p ORDER BY p.scenario_name), '[]'::jsonb)) FROM arm_reconciliation p INTO v_biz_checks;
            WHEN 'private.template_studies' THEN
                WITH count_by_phase AS (SELECT s.phase, count(*) AS count FROM private.template_studies s WHERE s.phase IS NOT NULL AND s.include_in_json = true GROUP BY 1),
                     count_by_therapeutic_area AS (SELECT s.therapeutic_area, count(*) AS count FROM private.template_studies s WHERE s.therapeutic_area IS NOT NULL AND s.include_in_json = true GROUP BY 1)
                SELECT jsonb_build_object('portfolio_summary', jsonb_build_object('count_by_phase', (SELECT COALESCE(jsonb_object_agg(phase, count), '{}'::jsonb) FROM count_by_phase), 'count_by_therapeutic_area', (SELECT COALESCE(jsonb_object_agg(therapeutic_area, count), '{}'::jsonb) FROM count_by_therapeutic_area))) INTO v_biz_checks;
            WHEN 'private.template_activities' THEN
                WITH activity_usage AS (SELECT soa.activity_bk, count(*) as scheduled_count FROM private.template_soa_mappings soa GROUP BY 1)
                SELECT jsonb_build_object('activity_distribution_by_type_and_domain', (SELECT COALESCE(jsonb_agg(p ORDER BY scheduled_activity_count DESC), '[]'::jsonb) FROM (SELECT a.activity_domain, a.activity_type, count(au.activity_bk) as scheduled_activity_count FROM private.template_activities a JOIN activity_usage au ON a.activity_bk = au.activity_bk GROUP BY 1, 2) p)) INTO v_biz_checks;
            WHEN 'private.template_study_epochs' THEN
                SELECT jsonb_build_object('epoch_duration_analysis', COALESCE(jsonb_agg(p ORDER BY duration_days DESC), '[]'::jsonb)) FROM (SELECT e.epoch_name, count(v.visit_bk) AS visit_count, MAX(v.offset_days) - MIN(v.offset_days) AS duration_days FROM private.template_study_epochs e JOIN private.template_study_visits v ON e.epoch_bk = v.parent_epoch_bk WHERE e.include_in_json = true AND v.include_in_json = true GROUP BY 1) p INTO v_biz_checks;
            WHEN 'private.template_study_visits' THEN
                WITH visit_intervals AS (SELECT e.epoch_name, v.visit_name, v.offset_days, v.offset_days - LAG(v.offset_days, 1, 0) OVER (PARTITION BY e.epoch_bk ORDER BY v.offset_days) AS interval_days FROM private.template_study_visits v JOIN private.template_study_epochs e ON v.parent_epoch_bk = e.epoch_bk WHERE v.include_in_json = true AND e.include_in_json = true)
                SELECT jsonb_build_object('visit_interval_analysis', COALESCE(jsonb_agg(p ORDER BY avg_interval DESC), '[]'::jsonb)) FROM (SELECT vi.epoch_name, MIN(vi.interval_days) AS min_interval, MAX(vi.interval_days) AS max_interval, AVG(vi.interval_days)::numeric(10,2) AS avg_interval FROM visit_intervals vi WHERE vi.interval_days > 0 GROUP BY 1) p INTO v_biz_checks;
            WHEN 'private.template_forecast_calculation_configs' THEN
                SELECT jsonb_build_object('calculation_method_usage_summary', (SELECT COALESCE(jsonb_agg(p ORDER BY number_of_rules DESC), '[]'::jsonb) FROM (SELECT cfg.calculation_method, COUNT(*) AS number_of_rules FROM private.template_forecast_calculation_configs cfg WHERE cfg.include_in_json = true GROUP BY 1) p)) INTO v_biz_checks;
            WHEN 'private.template_study_partners' THEN
                SELECT jsonb_build_object('most_frequent_partners', (SELECT COALESCE(jsonb_agg(p ORDER BY partnership_count DESC), '[]'::jsonb) FROM (SELECT o.organization_name, count(*) AS partnership_count FROM private.template_study_partners sp JOIN private.template_organizations o ON o.organization_bk = sp.partner_organization_bk WHERE sp.include_in_json = true GROUP BY 1 ORDER BY 2 DESC LIMIT 10) p)) INTO v_biz_checks;
            WHEN 'private.template_activity_costs' THEN
                WITH cost_variance AS ( SELECT act.activity_name, count(*) as num_prices, min(c.unit_cost) as min_cost, max(c.unit_cost) as max_cost, avg(c.unit_cost) as avg_cost FROM private.template_activity_costs c JOIN private.template_activities act ON c.activity_bk = act.activity_bk WHERE c.include_in_json = true GROUP BY 1 HAVING count(*) > 1 ORDER BY max_cost DESC )
                SELECT jsonb_build_object('cost_variance_analysis', (SELECT COALESCE(jsonb_agg(p), '[]'::jsonb) FROM cost_variance p)) INTO v_biz_checks;
            WHEN 'private.template_memberships' THEN
                WITH organization_engagement AS ( SELECT o.organization_name, COUNT(m.user_bk) AS member_count FROM private.template_memberships m JOIN private.template_organizations o ON m.organization_bk = o.organization_bk WHERE m.include_in_json = true GROUP BY 1 ORDER BY member_count DESC LIMIT 10 ),
                     user_connectivity AS ( SELECT u.user_name, COUNT(m.organization_bk) AS organization_count FROM private.template_memberships m JOIN private.template_users u ON m.user_bk = u.user_bk WHERE m.include_in_json = true GROUP BY 1 ORDER BY organization_count DESC LIMIT 10 )
                SELECT jsonb_build_object('top_10_most_engaged_organizations', (SELECT jsonb_agg(oe.*) FROM organization_engagement oe), 'top_10_most_connected_users', (SELECT jsonb_agg(uc.*) FROM user_connectivity uc)) INTO v_biz_checks;
            WHEN 'private.template_scenario_configurations' THEN
                WITH scenario_metrics AS ( SELECT st.study_short_name, sc.scenario_name, cfg.target_enrollment, cfg.target_sites, GREATEST(1, (cfg.end_date_offset_days - cfg.start_date_offset_days)) AS duration_days, ROUND( cfg.target_enrollment::numeric / GREATEST(1, cfg.target_sites::numeric), 2 ) AS subjects_per_site, ROUND( (cfg.target_enrollment::numeric / GREATEST(1, cfg.target_sites::numeric)) / (GREATEST(1, (cfg.end_date_offset_days - cfg.start_date_offset_days))::numeric / 30.44), 2 ) AS subjects_per_site_per_month FROM private.template_scenario_configurations cfg JOIN private.template_studies st ON cfg.parent_study_bk = st.study_bk JOIN private.template_scenarios sc ON cfg.parent_scenario_bk = sc.scenario_bk WHERE cfg.include_in_json = true )
                SELECT jsonb_build_object('scenario_operational_metrics', COALESCE(jsonb_agg(p ORDER BY p.subjects_per_site_per_month DESC), '[]'::jsonb)) FROM scenario_metrics p INTO v_biz_checks;
            WHEN 'private.template_sites' THEN
                WITH site_geography AS (
                        SELECT o.country_code, o.region
                        FROM private.template_sites s
                        JOIN private.template_study_partners p ON s.site_partner_bk = p.study_partner_bk
                        JOIN private.template_organizations o ON p.partner_organization_bk = o.organization_bk
                        WHERE s.include_in_json = true AND o.include_in_json = true
                     ),
                     count_by_country AS ( SELECT country_code, count(*) AS count FROM site_geography WHERE country_code IS NOT NULL GROUP BY 1 ),
                     count_by_region AS ( SELECT region, count(*) AS count FROM site_geography WHERE region IS NOT NULL GROUP BY 1 ),
                     count_by_study AS ( SELECT cfg.parent_study_bk, count(*) AS count FROM private.template_sites s JOIN private.template_scenario_configurations cfg ON s.parent_scenario_configuration_bk = cfg.scenario_configuration_bk WHERE s.include_in_json = true AND cfg.include_in_json = true GROUP BY 1 )
                SELECT jsonb_build_object( 'site_distribution', jsonb_build_object( 'count_by_country', (SELECT COALESCE(jsonb_object_agg(country_code, count), '{}'::jsonb) FROM count_by_country), 'count_by_region', (SELECT COALESCE(jsonb_object_agg(region, count), '{}'::jsonb) FROM count_by_region), 'count_by_study', (SELECT COALESCE(jsonb_object_agg(parent_study_bk, count), '{}'::jsonb) FROM count_by_study) ) ) INTO v_biz_checks;

            ELSE
                -- Do nothing for tables without specific BI checks
                NULL;
        END CASE;

        UPDATE private.materialized_template_payload
        SET business_logic_report = COALESCE(v_biz_checks, '{}'::jsonb)
        WHERE id = v_materialized_row.id;
    END LOOP;
    RAISE NOTICE '[BUSINESS LOGIC ENGINE v21.0] Successfully completed.';
END;
$$;


ALTER FUNCTION "private"."generate_business_checks"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."generate_business_checks"() IS 'The Business Intelligence engine that generates analytical summaries and plausibility checks on the materialized template data. Why: Provides the "Profiler Output" for the IDE, giving human experts the context to judge if a plan is operationally and financially sound.';



CREATE OR REPLACE FUNCTION "private"."generate_gaps_report"() RETURNS "void"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'private', 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       private.generate_gaps_report
*   Version:        1.0
*   Author:         Senior AI Business Intelligence Architect
*   Description:    This function populates the `gaps_report` column, identifying
*                   hierarchical incompleteness in the study blueprint. It acts
*                   as a "compiler warning" system for trial planners.
********************************************************************************/
DECLARE
    v_materialized_row RECORD;
    v_blueprint_payload JSONB;
    v_gaps JSONB;
BEGIN
    RAISE NOTICE '  -> [VALIDATION ENGINE - GAPS] Starting...';

    SELECT COALESCE(jsonb_object_agg(REPLACE(REPLACE(template_table_name, 'private.template_', ''), '_', ''), payload), '{}'::jsonb)
    INTO v_blueprint_payload
    FROM private.materialized_template_payload;

    FOR v_materialized_row IN SELECT * FROM private.materialized_template_payload LOOP
        v_gaps := '[]'::jsonb;

        CASE v_materialized_row.template_table_name
            WHEN 'private.template_studies' THEN
                SELECT jsonb_agg(jsonb_build_object('check_name', 'Study with no Scenario Configurations', 'study_bk', s->>'study_bk')) INTO v_gaps FROM jsonb_array_elements(v_materialized_row.payload) s WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') cfg WHERE cfg->>'parent_study_bk' = s->>'study_bk');
            WHEN 'private.template_scenario_configurations' THEN
                SELECT jsonb_agg(jsonb_build_object('check_name', 'Configuration with no Arms', 'config_bk', cfg->>'scenario_configuration_bk')) INTO v_gaps FROM jsonb_array_elements(v_materialized_row.payload) cfg WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studyarms') arm WHERE arm->>'parent_scenario_configuration_bk' = cfg->>'scenario_configuration_bk');
            WHEN 'private.template_study_arms' THEN
                SELECT jsonb_agg(jsonb_build_object('check_name', 'Arm with no Epochs', 'arm_bk', arm->>'arm_bk')) INTO v_gaps FROM jsonb_array_elements(v_materialized_row.payload) arm WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studyepochs') e WHERE e->>'parent_arm_bk' = arm->>'arm_bk');
            WHEN 'private.template_study_epochs' THEN
                SELECT jsonb_agg(jsonb_build_object('check_name', 'Epoch with no Visits', 'epoch_bk', e->>'epoch_bk')) INTO v_gaps FROM jsonb_array_elements(v_materialized_row.payload) e WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studyvisits') v WHERE v->>'parent_epoch_bk' = e->>'epoch_bk');
            WHEN 'private.template_study_visits' THEN
                SELECT jsonb_agg(jsonb_build_object('check_name', 'Visit with no Activities (SoA)', 'visit_bk', v->>'visit_bk')) INTO v_gaps FROM jsonb_array_elements(v_materialized_row.payload) v WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'soamappings') soa WHERE soa->>'visit_bk' = v->>'visit_bk');
            WHEN 'private.template_study_partners' THEN
                SELECT jsonb_agg(jsonb_build_object('check_name', 'Site Partner with no Staff', 'study_partner_bk', p->>'study_partner_bk')) INTO v_gaps FROM jsonb_array_elements(v_materialized_row.payload) p WHERE p->>'partner_role' = 'Site' AND NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studystaff') st WHERE st->>'parent_study_partner_bk' = p->>'study_partner_bk');
            ELSE
                -- No gap checks for this table
        END CASE;

        UPDATE private.materialized_template_payload SET gaps_report = COALESCE(v_gaps, '[]'::jsonb) WHERE id = v_materialized_row.id;
    END LOOP;
END;
$$;


ALTER FUNCTION "private"."generate_gaps_report"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."generate_gaps_report"() IS 'The validation engine that identifies missing hierarchical links in the template data (e.g., an arm with no epochs). Why: Acts as the "Compiler Warnings" system for the IDE, highlighting incompleteness that could lead to silent omissions or incorrect results.';



CREATE OR REPLACE FUNCTION "private"."get_sk_from_bk_for_seeding"("p_bk" "text", "p_table_name" "text") RETURNS bigint
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public'
    AS $$
DECLARE
    v_sk BIGINT;
    v_sql TEXT;
    v_pk_column_name TEXT;
    v_bk_column_name TEXT;
BEGIN
    -- Whitelist of tables this function is allowed to query.
    IF p_table_name NOT IN (
        'dim_organization', 'dim_user', 'dim_study', 'dim_budget_scenario',
        'dim_study_arm', 'dim_study_epochs', 'dim_study_visits', 'dim_activity',
        'dim_budget_category', 'dim_reimbursement_type', 'dim_amendment',
        'dim_site', 'dim_forecast_calculation_config'
    ) THEN
        RAISE EXCEPTION 'Invalid table name provided to seeder helper: %', p_table_name;
    END IF;

    -- Dynamically build column names based on our standard convention.
    v_pk_column_name := REPLACE(p_table_name, 'dim_', '') || '_sk';
    v_bk_column_name := REPLACE(p_table_name, 'dim_', '') || '_bk';

    -- Build and execute the query to get the surrogate key from the business key.
    v_sql := format('SELECT %I FROM %I WHERE %I = %L',
                    v_pk_column_name,
                    p_table_name,
                    v_bk_column_name,
                    p_bk);

    EXECUTE v_sql INTO v_sk;
    RETURN v_sk;
	RAISE NOTICE 'Function private.get_sk_from_bk_for_seeding created and secured.';
END;
$$;


ALTER FUNCTION "private"."get_sk_from_bk_for_seeding"("p_bk" "text", "p_table_name" "text") OWNER TO "postgres";


COMMENT ON FUNCTION "private"."get_sk_from_bk_for_seeding"("p_bk" "text", "p_table_name" "text") IS 'A secure helper function for the seeder orchestrator. Why: Allows the seeder to resolve surrogate keys from business keys without having direct query access to the public tables, maintaining a strong security boundary.';



CREATE OR REPLACE FUNCTION "private"."handle_stripe_checkout_completed"("p_1_organization_bk" "text", "p_2_stripe_customer_id" "text", "p_3_stripe_subscription_id" "text", "p_4_current_period_end" timestamp with time zone) RETURNS "void"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'public', 'pg_temp'
    AS $$declare
  v_org_sk bigint;
begin
  -- Map BK -> SK via your mapper. Rename if your function differs.
  v_org_sk := public.get_organization_sk_from_bk(p_1_organization_bk);

  if v_org_sk is null then
    raise exception 'Unknown organization_bk: %', p_1_organization_bk;
  end if;

  insert into public.subscriptions (
    organization_sk,
    organization_bk,
    stripe_customer_id,
    stripe_subscription_id,
    tier,
    status,
    current_period_end,
    updated_at
  )
  values (
    v_org_sk,
    p_1_organization_bk,
    p_2_stripe_customer_id,
    p_3_stripe_subscription_id,
    'pro',
    'active',
    p_4_current_period_end,
    now()
  )
  on conflict (organization_bk) do update
  set stripe_customer_id     = excluded.stripe_customer_id,
      stripe_subscription_id = excluded.stripe_subscription_id,
      tier                   = excluded.tier,
      status                 = excluded.status,
      current_period_end     = excluded.current_period_end,
      updated_at             = now();
end;$$;


ALTER FUNCTION "private"."handle_stripe_checkout_completed"("p_1_organization_bk" "text", "p_2_stripe_customer_id" "text", "p_3_stripe_subscription_id" "text", "p_4_current_period_end" timestamp with time zone) OWNER TO "postgres";


COMMENT ON FUNCTION "private"."handle_stripe_checkout_completed"("p_1_organization_bk" "text", "p_2_stripe_customer_id" "text", "p_3_stripe_subscription_id" "text", "p_4_current_period_end" timestamp with time zone) IS 'Internal Stripe webhook handler for successful checkouts. Why: Provisions or updates an organization''s subscription status upon successful payment. How: Maps the organization BK to its SK and upserts the subscription details.';



CREATE OR REPLACE FUNCTION "private"."handle_stripe_event"("p_event" "jsonb") RETURNS "void"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
DECLARE
    v_org_sk BIGINT;
    v_customer_id TEXT;
    v_subscription_id TEXT;
    v_session_status TEXT;
BEGIN
    -- Logic to find organization_sk from the event payload
    v_customer_id := p_event->'data'->'object'->>'customer';
    SELECT organization_sk INTO v_org_sk FROM public.subscriptions WHERE stripe_customer_id = v_customer_id;

    CASE p_event->>'type'
        WHEN 'checkout.session.completed' THEN
            v_subscription_id := p_event->'data'->'object'->>'subscription';
            UPDATE public.subscriptions
            SET tier = 'pro', status = 'active', stripe_subscription_id = v_subscription_id, updated_at = NOW()
            WHERE stripe_customer_id = v_customer_id;
        WHEN 'customer.subscription.deleted' THEN
            UPDATE public.subscriptions
            SET tier = 'free', status = 'canceled', updated_at = NOW()
            WHERE stripe_subscription_id = (p_event->'data'->'object'->>'id');
        -- Add other cases like 'invoice.payment_failed'
    END CASE;
END;
$$;


ALTER FUNCTION "private"."handle_stripe_event"("p_event" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "private"."handle_stripe_event"("p_event" "jsonb") IS 'A generic, internal Stripe webhook handler. Why: Centralizes the logic for handling various subscription state transitions (e.g., completion, cancellation). How: Parses the event type from the Stripe payload and updates subscription records accordingly.';



CREATE OR REPLACE FUNCTION "private"."is_config_a_group_definition"("p_config_sk" bigint) RETURNS boolean
    LANGUAGE "sql" STABLE SECURITY DEFINER
    AS $$
    SELECT is_group_definition FROM public.dim_forecast_calculation_config
    WHERE forecast_config_sk = p_config_sk;
$$;


ALTER FUNCTION "private"."is_config_a_group_definition"("p_config_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "private"."is_config_a_group_definition"("p_config_sk" bigint) IS 'A security helper that checks if a forecast configuration is a group definition. Why: Used in business logic to guard against incorrect composition of enrollment curve groups.';



CREATE OR REPLACE FUNCTION "private"."is_user_admin_in_org"("p_user_sk" bigint, "p_org_sk" bigint) RETURNS boolean
    LANGUAGE "sql" STABLE SECURITY DEFINER
    SET "search_path" TO 'public'
    AS $$
/********************************************************************************
*   Function:       private.is_user_admin_in_org
*   Version:        1.0 (Gold Standard - Security Helper)
*   Author:         Principal Database Architect
*   Description:    A secure, context-free helper to check if a specific user
*                   has the 'Admin' role in a specific organization.
*                   It is SECURITY DEFINER to bypass RLS for this internal check.
********************************************************************************/
  SELECT EXISTS (
    SELECT 1
    FROM public.user_organization_membership uom
    WHERE uom.user_sk = p_user_sk
      AND uom.organization_sk = p_org_sk
      AND uom.role = 'Admin'
  );
$$;


ALTER FUNCTION "private"."is_user_admin_in_org"("p_user_sk" bigint, "p_org_sk" bigint) OWNER TO "postgres";


COMMENT ON FUNCTION "private"."is_user_admin_in_org"("p_user_sk" bigint, "p_org_sk" bigint) IS 'A secure, context-free helper to check if a specific user has the ''Admin'' role in a specific organization. Why: Used by SECURITY DEFINER routines to authorize sensitive administrative operations without relying on the caller''s session context.';



CREATE OR REPLACE FUNCTION "private"."summarize_bi_report"("p_template_table_name" "text", "p_bi_report" "jsonb", "p_metadata" "jsonb") RETURNS "text"
    LANGUAGE "plpgsql" STABLE
    AS $_$
/********************************************************************************
*   Function:       private.summarize_bi_report
*   Version:        11.4 (Definitive - Hardened & Aligned)
*   Author:         Senior AI Database Architect
*   Description:    This definitive version hardens the function against errors
*                   by adding defensive checks for the existence and type of JSONB
*                   keys before processing them as arrays. It specifically fixes the
*                   "cannot extract from scalar" error for the `study_staff` BI
*                   and ensures all summaries are generated safely and correctly.
********************************************************************************/
DECLARE
    v_summary_text TEXT;
    v_json_array_len INT;
    v_min_intensity NUMERIC;
    v_max_intensity NUMERIC;
    v_studies_missing_contact INT;
    v_scenarios_with_discrepancy INT;
    v_total_forecast NUMERIC;
    v_total_assignments INT;
    v_unused_count INT;
BEGIN
    IF p_bi_report IS NULL OR p_bi_report = '{}'::jsonb OR p_bi_report::text = '[]'::text THEN
        RETURN NULL;
    END IF;

    CASE p_template_table_name
        WHEN 'private.template_study_staff' THEN
            -- THE FIX: Add a defensive guard clause.
            IF p_bi_report ? 'primary_contact_coverage_check' AND jsonb_typeof(p_bi_report->'primary_contact_coverage_check') = 'array' THEN
                v_studies_missing_contact := (SELECT COUNT(*) FROM jsonb_array_elements(p_bi_report->'primary_contact_coverage_check') elem WHERE (elem->>'primary_contact_count')::int = 0);
                v_summary_text := 'Staffing Profile: ' || CASE WHEN v_studies_missing_contact > 0 THEN 'WARNING: ' || v_studies_missing_contact || ' study/studies are missing a primary contact.' ELSE 'All studies have a primary contact assigned.' END;
            END IF;

        WHEN 'private.template_budget_categories' THEN
            IF p_bi_report ? 'portfolio_spend_by_category' AND jsonb_typeof(p_bi_report->'portfolio_spend_by_category') = 'array' THEN
                v_json_array_len := jsonb_array_length(p_bi_report->'portfolio_spend_by_category');
                v_unused_count := COALESCE(jsonb_array_length(p_bi_report->'unused_categories'), 0);
                IF v_json_array_len > 0 THEN
                    v_summary_text := 'Top Cost Driver: The largest category is "' || (p_bi_report->'portfolio_spend_by_category'->0->>'category_name') || '" at $' || TO_CHAR((p_bi_report->'portfolio_spend_by_category'->0->>'total_net_cost')::numeric, 'FM999G999G999G990') || '. Portfolio-wide, ' || v_unused_count || ' categories are defined but currently unused.';
                END IF;
            END IF;

        WHEN 'private.template_fact_forecast_detail' THEN
            IF p_bi_report ? 'spend_by_payee_portfolio_wide' AND jsonb_typeof(p_bi_report->'spend_by_payee_portfolio_wide') = 'array' THEN
                v_json_array_len := jsonb_array_length(p_bi_report->'spend_by_payee_portfolio_wide');
                IF v_json_array_len > 0 THEN
                    WITH payee_costs AS (
                        SELECT (value->>'total_net_cost')::numeric as cost
                        FROM jsonb_array_elements(p_bi_report->'spend_by_payee_portfolio_wide') value
                    )
                    SELECT SUM(cost) INTO v_total_forecast FROM payee_costs;

                    v_summary_text := 'Portfolio Financials: Total net forecast is $' || TO_CHAR(v_total_forecast, 'FM999G999G999G990') || ' across all scenarios. The top payee is "' || (p_bi_report->'spend_by_payee_portfolio_wide'->0->>'payee_name') || '", accounting for $' || TO_CHAR((p_bi_report->'spend_by_payee_portfolio_wide'->0->>'total_net_cost')::numeric, 'FM999G999G999G990') || ' of the total spend.';
                END IF;
            END IF;

        -- Other CASE statements remain unchanged...
        WHEN 'private.template_organizations' THEN
            v_summary_text := NULL;

        WHEN 'private.template_users' THEN
            v_summary_text := NULL;

        WHEN 'private.template_memberships' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'top_10_most_engaged_organizations'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Collaboration Hub: ' || (p_metadata->>'total_record_count') || ' memberships connect ' || (SELECT count(*) FROM private.template_users WHERE include_in_json = true) || ' users across ' || (SELECT count(*) FROM private.template_organizations WHERE include_in_json = true) || ' organizations.';
            END IF;

        WHEN 'private.template_reimbursement_types' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'reimbursement_rule_summary'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Reimbursement Rules: The most used rule, "' || (p_bi_report->'reimbursement_rule_summary'->0->>'rule_name') || '", was applied ' || (p_bi_report->'reimbursement_rule_summary'->0->>'usage_count') || ' times, governing a gross value of $' || TO_CHAR((p_bi_report->'reimbursement_rule_summary'->0->>'total_forecasted_gross_value')::numeric, 'FM999G999G990') || ' and resulting in a net financial impact of $' || TO_CHAR((p_bi_report->'reimbursement_rule_summary'->0->>'total_forecasted_net_value')::numeric, 'FM999G999G990') || '.';
            END IF;

        WHEN 'private.template_activities' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'activity_distribution_by_type_and_domain'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Operational Profile: The most common activity type is "' || (p_bi_report->'activity_distribution_by_type_and_domain'->0->>'activity_type') || '" in the "' || (p_bi_report->'activity_distribution_by_type_and_domain'->0->>'activity_domain') || '" domain.';
            END IF;

        WHEN 'private.template_studies' THEN
            v_summary_text := 'Portfolio Summary: Covers ' || (SELECT count(*) FROM jsonb_object_keys(p_bi_report->'portfolio_summary'->'count_by_phase')) || ' phase(s) and ' || (SELECT count(*) FROM jsonb_object_keys(p_bi_report->'portfolio_summary'->'count_by_therapeutic_area')) || ' therapeutic area(s).';

        WHEN 'private.template_scenarios' THEN
            IF jsonb_typeof(p_bi_report->'scenario_total_costs') = 'object' THEN
                v_summary_text := 'Financial Summary: Net budgets range up to $' || TO_CHAR((SELECT MAX((value->>'net_cost')::numeric) FROM jsonb_each(p_bi_report->'scenario_total_costs')), 'FM999G999G999G990');
            END IF;

        WHEN 'private.template_sites' THEN
            v_total_assignments := (p_metadata->>'total_record_count');
            v_summary_text := 'Geographic Footprint: ' || v_total_assignments || ' site-to-partner assignments across ' || (SELECT count(*) FROM jsonb_object_keys(p_bi_report->'site_distribution'->'count_by_study')) || ' studies and ' || (SELECT count(*) FROM jsonb_object_keys(p_bi_report->'site_distribution'->'count_by_country')) || ' countries.';

        WHEN 'private.template_amendments' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'amendment_financial_impact_analysis'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Financial Impact: Amendment ' || (p_bi_report->'amendment_financial_impact_analysis'->0->>'amendment_version_id') || ' triggered a new plan with a net cost of $' || TO_CHAR((p_bi_report->'amendment_financial_impact_analysis'->0->'triggered_scenario_total_cost'->>'net_cost')::numeric, 'FM999G999G999G990');
            END IF;

        WHEN 'private.template_study_arms' THEN
            v_scenarios_with_discrepancy := (SELECT COUNT(*) FROM jsonb_array_elements(p_bi_report->'arm_enrollment_reconciliation') elem WHERE (elem->>'discrepancy')::numeric != 0);
            v_summary_text := 'Enrollment Integrity: ' || CASE WHEN v_scenarios_with_discrepancy > 0 THEN 'WARNING: ' || v_scenarios_with_discrepancy || ' scenarios have enrollment discrepancies.' ELSE 'All scenarios have balanced arm vs. total enrollment targets.' END;

        WHEN 'private.template_study_epochs' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'epoch_duration_analysis'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Study Design: Epochs span up to ' || (SELECT MAX((value->>'duration_days')::int) FROM jsonb_array_elements(p_bi_report->'epoch_duration_analysis')) || ' days.';
            END IF;

        WHEN 'private.template_study_visits' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'visit_interval_analysis'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Visit Cadence: The longest average interval between visits is ' || (p_bi_report->'visit_interval_analysis'->0->>'avg_interval') || ' days.';
            END IF;

        WHEN 'private.template_scenario_configurations' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'scenario_operational_metrics'), 0);
            IF v_json_array_len > 0 THEN
                SELECT MIN((elem->>'subjects_per_site_per_month')::numeric), MAX((elem->>'subjects_per_site_per_month')::numeric) INTO v_min_intensity, v_max_intensity FROM jsonb_array_elements(p_bi_report->'scenario_operational_metrics') elem;
                v_summary_text := 'Operational Intensity: Plans range from ' || v_min_intensity || ' to ' || v_max_intensity || ' subjects/site/month.';
            END IF;

        WHEN 'private.template_study_partners' THEN
            v_total_assignments := (p_metadata->>'total_record_count');
            v_summary_text := 'Collaboration Model: '|| v_total_assignments ||' partnership roles defined across ' || jsonb_array_length(p_bi_report->'most_frequent_partners') || ' key external partners.';

        WHEN 'private.template_forecast_calculation_configs' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'calculation_method_usage_summary'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Financial Model: Dominated by the "' || (p_bi_report->'calculation_method_usage_summary'->0->>'calculation_method') || '" method, used in ' || (p_bi_report->'calculation_method_usage_summary'->0->>'number_of_rules') || ' rules.';
            END IF;

        WHEN 'private.template_soa_mappings' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'visit_complexity_top_10'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Visit Analysis: The most complex visit has ' || (p_bi_report->'visit_complexity_top_10'->0->>'procedure_count') || ' procedures. The most expensive costs $' || TO_CHAR((p_bi_report->'cost_per_visit_top_10'->0->>'total_visit_cost')::numeric, 'FM999G999G990');
            END IF;

        WHEN 'private.template_activity_costs' THEN
            v_json_array_len := COALESCE(jsonb_array_length(p_bi_report->'cost_variance_analysis'), 0);
            IF v_json_array_len > 0 THEN
                v_summary_text := 'Financial Hygiene: ' || jsonb_array_length(p_bi_report->'cost_variance_analysis') || ' activities have multiple price points across different scenarios.';
            END IF;

        WHEN 'private.template_fact_enrollment' THEN
            DECLARE
                v_top_scenario_bk TEXT;
                v_top_scenario_total BIGINT;
                v_top_scenario_curve JSONB;
                v_peak_month TEXT;
                v_peak_enrollment INT;
            BEGIN
                IF p_bi_report ? 'total_enrollment_by_scenario_config' AND jsonb_typeof(p_bi_report->'total_enrollment_by_scenario_config') = 'object' THEN
                    SELECT key, value::bigint INTO v_top_scenario_bk, v_top_scenario_total
                    FROM jsonb_each_text(p_bi_report->'total_enrollment_by_scenario_config')
                    ORDER BY value::bigint DESC LIMIT 1;

                    IF v_top_scenario_bk IS NOT NULL THEN
                        v_top_scenario_curve := p_bi_report->'projected_enrollment_curve_by_scenario_config'->v_top_scenario_bk;
                        IF v_top_scenario_curve IS NOT NULL AND jsonb_typeof(v_top_scenario_curve) = 'object' AND (SELECT count(*) FROM jsonb_object_keys(v_top_scenario_curve)) > 0 THEN
                            SELECT key, value::int INTO v_peak_month, v_peak_enrollment
                            FROM jsonb_each_text(v_top_scenario_curve)
                            ORDER BY value::int DESC LIMIT 1;
                            v_summary_text := 'Top Plan (' || split_part(v_top_scenario_bk, '-', 5) || '): ' || v_top_scenario_total || ' subjects total, peaking in ' || v_peak_month || ' with ' || v_peak_enrollment || ' subjects.';
                        END IF;
                    END IF;
                END IF;
            END;

        ELSE
            v_summary_text := NULL; -- Default for any unhandled table
    END CASE;

    RETURN v_summary_text;
END;
$_$;


ALTER FUNCTION "private"."summarize_bi_report"("p_template_table_name" "text", "p_bi_report" "jsonb", "p_metadata" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "private"."summarize_bi_report"("p_template_table_name" "text", "p_bi_report" "jsonb", "p_metadata" "jsonb") IS 'A utility function that generates a concise, human-readable text summary from a complex BI report JSONB object. Why: Turns raw analytical data into actionable insights for display in the UI.';



CREATE OR REPLACE FUNCTION "private"."summarize_validation_status"("p_failures" "jsonb", "p_gaps" "jsonb", "p_bi_report" "jsonb", "p_template_table_name" "text", "p_metadata" "jsonb") RETURNS "text"
    LANGUAGE "plpgsql" STABLE
    SET "search_path" TO 'private'
    AS $$
/********************************************************************************
*   Function:       private.summarize_validation_status
*   Version:        1.0
*   Author:         Senior AI Business Intelligence Architect
*   Description:    Generates a single, prioritized insight string based on the
*                   three-part validation system. It reports critical failures
*                   first, then warnings (gaps), and finally business insights.
********************************************************************************/
DECLARE
    v_failure_count INT;
    v_gap_count INT;
    v_first_failure_name TEXT;
    v_first_gap_name TEXT;
BEGIN
    v_failure_count := COALESCE(jsonb_array_length(p_failures), 0);
    v_gap_count := COALESCE(jsonb_array_length(p_gaps), 0);

    IF v_failure_count > 0 THEN
        v_first_failure_name := p_failures->0->>'check_name';
        RETURN 'CRITICAL: ' || v_failure_count || ' failure(s) found, starting with "' || v_first_failure_name || '". Plan is structurally unsound.';
    ELSIF v_gap_count > 0 THEN
        v_first_gap_name := p_gaps->0->>'check_name';
        RETURN 'WARNING: ' || v_gap_count || ' gap(s) found, starting with "' || v_first_gap_name || '". Plan is incomplete.';
    ELSE
        -- If no failures or gaps, call the existing BI summary function.
        RETURN private.summarize_bi_report(p_template_table_name, p_bi_report, p_metadata);
    END IF;
END;
$$;


ALTER FUNCTION "private"."summarize_validation_status"("p_failures" "jsonb", "p_gaps" "jsonb", "p_bi_report" "jsonb", "p_template_table_name" "text", "p_metadata" "jsonb") OWNER TO "postgres";


COMMENT ON FUNCTION "private"."summarize_validation_status"("p_failures" "jsonb", "p_gaps" "jsonb", "p_bi_report" "jsonb", "p_template_table_name" "text", "p_metadata" "jsonb") IS 'Produces a single, prioritized validation message for the template authoring UI. Why: Communicates the most critical issue first: failures, then gaps, and finally BI insights if the data is structurally sound.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_activity"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
/********************************************************************************
*   Trigger Function: trg_fn_cascade_from_activity
*   Version:          2.0 (Direct Foreign Key Refactor)
*   Author:           Senior AI Database Architect
*   Description:      This version is updated for the "Direct Foreign Key"
*                     architecture. The line that cascaded updates to the now-
*                     deleted `template_activity_budget_mappings` table has been
*                     removed.
********************************************************************************/
BEGIN
    -- The reference to the deleted mapping table is removed.
    -- UPDATE private.template_activity_budget_mappings SET include_in_json = NEW.include_in_json WHERE activity_bk = NEW.activity_bk;

    UPDATE private.template_activity_costs SET include_in_json = NEW.include_in_json WHERE activity_bk = NEW.activity_bk;
    UPDATE private.template_soa_mappings SET include_in_json = NEW.include_in_json WHERE activity_bk = NEW.activity_bk;
    UPDATE private.template_forecast_calculation_configs SET include_in_json = NEW.include_in_json WHERE activity_bk = NEW.activity_bk;
    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_from_activity"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_activity"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When an activity''s `include_in_json` flag is changed, this cascades the change to all its child records (costs, SoA mappings, etc.).';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_arm"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN UPDATE private.template_study_epochs SET include_in_json = NEW.include_in_json WHERE parent_arm_bk = NEW.arm_bk; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_from_arm"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_arm"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a study arm''s `include_in_json` flag is changed, this cascades the change to all its child epochs.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_budget_category"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    /********************************************************************************
    *   Trigger Function: trg_fn_cascade_from_budget_category
    *   Version:          2.0 (Direct Foreign Key Refactor)
    *   Author:           Senior AI Database Architect
    *   Description:      This trigger is reinstated for the new architecture.
    *                     It now correctly cascades the `include_in_json` state
    *                     downward to all activities that have a direct foreign
    *                     key relationship to this budget category.
    ********************************************************************************/
    BEGIN
        UPDATE private.template_activities
        SET include_in_json = NEW.include_in_json
        WHERE budget_category_bk = NEW.budget_category_bk;
        RETURN NULL;
    END;
    $$;


ALTER FUNCTION "private"."trg_fn_cascade_from_budget_category"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_budget_category"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a budget category''s `include_in_json` flag is changed, this cascades the change to all its child activities.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_epoch"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN UPDATE private.template_study_visits SET include_in_json = NEW.include_in_json WHERE parent_epoch_bk = NEW.epoch_bk; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_from_epoch"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_epoch"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When an epoch''s `include_in_json` flag is changed, this cascades the change to all its child visits.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_organization"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
DECLARE
    v_user_bks_in_org TEXT[];
BEGIN
    -- Find all users belonging to the organization being changed.
    SELECT array_agg(user_bk) INTO v_user_bks_in_org
    FROM private.template_memberships
    WHERE organization_bk = NEW.organization_bk;

    -- If users are found, cascade the organization's new state to them.
    IF v_user_bks_in_org IS NOT NULL THEN
        UPDATE private.template_users
        SET include_in_json = NEW.include_in_json
        WHERE user_bk = ANY(v_user_bks_in_org);
    END IF;
    
    -- This will implicitly trigger the user-to-membership cascade,
    -- setting the membership state correctly.

    -- Cascade to partner assignments
    UPDATE private.template_study_partners
    SET include_in_json = NEW.include_in_json
    WHERE partner_organization_bk = NEW.organization_bk;

    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_from_organization"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_organization"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When an organization''s `include_in_json` flag is changed, this cascades the change to its associated users and partner assignments.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_partner"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
/********************************************************************************
*   Trigger Function: trg_fn_cascade_from_partner
*   Version:          1.0 (New for Staff Refactor)
*   Description:      Cascades the include_in_json state from a study partner
*                     down to its assigned study staff.
********************************************************************************/
BEGIN
    UPDATE private.template_study_staff
    SET include_in_json = NEW.include_in_json
    WHERE parent_study_partner_bk = NEW.study_partner_bk;
    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_from_partner"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_partner"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a study partner''s `include_in_json` flag is changed, this cascades the change to its assigned staff.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_reimbursement_type"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
BEGIN
    UPDATE private.template_activity_costs
    SET include_in_json = NEW.include_in_json
    WHERE reimbursement_type_bk = NEW.reimbursement_type_bk;
    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_from_reimbursement_type"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_reimbursement_type"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a reimbursement type''s `include_in_json` flag is changed, this cascades the change to all activity costs that use it.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_scenario"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN UPDATE private.template_scenario_configurations SET include_in_json = NEW.include_in_json WHERE parent_scenario_bk = NEW.scenario_bk; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_from_scenario"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_scenario"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a scenario''s `include_in_json` flag is changed, this cascades the change to all its child scenario configurations.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_scenario_config"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN
        UPDATE private.template_study_arms SET include_in_json = NEW.include_in_json WHERE parent_scenario_configuration_bk = NEW.scenario_configuration_bk;
        UPDATE private.template_sites SET include_in_json = NEW.include_in_json WHERE parent_scenario_configuration_bk = NEW.scenario_configuration_bk;
        UPDATE private.template_study_partners SET include_in_json = NEW.include_in_json WHERE parent_scenario_configuration_bk = NEW.scenario_configuration_bk;
        UPDATE private.template_forecast_calculation_configs SET include_in_json = NEW.include_in_json WHERE parent_scenario_configuration_bk = NEW.scenario_configuration_bk;
        UPDATE private.template_activity_costs SET include_in_json = NEW.include_in_json WHERE parent_scenario_configuration_bk = NEW.scenario_configuration_bk;
        UPDATE private.template_soa_mappings SET include_in_json = NEW.include_in_json WHERE parent_scenario_configuration_bk = NEW.scenario_configuration_bk;
        RETURN NULL;
    END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_from_scenario_config"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_scenario_config"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a scenario configuration''s `include_in_json` flag is changed, this cascades the change to all its direct children (arms, sites, partners, etc.).';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_study"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN
        -- Step 1: Cascade to the configuration hub and other direct children. This is correct.
        UPDATE private.template_scenario_configurations
        SET include_in_json = NEW.include_in_json
        WHERE parent_study_bk = NEW.study_bk;

        UPDATE private.template_amendments
        SET include_in_json = NEW.include_in_json
        WHERE parent_study_bk = NEW.study_bk;

        -- THE FIX: Also cascade to the scenarios that are linked to this study
        -- via the configuration hub. This restores the original behavior.
        UPDATE private.template_scenarios
        SET include_in_json = NEW.include_in_json
        WHERE scenario_bk IN (
            SELECT parent_scenario_bk
            FROM private.template_scenario_configurations
            WHERE parent_study_bk = NEW.study_bk
        );

        RETURN NULL;
    END;
    $$;


ALTER FUNCTION "private"."trg_fn_cascade_from_study"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_study"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a study''s `include_in_json` flag is changed, this cascades the change to its child scenarios and scenario configurations.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_user_to_membership"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
BEGIN
    -- When a user's status changes, update their memberships.
    -- This trigger now correctly handles both enabling and disabling.
    UPDATE private.template_memberships m
    SET include_in_json = (NEW.include_in_json AND o.include_in_json)
    FROM private.template_organizations o
    WHERE m.user_bk = NEW.user_bk
      AND m.organization_bk = o.organization_bk;

    -- Cascade to site staff assignments
    UPDATE private.template_site_staff
    SET include_in_json = NEW.include_in_json
    WHERE staff_user_bk = NEW.user_bk;

    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_from_user_to_membership"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_user_to_membership"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a user''s `include_in_json` flag is changed, this cascades the change to their memberships and staff assignments.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_from_visit"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN UPDATE private.template_soa_mappings SET include_in_json = NEW.include_in_json WHERE visit_bk = NEW.visit_bk; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_from_visit"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_from_visit"() IS 'Trigger function for the template authoring system. Why: Enforces downward hierarchical integrity. How: When a visit''s `include_in_json` flag is changed, this cascades the change to its child SoA mappings.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_arm"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN IF NEW.include_in_json THEN UPDATE private.template_scenario_configurations SET include_in_json = TRUE WHERE scenario_configuration_bk = NEW.parent_scenario_configuration_bk; END IF; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_arm"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_arm"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If an arm is included, ensures its parent scenario configuration is also included to prevent orphaned records.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_budget_category"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN IF NEW.include_in_json AND NEW.parent_category_bk IS NOT NULL THEN UPDATE private.template_budget_categories SET include_in_json = TRUE WHERE budget_category_bk = NEW.parent_category_bk; END IF; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_budget_category"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_budget_category"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If a child budget category is included, ensures its parent category is also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_cost"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
/********************************************************************************
*   Trigger Function: trg_fn_cascade_up_from_cost
*   Version:          2.0 (Partner-Centric)
*   Description:      Refactored to cascade `include_in_json` changes up to
*                     the new partner records instead of direct organizations.
********************************************************************************/
BEGIN
    IF NEW.include_in_json THEN
        UPDATE private.template_scenario_configurations SET include_in_json = TRUE WHERE scenario_configuration_bk = NEW.parent_scenario_configuration_bk;
        UPDATE private.template_activities SET include_in_json = TRUE WHERE activity_bk = NEW.activity_bk;
        -- Cascade up to the partner records
        UPDATE private.template_study_partners SET include_in_json = TRUE WHERE study_partner_bk = NEW.cost_bearing_partner_bk;
        IF NEW.payer_partner_bk IS NOT NULL THEN
            UPDATE private.template_study_partners SET include_in_json = TRUE WHERE study_partner_bk = NEW.payer_partner_bk;
        END IF;
        IF NEW.reimbursement_type_bk IS NOT NULL THEN
            UPDATE private.template_reimbursement_types SET include_in_json = TRUE WHERE reimbursement_type_bk = NEW.reimbursement_type_bk;
        END IF;
    END IF;
    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_cost"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_cost"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If an activity cost is included, ensures its parent configuration, activity, and partners are also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_epoch"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN IF NEW.include_in_json THEN UPDATE private.template_study_arms SET include_in_json = TRUE WHERE arm_bk = NEW.parent_arm_bk; END IF; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_epoch"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_epoch"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If an epoch is included, ensures its parent arm is also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_partner"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN IF NEW.include_in_json THEN UPDATE private.template_scenario_configurations SET include_in_json = TRUE WHERE scenario_configuration_bk = NEW.parent_scenario_configuration_bk; UPDATE private.template_organizations SET include_in_json = TRUE WHERE organization_bk = NEW.partner_organization_bk; END IF; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_partner"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_partner"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If a partner is included, ensures its parent configuration and organization are also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_scenario_config"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN IF NEW.include_in_json THEN UPDATE private.template_studies SET include_in_json = TRUE WHERE study_bk = NEW.parent_study_bk; UPDATE private.template_scenarios SET include_in_json = TRUE WHERE scenario_bk = NEW.parent_scenario_bk; END IF; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_scenario_config"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_scenario_config"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If a scenario configuration is included, ensures its parent study and scenario are also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_site"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
/********************************************************************************
*   Trigger Function: trg_fn_cascade_up_from_site
*   Version:          2.0 (Partner-Centric)
*   Description:      Refactored to cascade `include_in_json` changes up to
*                     the new partner record instead of a direct organization.
********************************************************************************/
BEGIN
    IF NEW.include_in_json THEN
        UPDATE private.template_scenario_configurations SET include_in_json = TRUE WHERE scenario_configuration_bk = NEW.parent_scenario_configuration_bk;
        -- Cascade up to the partner record
        UPDATE private.template_study_partners SET include_in_json = TRUE WHERE study_partner_bk = NEW.site_partner_bk;
    END IF;
    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_site"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_site"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If a site is included, ensures its parent configuration and partner record are also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_soa"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN IF NEW.include_in_json THEN UPDATE private.template_scenario_configurations SET include_in_json = TRUE WHERE scenario_configuration_bk = NEW.parent_scenario_configuration_bk; UPDATE private.template_study_visits SET include_in_json = TRUE WHERE visit_bk = NEW.visit_bk; UPDATE private.template_activities SET include_in_json = TRUE WHERE activity_bk = NEW.activity_bk; END IF; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_soa"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_soa"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If a SoA mapping is included, ensures its parent configuration, visit, and activity are also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_study_staff"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
/********************************************************************************
*   Trigger Function: trg_fn_cascade_up_from_site_staff
*   Version:          2.0 (Partner-Centric Refactor)
*   Description:      Correctly cascades the include_in_json state from a
*                     staff member up to its parent study_partner and user.
********************************************************************************/
BEGIN
    IF NEW.include_in_json THEN
        UPDATE private.template_study_partners
        SET include_in_json = TRUE
        WHERE study_partner_bk = NEW.parent_study_partner_bk;

        UPDATE private.template_users
        SET include_in_json = TRUE
        WHERE user_bk = NEW.staff_user_bk;
    END IF;
    RETURN NULL;
END;
$$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_study_staff"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_study_staff"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If a staff member is included, ensures their parent partner and user records are also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_cascade_up_from_visit"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    BEGIN IF NEW.include_in_json THEN UPDATE private.template_study_epochs SET include_in_json = TRUE WHERE epoch_bk = NEW.parent_epoch_bk; END IF; RETURN NULL; END; $$;


ALTER FUNCTION "private"."trg_fn_cascade_up_from_visit"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_cascade_up_from_visit"() IS 'Trigger function for the template authoring system. Why: Enforces upward hierarchical integrity. How: If a visit is included, ensures its parent epoch is also included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_set_initial_membership_inclusion"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
DECLARE
    v_org_included BOOLEAN;
    v_user_included BOOLEAN;
BEGIN
    -- On INSERT, determine the correct initial state of the membership.
    SELECT include_in_json INTO v_org_included
    FROM private.template_organizations
    WHERE organization_bk = NEW.organization_bk;

    SELECT include_in_json INTO v_user_included
    FROM private.template_users
    WHERE user_bk = NEW.user_bk;

    -- The membership is only included if both parents are included.
    NEW.include_in_json := (COALESCE(v_org_included, FALSE) AND COALESCE(v_user_included, FALSE));

    RETURN NEW;
END;
$$;


ALTER FUNCTION "private"."trg_fn_set_initial_membership_inclusion"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_set_initial_membership_inclusion"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new membership record. How: A membership is only included by default if both its parent user and organization are included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_set_payload_stage_number"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO ''
    AS $$
/********************************************************************************
*   Trigger Function: trg_fn_set_payload_stage_number
*   Version:          4.0 (Comprehensive)
*   Author:           Senior AI Database Architect
*   Description:      This definitive version includes a CASE statement that
*                     covers every single template table in the system. This
*                     resolves the bug where stage numbers were appearing as NULL
*                     in the metadata dictionary.
********************************************************************************/
BEGIN
    -- Use a CASE statement to assign the correct stage number based on the table name.
    NEW.stage_number := CASE NEW.template_table_name
        -- Stage 1: Foundational Data (Organizations, Users, Activities, etc.)
        WHEN 'private.template_organizations' THEN 1
        WHEN 'private.template_users' THEN 1
        WHEN 'private.template_memberships' THEN 1
        WHEN 'private.template_reimbursement_types' THEN 1
        WHEN 'private.template_activities' THEN 1
        WHEN 'private.template_budget_categories' THEN 1

        -- Stage 2: Core Portfolio Blueprint (The "what" and "when")
        WHEN 'private.template_studies' THEN 2
        WHEN 'private.template_scenarios' THEN 2
        WHEN 'private.template_sites' THEN 2
        WHEN 'private.template_study_arms' THEN 2
        WHEN 'private.template_study_epochs' THEN 2
        WHEN 'private.template_study_visits' THEN 2
        WHEN 'private.template_amendments' THEN 2
        WHEN 'private.template_forecast_calculation_configs' THEN 2
        WHEN 'private.template_scenario_configurations' THEN 2
        WHEN 'private.template_study_partners' THEN 2
        WHEN 'private.template_study_staff' THEN 2

        -- Stage 3: Blueprint Wiring & Financial Rules (The "how" and "how much")
        WHEN 'private.template_soa_mappings' THEN 3
        WHEN 'private.template_activity_costs' THEN 3

        -- Stage 4: Calculated Enrollment Facts
        WHEN 'private.template_fact_enrollment' THEN 4

        -- Stage 5: Calculated Forecast Facts
        WHEN 'private.template_fact_forecast_detail' THEN 5

        ELSE 99 -- Default for any unknown tables
    END;

    RETURN NEW;
END;
$$;


ALTER FUNCTION "private"."trg_fn_set_payload_stage_number"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_set_payload_stage_number"() IS 'Trigger function for the template authoring system. Why: Assigns a stage number to each materialized payload. How: Ensures the seeder processes tables in the correct hierarchical order to prevent foreign key violations.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_sync_arm_child_state"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    DECLARE v_parent_inc BOOLEAN; BEGIN SELECT include_in_json INTO v_parent_inc FROM private.template_study_arms WHERE arm_bk = NEW.parent_arm_bk; NEW.include_in_json := COALESCE(v_parent_inc, FALSE); RETURN NEW; END; $$;


ALTER FUNCTION "private"."trg_fn_sync_arm_child_state"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_sync_arm_child_state"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new epoch. How: Inherits the state from its parent arm upon creation.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_sync_config_child_state"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    DECLARE v_parent_inc BOOLEAN; BEGIN SELECT include_in_json INTO v_parent_inc FROM private.template_scenario_configurations WHERE scenario_configuration_bk = NEW.parent_scenario_configuration_bk; NEW.include_in_json := COALESCE(v_parent_inc, FALSE); RETURN NEW; END; $$;


ALTER FUNCTION "private"."trg_fn_sync_config_child_state"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_sync_config_child_state"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new record (e.g., arm, site). How: Inherits the state from its parent scenario configuration upon creation.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_sync_cost_inclusion_state"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    /********************************************************************************
    *   Trigger Function: trg_fn_sync_cost_inclusion_state
    *   Version:          2.0 (Configuration-Centric Model Certified)
    *   Author:           Senior AI Database Architect
    *   Description:      This trigger ensures that a cost record's `include_in_json`
    *                     flag is always in sync with the state of its parents.
    *                     This version is updated to navigate the new configuration-
    *                     centric hierarchy, looking up the scenario's state via the
    *                     `template_scenario_configurations` table.
    ********************************************************************************/
    DECLARE
        v_scenario_included BOOLEAN;
        v_activity_included BOOLEAN;
        v_org_included BOOLEAN;
    BEGIN
        -- Look up the state of all parents using the new hierarchy.
        SELECT s.include_in_json INTO v_scenario_included
        FROM private.template_scenarios s
        JOIN private.template_scenario_configurations sc
          ON s.scenario_bk = sc.parent_scenario_bk
        WHERE sc.scenario_configuration_bk = NEW.parent_scenario_configuration_bk;

        SELECT include_in_json INTO v_activity_included
        FROM private.template_activities
        WHERE activity_bk = NEW.activity_bk;

        SELECT include_in_json INTO v_org_included
        FROM private.template_organizations
        WHERE organization_bk = NEW.cost_bearing_partner_bk;

        -- The cost is only included if ALL of its parents are included.
        -- COALESCE to FALSE to handle cases where a parent might not be found.
        NEW.include_in_json := (
            COALESCE(v_scenario_included, FALSE) AND
            COALESCE(v_activity_included, FALSE) AND
            COALESCE(v_org_included, FALSE)
        );

        RETURN NEW;
    END;
    $$;


ALTER FUNCTION "private"."trg_fn_sync_cost_inclusion_state"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_sync_cost_inclusion_state"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new activity cost. How: A cost is only included by default if all its parents (scenario, activity, organization) are included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_sync_epoch_child_state"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    DECLARE v_parent_inc BOOLEAN; BEGIN SELECT include_in_json INTO v_parent_inc FROM private.template_study_epochs WHERE epoch_bk = NEW.parent_epoch_bk; NEW.include_in_json := COALESCE(v_parent_inc, FALSE); RETURN NEW; END; $$;


ALTER FUNCTION "private"."trg_fn_sync_epoch_child_state"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_sync_epoch_child_state"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new visit. How: Inherits the state from its parent epoch upon creation.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_sync_scenario_config_state"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    DECLARE v_study_inc BOOLEAN; v_scen_inc BOOLEAN; BEGIN SELECT include_in_json INTO v_study_inc FROM private.template_studies WHERE study_bk = NEW.parent_study_bk; SELECT include_in_json INTO v_scen_inc FROM private.template_scenarios WHERE scenario_bk = NEW.parent_scenario_bk; NEW.include_in_json := (COALESCE(v_study_inc, FALSE) AND COALESCE(v_scen_inc, FALSE)); RETURN NEW; END; $$;


ALTER FUNCTION "private"."trg_fn_sync_scenario_config_state"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_sync_scenario_config_state"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new scenario configuration. How: A configuration is only included by default if both its parent study and scenario are included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_sync_soa_state"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
    DECLARE v_config_inc BOOLEAN; v_visit_inc BOOLEAN; BEGIN SELECT include_in_json INTO v_config_inc FROM private.template_scenario_configurations WHERE scenario_configuration_bk = NEW.parent_scenario_configuration_bk; SELECT include_in_json INTO v_visit_inc FROM private.template_study_visits WHERE visit_bk = NEW.visit_bk; NEW.include_in_json := (COALESCE(v_config_inc, FALSE) AND COALESCE(v_visit_inc, FALSE)); RETURN NEW; END; $$;


ALTER FUNCTION "private"."trg_fn_sync_soa_state"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_sync_soa_state"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new SoA mapping. How: A mapping is only included by default if both its parent configuration and visit are included.';



CREATE OR REPLACE FUNCTION "private"."trg_fn_sync_study_staff_state"() RETURNS "trigger"
    LANGUAGE "plpgsql" SECURITY DEFINER
    AS $$
/********************************************************************************
*   Trigger Function: trg_fn_sync_site_child_state
*   Version:          2.0 (Partner-Centric Refactor)
*   Description:      Correctly syncs the initial state of a new study_staff
*                     record from its parent study_partner.
********************************************************************************/
DECLARE
    v_parent_inc BOOLEAN;
BEGIN
    SELECT include_in_json INTO v_parent_inc
    FROM private.template_study_partners
    WHERE study_partner_bk = NEW.parent_study_partner_bk;

    NEW.include_in_json := COALESCE(v_parent_inc, FALSE);
    RETURN NEW;
END;
$$;


ALTER FUNCTION "private"."trg_fn_sync_study_staff_state"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."trg_fn_sync_study_staff_state"() IS 'Trigger function for the template authoring system. Why: Sets the initial `include_in_json` state of a new staff assignment. How: Inherits the state from its parent study partner upon creation.';



CREATE OR REPLACE FUNCTION "private"."validate_failures"() RETURNS "void"
    LANGUAGE "plpgsql" SECURITY DEFINER
    SET "search_path" TO 'private', 'public', 'extensions'
    AS $$
/********************************************************************************
*   Function:       private.validate_failures
*   Version:        11.1 (JSON Structure Correction)
*   Author:         Senior AI Business Intelligence Architect
*   Description:    This is the definitive, complete version of the failures
*                   validator. It provides the fix for the malformed JSON in
*                   the failures_report by correcting the aggregation pattern
*                   for all checks. All subqueries now correctly aggregate the
*                   output of jsonb_build_object, producing a clean array of
*                   failure objects, e.g., `[{"check_name": ...}]` instead of
*                   `[{"jsonb_build_object": ...}]`. This resolves all
*                   downstream parsing errors.
********************************************************************************/
DECLARE
    v_materialized_row RECORD;
    v_blueprint_payload JSONB;
    v_failures JSONB;
BEGIN
    RAISE NOTICE '  -> [VALIDATION ENGINE - FAILURES v11.1] Starting...';

    -- Create a single, global payload for cross-table joins
    SELECT COALESCE(jsonb_object_agg(REPLACE(REPLACE(template_table_name, 'private.template_', ''), '_', ''), payload), '{}'::jsonb)
    INTO v_blueprint_payload
    FROM private.materialized_template_payload;

    FOR v_materialized_row IN SELECT * FROM private.materialized_template_payload LOOP
        v_failures := '[]'::jsonb;

        CASE v_materialized_row.template_table_name
            WHEN 'private.template_soa_mappings' THEN
                WITH one_time_anomalies AS (
                    SELECT
                        a.arm_name, a.arm_bk, act.activity_name, act.activity_bk,
                        COUNT(soa.map_soa_bk) AS scheduled_count
                    FROM private.template_soa_mappings soa
                    JOIN private.template_activities act ON soa.activity_bk = act.activity_bk
                    JOIN private.template_study_visits v ON soa.visit_bk = v.visit_bk
                    JOIN private.template_study_epochs e ON v.parent_epoch_bk = e.epoch_bk
                    JOIN private.template_study_arms a ON e.parent_arm_bk = a.arm_bk
                    WHERE act.is_one_time_procedure = TRUE AND soa.include_in_json = TRUE
                    GROUP BY a.arm_name, a.arm_bk, act.activity_name, act.activity_bk
                    HAVING COUNT(soa.map_soa_bk) > 1
                )
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'SoA with Invalid Visit', 'map_soa_bk', soa->>'map_soa_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) soa WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studyvisits') v WHERE v->>'visit_bk' = soa->>'visit_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'SoA with Invalid Activity', 'map_soa_bk', soa->>'map_soa_bk') FROM jsonb_array_elements(v_materialized_row.payload) soa WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'activities') a WHERE a->>'activity_bk' = soa->>'activity_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Orphaned SoA', 'map_soa_bk', soa->>'map_soa_bk') FROM jsonb_array_elements(v_materialized_row.payload) soa WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') cfg WHERE cfg->>'scenario_configuration_bk' = soa->>'parent_scenario_configuration_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'One-Time Procedure Scheduled Multiple Times', 'arm_bk', ota.arm_bk, 'arm_name', ota.arm_name, 'activity_bk', ota.activity_bk, 'activity_name', ota.activity_name, 'scheduled_count', ota.scheduled_count, 'severity', 'error')
                    FROM one_time_anomalies ota
                ) AS sub;

            WHEN 'private.template_sites' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Orphaned Site', 'site_bk', s->>'site_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) s
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') cfg WHERE cfg->>'scenario_configuration_bk' = s->>'parent_scenario_configuration_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Site with Invalid Partner Link', 'site_bk', s->>'site_bk', 'invalid_partner_bk', s->>'site_partner_bk') FROM jsonb_array_elements(v_materialized_row.payload) s
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studypartners') p WHERE p->>'study_partner_bk' = s->>'site_partner_bk')
                ) AS sub;

            WHEN 'private.template_study_staff' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Staff with Invalid Partner', 'study_staff_bk', st->>'study_staff_bk', 'invalid_parent_bk', st->>'parent_study_partner_bk') AS failure_object
                    FROM jsonb_array_elements(v_materialized_row.payload) st
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studypartners') s WHERE s->>'study_partner_bk' = st->>'parent_study_partner_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Staff with Invalid User', 'study_staff_bk', st->>'study_staff_bk')
                    FROM jsonb_array_elements(v_materialized_row.payload) st
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'users') u WHERE u->>'user_bk' = st->>'staff_user_bk')
                ) AS sub;

            WHEN 'private.template_study_partners' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Orphaned Partner', 'study_partner_bk', p->>'study_partner_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) p WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') cfg WHERE cfg->>'scenario_configuration_bk' = p->>'parent_scenario_configuration_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Partner with Invalid Org', 'study_partner_bk', p->>'study_partner_bk') FROM jsonb_array_elements(v_materialized_row.payload) p WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'organizations') o WHERE o->>'organization_bk' = p->>'partner_organization_bk')
                ) AS sub;

            WHEN 'private.template_activity_costs' THEN
                 SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                 FROM (
                    SELECT jsonb_build_object('check_name', 'Cost with Invalid Config', 'cost_bk', c->>'cost_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) c
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') cfg WHERE cfg->>'scenario_configuration_bk' = c->>'parent_scenario_configuration_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Cost with Invalid Activity', 'cost_bk', c->>'cost_bk') FROM jsonb_array_elements(v_materialized_row.payload) c
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'activities') a WHERE a->>'activity_bk' = c->>'activity_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Cost with Invalid Cost Bearer Partner', 'cost_bk', c->>'cost_bk', 'invalid_partner_bk', c->>'cost_bearing_partner_bk') FROM jsonb_array_elements(v_materialized_row.payload) c
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studypartners') p WHERE p->>'study_partner_bk' = c->>'cost_bearing_partner_bk')
                ) AS sub;

            WHEN 'private.template_fact_enrollment' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Enrollment Fact with Invalid Site', 'enrollment_bk', enr->>'enrollment_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) enr WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'sites') s WHERE s->>'site_bk' = enr->>'site_bk')
                    UNION ALL
                    SELECT jsonb_build_object( 'check_name', 'Enrollment Fact with Invalid Parent Configuration', 'enrollment_bk', enr->>'enrollment_bk', 'invalid_config_bk', enr->>'scenario_configuration_bk' ) FROM jsonb_array_elements(v_materialized_row.payload) enr WHERE NOT EXISTS ( SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') scfg WHERE scfg->>'scenario_configuration_bk' = enr->>'scenario_configuration_bk' )
                ) AS sub;

            WHEN 'private.template_forecast_calculation_configs' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Config Rule with Invalid Config', 'config_bk', cfg->>'config_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) cfg WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') scfg WHERE scfg->>'scenario_configuration_bk' = cfg->>'parent_scenario_configuration_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Config Rule with Invalid Activity', 'config_bk', cfg->>'config_bk') FROM jsonb_array_elements(v_materialized_row.payload) cfg WHERE cfg->>'activity_bk' IS NOT NULL AND NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'activities') a WHERE a->>'activity_bk' = cfg->>'activity_bk')
                ) AS sub;

            WHEN 'private.template_scenario_configurations' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Invalid Date Logic', 'config_bk', sc->>'scenario_configuration_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) sc WHERE (sc->>'end_date_offset_days')::int <= (sc->>'start_date_offset_days')::int
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Orphaned Config (Study)', 'config_bk', sc->>'scenario_configuration_bk') FROM jsonb_array_elements(v_materialized_row.payload) sc WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studies') s WHERE s->>'study_bk' = sc->>'parent_study_bk')
                    UNION ALL
                    SELECT jsonb_build_object('check_name', 'Orphaned Config (Scenario)', 'config_bk', sc->>'scenario_configuration_bk') FROM jsonb_array_elements(v_materialized_row.payload) sc WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarios') s WHERE s->>'scenario_bk' = sc->>'parent_scenario_bk')
                ) AS sub;

            WHEN 'private.template_study_arms' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Orphaned Arm', 'arm_bk', arm->>'arm_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) arm WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'scenarioconfigurations') cfg WHERE cfg->>'scenario_configuration_bk' = arm->>'parent_scenario_configuration_bk')
                ) AS sub;

            WHEN 'private.template_study_epochs' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Orphaned Epoch', 'epoch_bk', epoch->>'epoch_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) epoch WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studyarms') a WHERE a->>'arm_bk' = epoch->>'parent_arm_bk')
                ) AS sub;

            WHEN 'private.template_study_visits' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Orphaned Visit', 'visit_bk', visit->>'visit_bk') AS failure_object FROM jsonb_array_elements(v_materialized_row.payload) visit WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'studyepochs') e WHERE e->>'epoch_bk' = visit->>'parent_epoch_bk')
                ) AS sub;

            WHEN 'private.template_activities' THEN
                SELECT COALESCE(jsonb_agg(failure_object), '[]'::jsonb) INTO v_failures
                FROM (
                    SELECT jsonb_build_object('check_name', 'Activity with Invalid Budget Category', 'activity_bk', a->>'activity_bk', 'invalid_budget_category_bk', a->>'budget_category_bk') AS failure_object
                    FROM jsonb_array_elements(v_materialized_row.payload) a
                    WHERE NOT EXISTS (SELECT 1 FROM jsonb_array_elements(v_blueprint_payload->'budgetcategories') bc WHERE bc->>'budget_category_bk' = a->>'budget_category_bk')
                ) AS sub;

            ELSE
                -- No failure checks for this table
        END CASE;

        UPDATE private.materialized_template_payload
        SET failures_report = v_failures
        WHERE id = v_materialized_row.id;
    END LOOP;
END;
$$;


ALTER FUNCTION "private"."validate_failures"() OWNER TO "postgres";


COMMENT ON FUNCTION "private"."validate_failures"() IS 'The validation engine that identifies definitive data integrity violations (e.g., orphaned records, broken foreign key links). Why: Acts as the "Compiler Errors" system for the IDE, identifying structural flaws that would cause a forecast to fail.';




CREATE TABLE IF NOT EXISTS "private"."materialized_template_payload" (
    "id" integer NOT NULL,
    "template_table_name" "text" NOT NULL,
    "public_table_name" "text" NOT NULL,
    "payload" "jsonb" NOT NULL,
    "last_refreshed_at" timestamp with time zone DEFAULT "now"() NOT NULL,
    "metadata" "jsonb",
    "stage_number" integer,
    "failures_report" "jsonb",
    "business_logic_report" "jsonb",
    "gaps_report" "jsonb"
);


ALTER TABLE "private"."materialized_template_payload" OWNER TO "postgres";


COMMENT ON TABLE "private"."materialized_template_payload" IS 'Stores per-template/table materialized payloads and derived reports. Why: decouples heavy calculations from read paths. How: each row holds JSON payload, metadata, and BI/gaps outputs for a specific template/public mapping.';



COMMENT ON COLUMN "private"."materialized_template_payload"."id" IS 'Surrogate key for the materialized row.';



COMMENT ON COLUMN "private"."materialized_template_payload"."template_table_name" IS 'Name of the source template table (e.g., private.template_studies). Why: identifies data provenance.';



COMMENT ON COLUMN "private"."materialized_template_payload"."public_table_name" IS 'Name of the corresponding public/dimension/fact table. Why: used to link payloads to public schema constructs.';



COMMENT ON COLUMN "private"."materialized_template_payload"."payload" IS 'JSONB array/object holding the materialized data. How: built by cache engine and used by BI/gaps functions.';



COMMENT ON COLUMN "private"."materialized_template_payload"."last_refreshed_at" IS 'Timestamp of the last time the cache for this table was successfully refreshed.';



COMMENT ON COLUMN "private"."materialized_template_payload"."metadata" IS 'JSONB with materialization metadata such as counts. Why: used for QC and summaries.';



COMMENT ON COLUMN "private"."materialized_template_payload"."stage_number" IS 'The processing stage for the seeder orchestrator. Why: Ensures data is loaded in the correct hierarchical order to prevent foreign key violations.';



COMMENT ON COLUMN "private"."materialized_template_payload"."failures_report" IS 'JSONB array of data integrity violations (e.g., broken foreign keys). An empty array signifies the data is structurally sound, like a "compiler error" report.';



COMMENT ON COLUMN "private"."materialized_template_payload"."business_logic_report" IS 'JSONB BI results for the row''s template table. Why: consumed by UI analytics.';



COMMENT ON COLUMN "private"."materialized_template_payload"."gaps_report" IS 'JSONB list of data gaps or warnings. Why: guides authors to complete blueprint structure.';



CREATE SEQUENCE IF NOT EXISTS "private"."materialized_template_payload_id_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE "private"."materialized_template_payload_id_seq" OWNER TO "postgres";


ALTER SEQUENCE "private"."materialized_template_payload_id_seq" OWNED BY "private"."materialized_template_payload"."id";



CREATE TABLE IF NOT EXISTS "private"."sync_log" (
    "sync_type" "text" NOT NULL,
    "last_synced_at" timestamp with time zone NOT NULL
);


ALTER TABLE "private"."sync_log" OWNER TO "postgres";


COMMENT ON TABLE "private"."sync_log" IS 'Tracks the last execution time of major synchronization jobs (e.g., Clerk sync). Why: Used to throttle frequent, expensive operations and prevent redundant work by checking if a recent sync has already occurred.';



COMMENT ON COLUMN "private"."sync_log"."sync_type" IS 'The type of synchronization job being logged (e.g., ''CLERK_GLOBAL_SYNC''). Acts as the primary key.';



COMMENT ON COLUMN "private"."sync_log"."last_synced_at" IS 'The timestamp of the last successful execution of this synchronization job.';



CREATE TABLE IF NOT EXISTS "private"."template_activities" (
    "activity_bk" "text" NOT NULL,
    "activity_name" "text" NOT NULL,
    "activity_type" "text" NOT NULL,
    "activity_domain" "text" NOT NULL,
    "activity_category" "text" NOT NULL,
    "standard_code" "text",
    "description" "text",
    "is_billable" boolean DEFAULT true,
    "reimbursement_type_bk" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    "is_one_time_procedure" boolean DEFAULT false,
    "budget_category_bk" "text" NOT NULL,
    CONSTRAINT "chk_activity_type_enum" CHECK (("activity_type" = ANY (ARRAY['PROCEDURE'::"text", 'ASSESSMENT'::"text", 'LABORATORY'::"text", 'LOGISTICS'::"text", 'ADMINISTRATIVE'::"text", 'TECHNOLOGY'::"text", 'EVENT'::"text"])))
);


ALTER TABLE "private"."template_activities" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_activities" IS 'The master catalog of all possible clinical trial activities for template data. Why: Provides a standardized, reusable library of procedures, assessments, and events for building study blueprints.';



COMMENT ON COLUMN "private"."template_activities"."activity_bk" IS 'The business key for the activity, used for idempotent seeding and cross-environment references.';



COMMENT ON COLUMN "private"."template_activities"."activity_name" IS 'The human-readable name of the activity (e.g., "Patient Screening Visit").';



COMMENT ON COLUMN "private"."template_activities"."activity_type" IS 'Classifies the operational nature of the activity (e.g., ''PROCEDURE'', ''ASSESSMENT''). Drives scheduling and reporting logic.';



COMMENT ON COLUMN "private"."template_activities"."activity_domain" IS 'A high-level functional grouping for the activity (e.g., "Site Operations", "Patient & Procedures"). Used for BI rollups.';



COMMENT ON COLUMN "private"."template_activities"."activity_category" IS 'A more granular, user-defined category for the activity.';



COMMENT ON COLUMN "private"."template_activities"."standard_code" IS 'An optional standard medical or procedural code (e.g., CPT, LOINC) for interoperability.';



COMMENT ON COLUMN "private"."template_activities"."description" IS 'A detailed description of the activity and its purpose.';



COMMENT ON COLUMN "private"."template_activities"."is_billable" IS 'Flag indicating if this activity typically incurs a cost. If false, it is treated as a zero-cost operational event.';



COMMENT ON COLUMN "private"."template_activities"."reimbursement_type_bk" IS 'Links to the default reimbursement policy for this activity, which determines the financial multiplier applied to its cost.';



COMMENT ON COLUMN "private"."template_activities"."include_in_json" IS 'Internal flag for the template authoring system to control which records are included in the final materialized payload.';



COMMENT ON COLUMN "private"."template_activities"."is_one_time_procedure" IS 'Flag indicating if this activity should only occur once per subject within a given study arm. Used by the validation engine to detect scheduling anomalies.';



COMMENT ON COLUMN "private"."template_activities"."budget_category_bk" IS 'Links this operational activity to a financial account in the budget hierarchy for cost rollups and reporting.';



COMMENT ON CONSTRAINT "chk_activity_type_enum" ON "private"."template_activities" IS 'CHECK constraint to enforce that the activity_type value is a valid member of the activity_type_enum, ensuring data consistency.';



CREATE TABLE IF NOT EXISTS "private"."template_activity_costs" (
    "cost_bk" "text" NOT NULL,
    "activity_bk" "text" NOT NULL,
    "cost_unit" "text" NOT NULL,
    "unit_cost" numeric(15,2) NOT NULL,
    "currency_code" "text" DEFAULT 'USD'::"text",
    "effective_date_offset_days" integer DEFAULT 0,
    "end_date_offset_days" integer,
    "reimbursement_type_bk" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_scenario_configuration_bk" "text" NOT NULL,
    "cost_bearing_partner_bk" "text" NOT NULL,
    "payer_partner_bk" "text"
);


ALTER TABLE "private"."template_activity_costs" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_activity_costs" IS 'Defines the unit costs for activities within a specific template scenario configuration. Why: Acts as the financial source of truth for pricing activities in a template plan before it is seeded into a user sandbox.';



COMMENT ON COLUMN "private"."template_activity_costs"."cost_bk" IS 'The business key for the activity cost record.';



COMMENT ON COLUMN "private"."template_activity_costs"."activity_bk" IS 'Links this cost record to the specific activity being priced.';



COMMENT ON COLUMN "private"."template_activity_costs"."cost_unit" IS 'Specifies the unit of measure for the `unit_cost` (e.g., ''Per Visit'', ''Per Subject'').';



COMMENT ON COLUMN "private"."template_activity_costs"."unit_cost" IS 'The cost for a single unit of the specified `cost_unit`.';



COMMENT ON COLUMN "private"."template_activity_costs"."currency_code" IS 'The ISO currency code for the unit cost (e.g., ''USD'').';



COMMENT ON COLUMN "private"."template_activity_costs"."effective_date_offset_days" IS 'The number of days from the seeder run date that this cost becomes effective.';



COMMENT ON COLUMN "private"."template_activity_costs"."end_date_offset_days" IS 'The number of days from the seeder run date that this cost is no longer effective. NULL indicates no end date.';



COMMENT ON COLUMN "private"."template_activity_costs"."reimbursement_type_bk" IS 'Links to a specific reimbursement policy that applies a financial multiplier to this cost.';



COMMENT ON COLUMN "private"."template_activity_costs"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_activity_costs"."parent_scenario_configuration_bk" IS 'Links this cost record to a specific plan (a study-scenario combination), ensuring pricing is versioned and scoped correctly.';



COMMENT ON COLUMN "private"."template_activity_costs"."cost_bearing_partner_bk" IS 'Identifies the study partner (the payee) who will receive payment for this event.';



COMMENT ON COLUMN "private"."template_activity_costs"."payer_partner_bk" IS 'Identifies the study partner (the payer, typically the Sponsor) responsible for paying for this event.';



CREATE TABLE IF NOT EXISTS "private"."template_amendments" (
    "amendment_bk" "text" NOT NULL,
    "parent_study_bk" "text" NOT NULL,
    "amendment_version_id" "text" NOT NULL,
    "approval_date_offset_days" integer,
    "effective_date_offset_days" integer NOT NULL,
    "summary" "text" NOT NULL,
    "amendment_status" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    CONSTRAINT "chk_amendment_status_enum" CHECK (("amendment_status" = ANY (ARRAY['Draft'::"text", 'In Review'::"text", 'Approved'::"text", 'Implemented'::"text"])))
);


ALTER TABLE "private"."template_amendments" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_amendments" IS 'Stores template protocol amendments that can be applied to studies. Why: Allows for the modeling of how a study plan changes over time, which can trigger the creation of new "What-If" scenarios.';



COMMENT ON COLUMN "private"."template_amendments"."amendment_bk" IS 'The business key for the amendment record.';



COMMENT ON COLUMN "private"."template_amendments"."parent_study_bk" IS 'Links this amendment to the parent study it modifies.';



COMMENT ON COLUMN "private"."template_amendments"."amendment_version_id" IS 'The unique, human-readable identifier for the amendment (e.g., "v2.0").';



COMMENT ON COLUMN "private"."template_amendments"."approval_date_offset_days" IS 'The number of days from the seeder run date that this amendment is considered approved.';



COMMENT ON COLUMN "private"."template_amendments"."effective_date_offset_days" IS 'The number of days from the seeder run date that this amendment takes effect.';



COMMENT ON COLUMN "private"."template_amendments"."summary" IS 'A summary of the changes introduced by this amendment.';



COMMENT ON COLUMN "private"."template_amendments"."amendment_status" IS 'Tracks the lifecycle stage of the amendment (''Draft'', ''In Review'', ''Approved'', ''Implemented'').';



COMMENT ON COLUMN "private"."template_amendments"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON CONSTRAINT "chk_amendment_status_enum" ON "private"."template_amendments" IS 'CHECK constraint to enforce that the amendment_status value is a valid member of the amendment_status_enum.';



CREATE TABLE IF NOT EXISTS "private"."template_budget_categories" (
    "budget_category_bk" "text" NOT NULL,
    "category_name" "text" NOT NULL,
    "parent_category_bk" "text",
    "is_rollup_target" boolean DEFAULT false,
    "description" "text",
    "include_in_json" boolean DEFAULT true NOT NULL
);


ALTER TABLE "private"."template_budget_categories" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_budget_categories" IS 'The hierarchical chart of accounts for template data. Why: Provides the foundational structure for financial rollups and reporting within the template portfolio.';



COMMENT ON COLUMN "private"."template_budget_categories"."budget_category_bk" IS 'The business key for the budget category.';



COMMENT ON COLUMN "private"."template_budget_categories"."category_name" IS 'The human-readable name of the budget category (e.g., "Site Pass-Through Costs").';



COMMENT ON COLUMN "private"."template_budget_categories"."parent_category_bk" IS 'Links this category to its parent in the budget hierarchy. A NULL value indicates a top-level category.';



COMMENT ON COLUMN "private"."template_budget_categories"."is_rollup_target" IS 'A flag indicating that this category is a significant endpoint for financial summary reports.';



COMMENT ON COLUMN "private"."template_budget_categories"."description" IS 'A detailed description of the types of costs included in this category.';



COMMENT ON COLUMN "private"."template_budget_categories"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



CREATE TABLE IF NOT EXISTS "private"."template_fact_enrollment" (
    "site_bk" "text" NOT NULL,
    "enrollment_bk" "text" NOT NULL,
    "projected_enrollment_count" integer DEFAULT 0 NOT NULL,
    "actual_enrollment_count" integer DEFAULT 0 NOT NULL,
    "is_asserted_actual" boolean DEFAULT false NOT NULL,
    "last_edit_source" "text" DEFAULT 'SYSTEM_CALCULATION'::"text" NOT NULL,
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_scenario_configuration_bk" "text" NOT NULL,
    "snapshot_date_offset_days" integer NOT NULL
);


ALTER TABLE "private"."template_fact_enrollment" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_fact_enrollment" IS 'Template enrollment facts by site and month. Why: foundation for schedule-based forecasting and capacity planning. How: built by enrollment engine using offsets from current_date.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."site_bk" IS 'Links this enrollment fact to the specific clinical site where the subjects were enrolled. Denormalized for lineage and cross-environment joins.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."enrollment_bk" IS 'Stable BK for the fact row. Why: supports adoption and idempotent upserts.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."projected_enrollment_count" IS 'The number of subjects forecasted to be enrolled at this site during the snapshot month, as calculated by the enrollment engine.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."actual_enrollment_count" IS 'The number of subjects confirmed to have been enrolled at this site during the snapshot month. This value is typically ingested from an external system or entered manually.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."is_asserted_actual" IS 'A flag indicating that the `actual_enrollment_count` was provided by a user or an external data feed, overriding any system calculations. This value is protected from being overwritten by the engine.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."last_edit_source" IS 'Tracks the provenance of the record, either ''SYSTEM_CALCULATION'' or ''USER_EDIT''. This is critical for protecting user-overridden data from being overwritten by automated processes.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."parent_scenario_configuration_bk" IS 'Links this enrollment fact to the specific scenario configuration (the plan) it belongs to. This is the primary key for scoping all calculations.';



COMMENT ON COLUMN "private"."template_fact_enrollment"."snapshot_date_offset_days" IS 'Offset days from current_date to the snapshot month. Why: allows deterministic regeneration and compact storage.';



CREATE TABLE IF NOT EXISTS "private"."template_fact_forecast_detail" (
    "forecast_detail_bk" "text" NOT NULL,
    "snapshot_date_sk" "date" NOT NULL,
    "activity_bk" "text" NOT NULL,
    "payee_partner_bk" "text",
    "cost_unit" "text" NOT NULL,
    "unit_cost" numeric(15,2) NOT NULL,
    "forecast_units" numeric(18,2) NOT NULL,
    "reimbursement_factor" numeric(10,4) DEFAULT 1.0000 NOT NULL,
    "site_bk" "text",
    "source_visit_bk" "text",
    "source_enrollment_bk" "text",
    "source_epoch_bk" "text",
    "source_forecast_config_bk" "text",
    "is_asserted_actual" boolean DEFAULT false NOT NULL,
    "last_edit_source" "text" DEFAULT 'SYSTEM_CALCULATION'::"text" NOT NULL,
    "include_in_json" boolean DEFAULT true NOT NULL,
    "source_activity_cost_bk" "text",
    "payer_partner_bk" "text",
    "source_map_soa_bk" "text",
    "source_arm_bk" "text",
    "original_scenario_bk" "text",
    "original_study_bk" "text",
    "parent_scenario_configuration_bk" "text" NOT NULL
);


ALTER TABLE "private"."template_fact_forecast_detail" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_fact_forecast_detail" IS 'Template forecast detail facts at the lowest grain. Why: supports financial rollups and scenario comparisons. How: generated by forecast engine with lineage keys.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."forecast_detail_bk" IS 'Stable BK for the forecast fact row (supports adoption/upsert).';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."snapshot_date_sk" IS 'The specific date (typically the first of the month) to which this financial or operational event is attributed. Links to the date dimension for time-series analysis.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."activity_bk" IS 'Links to the specific procedure, assessment, or other event from the activity catalog that this forecast record represents.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."payee_partner_bk" IS 'Identifies the study partner (e.g., a clinical site or vendor) who will receive payment for this event.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."cost_unit" IS 'Specifies the unit of measure for the `unit_cost` (e.g., ''Per Visit'', ''Per Subject''). Provides the context for the financial calculation.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."unit_cost" IS 'The cost for a single unit of the specified `cost_unit`. This is the base price before any reimbursement factors are applied.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."forecast_units" IS 'The quantity of units (e.g., number of subjects, number of procedures) forecasted for this event during the snapshot period.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."reimbursement_factor" IS 'A multiplier (e.g., 1.0 for full cost, 0.8 for a 20% discount) applied to the gross cost to calculate the net cost, based on the reimbursement policy.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."site_bk" IS 'Links to the clinical site where this event occurred. This is NULL for events that are not site-specific (e.g., monthly CRO management fees).';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."source_visit_bk" IS 'Lineage: originating visit BK for schedule-based events.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."source_enrollment_bk" IS 'Lineage: originating enrollment BK when relevant.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."source_epoch_bk" IS 'Lineage: epoch BK for context.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."source_forecast_config_bk" IS 'Lineage: forecast config rule BK.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."is_asserted_actual" IS 'TRUE when the record is a user-asserted historical actual.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."last_edit_source" IS 'Audit: identifies system vs user source.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."source_activity_cost_bk" IS 'Lineage: activity cost BK applied to price.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."payer_partner_bk" IS 'Identifies the study partner (typically the Sponsor) who is responsible for paying for this event.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."source_map_soa_bk" IS 'Lineage: SoA mapping BK connecting visit and activity.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."source_arm_bk" IS 'Lineage: arm BK used in weighting.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."original_scenario_bk" IS 'Original scenario BK used for BI lineage and joins.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."original_study_bk" IS 'Original study BK used for BI lineage and joins.';



COMMENT ON COLUMN "private"."template_fact_forecast_detail"."parent_scenario_configuration_bk" IS 'Owning scenario configuration BK.';



CREATE TABLE IF NOT EXISTS "private"."template_forecast_calculation_configs" (
    "config_bk" "text" NOT NULL,
    "config_name" "text" NOT NULL,
    "activity_bk" "text",
    "calculation_method" "text" NOT NULL,
    "is_group_definition" boolean DEFAULT false,
    "enrollment_curve_type" "text",
    "curve_points" "jsonb",
    "cost_unit" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_scenario_configuration_bk" "text" NOT NULL,
    CONSTRAINT "chk_calculation_method_enum" CHECK (("calculation_method" = ANY (ARRAY['DIRECT_MONTHLY'::"text", 'ENROLLMENT_BASED'::"text", 'SCHEDULE_BASED'::"text", 'ENROLLMENT_CURVE_BASED'::"text"])))
);


ALTER TABLE "private"."template_forecast_calculation_configs" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_forecast_calculation_configs" IS 'Defines the rules that determine how costs are calculated in a template. Why: This is the logic engine for the forecast, specifying whether a cost is schedule-based, enrollment-based, or follows a direct monthly pattern.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."config_bk" IS 'The business key for the forecast calculation rule.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."config_name" IS 'The human-readable name of the rule (e.g., "Monthly CRO Fee").';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."activity_bk" IS 'Links this rule to a specific activity. Required for SCHEDULE_BASED rules.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."calculation_method" IS 'The core logic for the forecast engine: ''DIRECT_MONTHLY'', ''ENROLLMENT_BASED'', ''SCHEDULE_BASED'', or ''ENROLLMENT_CURVE_BASED''.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."is_group_definition" IS 'Flag indicating if this rule defines a performance group (e.g., an enrollment curve) that sites can be assigned to.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."enrollment_curve_type" IS 'A descriptive name for the enrollment curve (e.g., "Standard S-Curve").';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."curve_points" IS 'A JSONB array defining the percentage of total enrollment to be achieved each month for curve-based forecasting.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."cost_unit" IS 'An optional filter to apply this rule only to costs with a matching cost unit.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_forecast_calculation_configs"."parent_scenario_configuration_bk" IS 'Links this rule to a specific plan (a study-scenario combination).';



COMMENT ON CONSTRAINT "chk_calculation_method_enum" ON "private"."template_forecast_calculation_configs" IS 'CHECK constraint to enforce that the calculation_method value is a valid member of the calculation_method_enum.';



CREATE TABLE IF NOT EXISTS "private"."template_memberships" (
    "user_bk" "text" NOT NULL,
    "organization_bk" "text" NOT NULL,
    "role" "text" NOT NULL,
    "include_in_json" boolean DEFAULT true NOT NULL
);


ALTER TABLE "private"."template_memberships" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_memberships" IS 'Links template users to template organizations with a specific role. Why: Models the staffing, responsibilities, and access control within the template portfolio.';



COMMENT ON COLUMN "private"."template_memberships"."user_bk" IS 'Links this membership to a specific user in the template data.';



COMMENT ON COLUMN "private"."template_memberships"."organization_bk" IS 'Links this membership to a specific organization in the template data.';



COMMENT ON COLUMN "private"."template_memberships"."role" IS 'The role assigned to the user within the organization (e.g., ''Admin'', ''Member'').';



COMMENT ON COLUMN "private"."template_memberships"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



CREATE MATERIALIZED VIEW "private"."template_metadata_dictionary" AS
 SELECT "template_table_name",
    "public_table_name",
    "metadata",
    "failures_report",
    "gaps_report",
    "business_logic_report",
    "last_refreshed_at",
    "stage_number",
    "round"((("pg_column_size"("payload"))::numeric / 1024.0), 2) AS "estimated_payload_size_kb",
    ("pg_column_size"("payload") / 4) AS "estimated_token_count"
   FROM "private"."materialized_template_payload" "p"
  WITH NO DATA;


ALTER MATERIALIZED VIEW "private"."template_metadata_dictionary" OWNER TO "postgres";


COMMENT ON MATERIALIZED VIEW "private"."template_metadata_dictionary" IS 'A materialized view summarizing the state of the template data cache. Why: Provides a fast, consolidated view for the seeder and validation engines to get metadata and reports without re-calculating.';



CREATE TABLE IF NOT EXISTS "private"."template_organizations" (
    "organization_bk" "text" NOT NULL,
    "organization_name" "text" NOT NULL,
    "organization_type" "text" NOT NULL,
    "country_code" "text",
    "region" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    CONSTRAINT "chk_organization_type_enum" CHECK (("organization_type" = ANY (ARRAY['CRO'::"text", 'Lab'::"text", 'Other'::"text", 'Site'::"text", 'Sponsor'::"text", 'Technology'::"text", 'Vendor'::"text"])))
);


ALTER TABLE "private"."template_organizations" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_organizations" IS 'A directory of all fictitious organizations used in the template data. Why: Represents the ecosystem of potential partners (Sponsors, CROs, Sites, Vendors) available for building study plans.';



COMMENT ON COLUMN "private"."template_organizations"."organization_bk" IS 'The business key for the organization.';



COMMENT ON COLUMN "private"."template_organizations"."organization_name" IS 'The name of the organization.';



COMMENT ON COLUMN "private"."template_organizations"."organization_type" IS 'Classifies the primary business function of the organization (e.g., ''Sponsor'', ''Site'', ''CRO'').';



COMMENT ON COLUMN "private"."template_organizations"."country_code" IS 'The ISO 3166-1 alpha-2 country code for the organization.';



COMMENT ON COLUMN "private"."template_organizations"."region" IS 'The state, province, or region where the organization is located.';



COMMENT ON COLUMN "private"."template_organizations"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON CONSTRAINT "chk_organization_type_enum" ON "private"."template_organizations" IS 'CHECK constraint to enforce that the organization_type value is a valid member of the organization_type_enum.';



CREATE TABLE IF NOT EXISTS "private"."template_reimbursement_types" (
    "reimbursement_type_bk" "text" NOT NULL,
    "reimbursement_type_code" "text" NOT NULL,
    "reimbursement_type_name" "text" NOT NULL,
    "financial_impact" "text" NOT NULL,
    "is_default" boolean DEFAULT false,
    "description" "text",
    "include_in_json" boolean DEFAULT true NOT NULL
);


ALTER TABLE "private"."template_reimbursement_types" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_reimbursement_types" IS 'Defines financial reimbursement policies and their corresponding multipliers for template data. Why: Models financial scenarios like discounts, holdbacks, or pass-through costs.';



COMMENT ON COLUMN "private"."template_reimbursement_types"."reimbursement_type_bk" IS 'The business key for the reimbursement type.';



COMMENT ON COLUMN "private"."template_reimbursement_types"."reimbursement_type_code" IS 'A short, unique code for the reimbursement type (e.g., "STD_REIMB").';



COMMENT ON COLUMN "private"."template_reimbursement_types"."reimbursement_type_name" IS 'The human-readable name of the policy (e.g., "Standard Reimbursement").';



COMMENT ON COLUMN "private"."template_reimbursement_types"."financial_impact" IS 'A numeric multiplier applied to a cost to calculate the net financial impact (e.g., 1.0 for full cost, 0.8 for a 20% discount).';



COMMENT ON COLUMN "private"."template_reimbursement_types"."is_default" IS 'Flag indicating if this is the default reimbursement policy for the portfolio.';



COMMENT ON COLUMN "private"."template_reimbursement_types"."description" IS 'A detailed description of the policy.';



COMMENT ON COLUMN "private"."template_reimbursement_types"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



CREATE TABLE IF NOT EXISTS "private"."template_scenario_configurations" (
    "parent_scenario_bk" "text" NOT NULL,
    "start_date_offset_days" integer,
    "end_date_offset_days" integer,
    "target_enrollment" integer,
    "target_sites" integer,
    "study_status" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_study_bk" "text" NOT NULL,
    "scenario_configuration_bk" "text" NOT NULL,
    CONSTRAINT "chk_study_status_enum" CHECK (("study_status" = ANY (ARRAY['Active'::"text", 'Closed'::"text", 'Completed'::"text", 'Enrolling'::"text", 'Planning'::"text", 'Suspended'::"text", 'Terminated'::"text"])))
);


ALTER TABLE "private"."template_scenario_configurations" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_scenario_configurations" IS 'The central hub linking a template study to a template scenario. Why: Creates a concrete, versioned plan to which all other blueprint components (arms, sites, costs) are attached, forming the root of a complete forecastable model.';



COMMENT ON COLUMN "private"."template_scenario_configurations"."parent_scenario_bk" IS 'Links this configuration to a parent scenario (e.g., "Baseline").';



COMMENT ON COLUMN "private"."template_scenario_configurations"."start_date_offset_days" IS 'The number of days from the seeder run date that the study plan starts.';



COMMENT ON COLUMN "private"."template_scenario_configurations"."end_date_offset_days" IS 'The number of days from the seeder run date that the study plan ends.';



COMMENT ON COLUMN "private"."template_scenario_configurations"."target_enrollment" IS 'The total number of subjects to be enrolled across all arms in this plan.';



COMMENT ON COLUMN "private"."template_scenario_configurations"."target_sites" IS 'The total number of clinical sites planned for this study scenario.';



COMMENT ON COLUMN "private"."template_scenario_configurations"."study_status" IS 'The operational status of the study within this plan (e.g., ''Planning'', ''Enrolling'').';



COMMENT ON COLUMN "private"."template_scenario_configurations"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_scenario_configurations"."parent_study_bk" IS 'Links this configuration to a parent study protocol.';



COMMENT ON COLUMN "private"."template_scenario_configurations"."scenario_configuration_bk" IS 'The business key for this specific study-scenario plan.';



COMMENT ON CONSTRAINT "chk_study_status_enum" ON "private"."template_scenario_configurations" IS 'CHECK constraint to enforce that the study_status value is a valid member of the study_status_enum.';



CREATE TABLE IF NOT EXISTS "private"."template_scenarios" (
    "scenario_bk" "text" NOT NULL,
    "scenario_name" "text" NOT NULL,
    "scenario_type" "text" NOT NULL,
    "triggering_amendment_bk" "text",
    "description" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    CONSTRAINT "chk_scenario_type_enum" CHECK (("scenario_type" = ANY (ARRAY['Actual'::"text", 'Approved Budget'::"text", 'Baseline'::"text", 'Budget'::"text", 'Forecast'::"text", 'What-If'::"text"])))
);


ALTER TABLE "private"."template_scenarios" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_scenarios" IS 'A catalog of template scenarios (e.g., "Baseline", "What-If", "Approved Budget"). Why: Allows for the comparison of different planning assumptions and financial models within the template data.';



COMMENT ON COLUMN "private"."template_scenarios"."scenario_bk" IS 'The business key for the scenario.';



COMMENT ON COLUMN "private"."template_scenarios"."scenario_name" IS 'The human-readable name of the scenario (e.g., "2024 Approved Budget").';



COMMENT ON COLUMN "private"."template_scenarios"."scenario_type" IS 'Classifies the business purpose of the scenario (e.g., ''Baseline'', ''Forecast'', ''What-If'').';



COMMENT ON COLUMN "private"."template_scenarios"."triggering_amendment_bk" IS 'An optional link to an amendment that caused this scenario to be created.';



COMMENT ON COLUMN "private"."template_scenarios"."description" IS 'A detailed description of the assumptions and purpose of this scenario.';



COMMENT ON COLUMN "private"."template_scenarios"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON CONSTRAINT "chk_scenario_type_enum" ON "private"."template_scenarios" IS 'CHECK constraint to enforce that the scenario_type value is a valid member of the budget_scenario_type_enum.';



CREATE TABLE IF NOT EXISTS "private"."template_sites" (
    "target_enrollment" integer,
    "activation_date_offset_days" integer,
    "closeout_date_offset_days" integer,
    "site_status" "text",
    "performance_group_config_bk" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_scenario_configuration_bk" "text" NOT NULL,
    "site_bk" "text" NOT NULL,
    "site_partner_bk" "text" NOT NULL,
    CONSTRAINT "chk_site_status_enum" CHECK (("site_status" = ANY (ARRAY['Planned'::"text", 'Initiating'::"text", 'Active'::"text", 'Enrolling'::"text", 'Closed'::"text", 'Suspended'::"text"])))
);


ALTER TABLE "private"."template_sites" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_sites" IS 'Defines the clinical sites participating in a specific template scenario configuration. Why: Drives enrollment projections and site-specific costs within a template plan.';



COMMENT ON COLUMN "private"."template_sites"."target_enrollment" IS 'The number of subjects this site is expected to enroll for this plan.';



COMMENT ON COLUMN "private"."template_sites"."activation_date_offset_days" IS 'The number of days from the seeder run date that this site is expected to be activated.';



COMMENT ON COLUMN "private"."template_sites"."closeout_date_offset_days" IS 'The number of days from the seeder run date that this site is expected to be closed out.';



COMMENT ON COLUMN "private"."template_sites"."site_status" IS 'The operational status of the site within this plan (e.g., ''Planned'', ''Active'').';



COMMENT ON COLUMN "private"."template_sites"."performance_group_config_bk" IS 'Links this site to an enrollment curve definition, allowing for modeling of different enrollment speeds (e.g., "High-Enrolling", "Standard").';



COMMENT ON COLUMN "private"."template_sites"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_sites"."parent_scenario_configuration_bk" IS 'Links this site assignment to a specific plan.';



COMMENT ON COLUMN "private"."template_sites"."site_bk" IS 'The business key for the site assignment record.';



COMMENT ON COLUMN "private"."template_sites"."site_partner_bk" IS 'Links this site assignment to the study partner record that represents the site organization.';



COMMENT ON CONSTRAINT "chk_site_status_enum" ON "private"."template_sites" IS 'CHECK constraint to enforce that the site_status value is a valid member of the site_status_enum.';



CREATE TABLE IF NOT EXISTS "private"."template_soa_mappings" (
    "visit_bk" "text" NOT NULL,
    "activity_bk" "text" NOT NULL,
    "is_required" boolean DEFAULT true,
    "notes" "text",
    "map_soa_bk" "text" NOT NULL,
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_scenario_configuration_bk" "text" NOT NULL
);


ALTER TABLE "private"."template_soa_mappings" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_soa_mappings" IS 'The template for the Schedule of Activities (SoA). Why: Defines which activities occur at which visits for a given template plan, forming the core of schedule-based forecasting.';



COMMENT ON COLUMN "private"."template_soa_mappings"."visit_bk" IS 'Links this SoA entry to a specific visit in the schedule.';



COMMENT ON COLUMN "private"."template_soa_mappings"."activity_bk" IS 'Links this SoA entry to a specific activity in the catalog.';



COMMENT ON COLUMN "private"."template_soa_mappings"."is_required" IS 'Flag indicating if the activity is mandatory for this visit.';



COMMENT ON COLUMN "private"."template_soa_mappings"."notes" IS 'Optional notes about this specific visit-activity combination.';



COMMENT ON COLUMN "private"."template_soa_mappings"."map_soa_bk" IS 'The business key for the SoA mapping record.';



COMMENT ON COLUMN "private"."template_soa_mappings"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_soa_mappings"."parent_scenario_configuration_bk" IS 'Links this SoA mapping to a specific plan.';



CREATE TABLE IF NOT EXISTS "private"."template_studies" (
    "study_bk" "text" NOT NULL,
    "protocol_number" "text" NOT NULL,
    "study_title" "text" NOT NULL,
    "study_short_name" "text",
    "therapeutic_area" "text",
    "phase" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    CONSTRAINT "chk_phase_enum" CHECK (("phase" = ANY (ARRAY['Phase 1'::"text", 'Phase 2'::"text", 'Phase 3'::"text", 'Phase 4'::"text", 'Observational'::"text", 'Other'::"text"])))
);


ALTER TABLE "private"."template_studies" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_studies" IS 'The master list of template clinical trial protocols. Why: Serves as the top-level container for all study-related planning, to which scenarios and amendments are linked.';



COMMENT ON COLUMN "private"."template_studies"."study_bk" IS 'The business key for the study.';



COMMENT ON COLUMN "private"."template_studies"."protocol_number" IS 'The unique, sponsor-provided identifier for the clinical trial protocol.';



COMMENT ON COLUMN "private"."template_studies"."study_title" IS 'The full, formal title of the clinical trial.';



COMMENT ON COLUMN "private"."template_studies"."study_short_name" IS 'A short, user-friendly name or alias for the study.';



COMMENT ON COLUMN "private"."template_studies"."therapeutic_area" IS 'The medical field the study belongs to (e.g., "Oncology", "Cardiology").';



COMMENT ON COLUMN "private"."template_studies"."phase" IS 'The clinical trial phase (e.g., ''Phase 1'', ''Phase 2'').';



COMMENT ON COLUMN "private"."template_studies"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON CONSTRAINT "chk_phase_enum" ON "private"."template_studies" IS 'CHECK constraint to enforce that the phase value is a valid member of the study_phase_enum.';



CREATE TABLE IF NOT EXISTS "private"."template_study_arms" (
    "arm_bk" "text" NOT NULL,
    "arm_name" "text" NOT NULL,
    "target_enrollment" integer,
    "description" "text",
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_scenario_configuration_bk" "text" NOT NULL
);


ALTER TABLE "private"."template_study_arms" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_study_arms" IS 'Defines the treatment, control, or other arms for a template scenario. Why: Partitions the study schedule and enrollment targets, enabling weighted forecasting that reflects real-world trial designs.';



COMMENT ON COLUMN "private"."template_study_arms"."arm_bk" IS 'The business key for the study arm.';



COMMENT ON COLUMN "private"."template_study_arms"."arm_name" IS 'The human-readable name of the arm (e.g., "Treatment Arm A", "Control Arm").';



COMMENT ON COLUMN "private"."template_study_arms"."target_enrollment" IS 'The number of subjects planned to be enrolled in this specific arm.';



COMMENT ON COLUMN "private"."template_study_arms"."description" IS 'A detailed description of the treatment or protocol for this arm.';



COMMENT ON COLUMN "private"."template_study_arms"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_study_arms"."parent_scenario_configuration_bk" IS 'Links this study arm to a specific plan.';



CREATE TABLE IF NOT EXISTS "private"."template_study_epochs" (
    "epoch_bk" "text" NOT NULL,
    "parent_arm_bk" "text" NOT NULL,
    "epoch_name" "text" NOT NULL,
    "epoch_order" integer NOT NULL,
    "description" "text",
    "include_in_json" boolean DEFAULT true NOT NULL
);


ALTER TABLE "private"."template_study_epochs" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_study_epochs" IS 'Defines major time periods within a study arm (e.g., Screening, Treatment, Follow-up). Why: Groups visits into logical phases, providing structure to the visit schedule.';



COMMENT ON COLUMN "private"."template_study_epochs"."epoch_bk" IS 'The business key for the study epoch.';



COMMENT ON COLUMN "private"."template_study_epochs"."parent_arm_bk" IS 'Links this epoch to its parent study arm.';



COMMENT ON COLUMN "private"."template_study_epochs"."epoch_name" IS 'The name of the epoch (e.g., "Screening", "Treatment Phase 1").';



COMMENT ON COLUMN "private"."template_study_epochs"."epoch_order" IS 'The sequential order of this epoch within its parent arm.';



COMMENT ON COLUMN "private"."template_study_epochs"."description" IS 'A detailed description of the purpose of this epoch.';



COMMENT ON COLUMN "private"."template_study_epochs"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



CREATE TABLE IF NOT EXISTS "private"."template_study_partners" (
    "partner_organization_bk" "text" NOT NULL,
    "partner_role" "text" NOT NULL,
    "is_primary" boolean DEFAULT true NOT NULL,
    "include_in_json" boolean DEFAULT true NOT NULL,
    "parent_scenario_configuration_bk" "text" NOT NULL,
    "study_partner_bk" "text" NOT NULL
);


ALTER TABLE "private"."template_study_partners" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_study_partners" IS 'Assigns roles (e.g., Sponsor, CRO, Lab) to specific organizations for a given template plan. Why: Defines the stakeholders and their financial and operational responsibilities within the trial.';



COMMENT ON COLUMN "private"."template_study_partners"."partner_organization_bk" IS 'Links this partner assignment to a specific organization from the template organization directory.';



COMMENT ON COLUMN "private"."template_study_partners"."partner_role" IS 'The role this organization plays in the study (e.g., ''Sponsor'', ''CRO'', ''Lab'').';



COMMENT ON COLUMN "private"."template_study_partners"."is_primary" IS 'Flag indicating if this is the primary partner for a given role (e.g., the primary Sponsor).';



COMMENT ON COLUMN "private"."template_study_partners"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_study_partners"."parent_scenario_configuration_bk" IS 'Links this partner assignment to a specific plan.';



COMMENT ON COLUMN "private"."template_study_partners"."study_partner_bk" IS 'The business key for the study partner assignment record.';



CREATE TABLE IF NOT EXISTS "private"."template_study_staff" (
    "staff_user_bk" "text" NOT NULL,
    "staff_role" "text" NOT NULL,
    "is_primary_contact" boolean DEFAULT false,
    "include_in_json" boolean DEFAULT true NOT NULL,
    "study_staff_bk" "text" NOT NULL,
    "parent_study_partner_bk" "text"
);


ALTER TABLE "private"."template_study_staff" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_study_staff" IS 'Assigns template users to template study partners with a specific role. Why: Models the specific personnel (e.g., Principal Investigator, Study Coordinator) involved in the trial.';



COMMENT ON COLUMN "private"."template_study_staff"."staff_user_bk" IS 'Links this staff assignment to a specific user from the template user directory.';



COMMENT ON COLUMN "private"."template_study_staff"."staff_role" IS 'The role of the user in this context (e.g., "Principal Investigator").';



COMMENT ON COLUMN "private"."template_study_staff"."is_primary_contact" IS 'Flag indicating if this user is the primary point of contact for their assigned partner.';



COMMENT ON COLUMN "private"."template_study_staff"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



COMMENT ON COLUMN "private"."template_study_staff"."study_staff_bk" IS 'The business key for the study staff assignment record.';



COMMENT ON COLUMN "private"."template_study_staff"."parent_study_partner_bk" IS 'Links this staff assignment to a specific study partner record.';



CREATE TABLE IF NOT EXISTS "private"."template_study_visits" (
    "visit_bk" "text" NOT NULL,
    "parent_epoch_bk" "text" NOT NULL,
    "visit_name" "text" NOT NULL,
    "visit_order" integer NOT NULL,
    "offset_days" integer NOT NULL,
    "offset_window_days" integer,
    "description" "text",
    "include_in_json" boolean DEFAULT true NOT NULL
);


ALTER TABLE "private"."template_study_visits" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_study_visits" IS 'Defines the detailed visit schedule within an epoch for template data. Why: Provides the timeline backbone for the Schedule of Activities, specifying the timing of all patient interactions.';



COMMENT ON COLUMN "private"."template_study_visits"."visit_bk" IS 'The business key for the study visit.';



COMMENT ON COLUMN "private"."template_study_visits"."parent_epoch_bk" IS 'Links this visit to its parent epoch.';



COMMENT ON COLUMN "private"."template_study_visits"."visit_name" IS 'The name of the visit (e.g., "Week 4 Visit").';



COMMENT ON COLUMN "private"."template_study_visits"."visit_order" IS 'The sequential order of this visit within its parent epoch.';



COMMENT ON COLUMN "private"."template_study_visits"."offset_days" IS 'The number of days from the start of the epoch that this visit is scheduled to occur.';



COMMENT ON COLUMN "private"."template_study_visits"."offset_window_days" IS 'The number of days before or after the `offset_days` that the visit can still occur.';



COMMENT ON COLUMN "private"."template_study_visits"."description" IS 'A detailed description of the purpose and events of this visit.';



COMMENT ON COLUMN "private"."template_study_visits"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';



CREATE TABLE IF NOT EXISTS "private"."template_users" (
    "user_bk" "text" NOT NULL,
    "user_name" "text" NOT NULL,
    "user_email" "text" NOT NULL,
    "user_status" "text" DEFAULT 'Active'::"text",
    "include_in_json" boolean DEFAULT true NOT NULL
);


ALTER TABLE "private"."template_users" OWNER TO "postgres";


COMMENT ON TABLE "private"."template_users" IS 'A directory of all fictitious users for the template data. Why: Represents the personnel who can be assigned roles and responsibilities in a clinical trial.';



COMMENT ON COLUMN "private"."template_users"."user_bk" IS 'The business key for the user.';



COMMENT ON COLUMN "private"."template_users"."user_name" IS 'The full name of the user.';



COMMENT ON COLUMN "private"."template_users"."user_email" IS 'The email address of the user.';



COMMENT ON COLUMN "private"."template_users"."user_status" IS 'The status of the user account (e.g., ''Active'').';



COMMENT ON COLUMN "private"."template_users"."include_in_json" IS 'Internal flag for the template authoring system to control inclusion in the final payload.';




CREATE TABLE IF NOT EXISTS "public"."dim_activity" (
    "activity_sk" bigint NOT NULL,
    "activity_bk" "text" NOT NULL,
    "organization_sk" bigint NOT NULL,
    "activity_name" "text" NOT NULL,
    "activity_type" "public"."activity_type_enum" NOT NULL,
    "reimbursement_type_sk" bigint,
    "standard_code" "text",
    "description" "text",
    "is_billable" boolean DEFAULT true NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "activity_domain" "text",
    "activity_category" "text",
    "budget_category_sk" bigint NOT NULL,
    CONSTRAINT "chk_activity_domain" CHECK (("activity_domain" = ANY (ARRAY['Study & Protocol'::"text", 'Site Operations'::"text", 'Patient & Procedures'::"text", 'Sample & Lab'::"text", 'Data & Biostats'::"text", 'Regulatory & Quality'::"text", 'Technology & Vendors'::"text"])))
);


ALTER TABLE "public"."dim_activity" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_activity" IS 'The master catalog of all clinical trial activities (procedures, assessments, etc.). Why: Provides a standardized, reusable library for building study blueprints and serves as a primary dimension for financial and operational reporting. How: Linked to budget categories and reimbursement policies.';



COMMENT ON COLUMN "public"."dim_activity"."activity_sk" IS 'Surrogate key for the activity. Why: Provides a stable, internal identifier for performant joins.';



COMMENT ON COLUMN "public"."dim_activity"."activity_bk" IS 'Globally unique business key. Why: Enables idempotent seeding, cross-environment portability, and the "Adoption" workflow where a user takes ownership of a template record.';



COMMENT ON COLUMN "public"."dim_activity"."organization_sk" IS 'The RLS key linking the record to a specific tenant. Why: Enforces absolute data isolation.';



COMMENT ON COLUMN "public"."dim_activity"."activity_name" IS 'The human-readable name of the activity, unique within an organization.';



COMMENT ON COLUMN "public"."dim_activity"."activity_type" IS 'Classifies the operational nature of the activity (e.g., ''PROCEDURE'', ''ASSESSMENT'', ''LABORATORY''). This classification drives scheduling, costing, and reporting logic.';



COMMENT ON COLUMN "public"."dim_activity"."reimbursement_type_sk" IS 'Links to the default reimbursement policy for this activity, which determines the financial multiplier applied to its cost.';



COMMENT ON COLUMN "public"."dim_activity"."standard_code" IS 'An optional standard medical or procedural code (e.g., CPT, LOINC) for interoperability.';



COMMENT ON COLUMN "public"."dim_activity"."description" IS 'A detailed description of the activity and its purpose.';



COMMENT ON COLUMN "public"."dim_activity"."is_billable" IS 'Flag indicating if this activity typically incurs a cost. If false, it is treated as a zero-cost operational event, which is critical for the Operational Ledger Mandate.';



COMMENT ON COLUMN "public"."dim_activity"."is_deleted" IS 'Soft-delete flag, preserving historical data and referential integrity upon user deletion.';



COMMENT ON COLUMN "public"."dim_activity"."created_at" IS 'Standard audit trail column: timestamp of record creation.';



COMMENT ON COLUMN "public"."dim_activity"."updated_at" IS 'Standard audit trail column: timestamp of last modification, updated automatically by a trigger.';



COMMENT ON COLUMN "public"."dim_activity"."created_by_user_sk" IS 'Standard audit trail column: links to the user who created the record.';



COMMENT ON COLUMN "public"."dim_activity"."updated_by_user_sk" IS 'Standard audit trail column: links to the user who last updated the record.';



COMMENT ON COLUMN "public"."dim_activity"."activity_domain" IS 'A high-level functional grouping for the activity (e.g., "Site Operations", "Patient & Procedures"). Used for BI rollups.';



COMMENT ON COLUMN "public"."dim_activity"."activity_category" IS 'A more granular, user-defined category for the activity.';



COMMENT ON COLUMN "public"."dim_activity"."budget_category_sk" IS 'Links this operational activity to a financial account in the budget hierarchy for cost rollups and reporting.';



COMMENT ON CONSTRAINT "chk_activity_domain" ON "public"."dim_activity" IS 'CHECK constraint to enforce that the activity_domain is one of the predefined valid domains, ensuring a consistent taxonomy.';



ALTER TABLE "public"."dim_activity" ALTER COLUMN "activity_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_activity_activity_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_activity_cost" (
    "activity_cost_sk" bigint NOT NULL,
    "activity_sk" bigint NOT NULL,
    "organization_sk" bigint NOT NULL,
    "cost_unit" "public"."cost_unit_enum" NOT NULL,
    "unit_cost" numeric(15,2) NOT NULL,
    "currency_code" "text" DEFAULT 'USD'::character varying NOT NULL,
    "effective_date" "date" DEFAULT CURRENT_DATE NOT NULL,
    "end_date" "date",
    "notes" "text",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "reimbursement_type_sk" bigint,
    "activity_cost_bk" "text" NOT NULL,
    "scenario_configuration_sk" bigint NOT NULL,
    "cost_bearing_partner_sk" bigint,
    "payer_partner_sk" bigint,
    CONSTRAINT "chk_activity_cost_dates" CHECK ((("end_date" IS NULL) OR ("end_date" > "effective_date"))),
    CONSTRAINT "dim_activity_cost_unit_cost_check" CHECK (("unit_cost" >= (0)::numeric))
);


ALTER TABLE "public"."dim_activity_cost" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_activity_cost" IS 'Defines the unit costs for activities within a specific scenario configuration. Why: Acts as the financial source of truth for pricing activities in a plan, enabling versioned pricing and what-if analysis. How: Scoped by configuration, activity, cost unit, and an effective date window.';



COMMENT ON COLUMN "public"."dim_activity_cost"."activity_cost_sk" IS 'Surrogate key for the activity cost record.';



COMMENT ON COLUMN "public"."dim_activity_cost"."activity_sk" IS 'Links this cost record to the specific activity being priced.';



COMMENT ON COLUMN "public"."dim_activity_cost"."organization_sk" IS 'The RLS key for tenant isolation, inherited from the parent scenario configuration via a trigger.';



COMMENT ON COLUMN "public"."dim_activity_cost"."cost_unit" IS 'Specifies the unit of measure for the `unit_cost` (e.g., ''Per Visit'', ''Per Subject''). This is a critical component of the pricing model, constrained by a CHECK list.';



COMMENT ON COLUMN "public"."dim_activity_cost"."unit_cost" IS 'The cost for a single unit of the specified `cost_unit`.';



COMMENT ON COLUMN "public"."dim_activity_cost"."currency_code" IS 'The ISO currency code for the unit cost (e.g., ''USD'').';



COMMENT ON COLUMN "public"."dim_activity_cost"."effective_date" IS 'The date from which this cost becomes effective, allowing for time-bounded pricing.';



COMMENT ON COLUMN "public"."dim_activity_cost"."end_date" IS 'The optional date after which this cost is no longer effective.';



COMMENT ON COLUMN "public"."dim_activity_cost"."notes" IS 'Optional notes providing context for the pricing.';



COMMENT ON COLUMN "public"."dim_activity_cost"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_activity_cost"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_activity_cost"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_activity_cost"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_activity_cost"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_activity_cost"."reimbursement_type_sk" IS 'Links to a specific reimbursement policy that applies a financial multiplier to this cost.';



COMMENT ON COLUMN "public"."dim_activity_cost"."activity_cost_bk" IS 'Globally unique business key for the cost record, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."dim_activity_cost"."scenario_configuration_sk" IS 'Links this cost record to a specific plan (a study-scenario combination), ensuring that pricing is versioned and scoped correctly for what-if analysis.';



COMMENT ON COLUMN "public"."dim_activity_cost"."cost_bearing_partner_sk" IS 'Identifies the study partner (the payee) who will receive payment for this event.';



COMMENT ON COLUMN "public"."dim_activity_cost"."payer_partner_sk" IS 'Identifies the study partner (the payer, typically the Sponsor) responsible for paying for this event.';



COMMENT ON CONSTRAINT "chk_activity_cost_dates" ON "public"."dim_activity_cost" IS 'CHECK constraint to ensure that if an end_date is specified, it must be after the effective_date.';



COMMENT ON CONSTRAINT "dim_activity_cost_unit_cost_check" ON "public"."dim_activity_cost" IS 'CHECK constraint to ensure that the unit_cost is not negative.';



ALTER TABLE "public"."dim_activity_cost" ALTER COLUMN "activity_cost_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_activity_cost_activity_cost_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_amendment" (
    "amendment_sk" bigint NOT NULL,
    "study_sk" bigint NOT NULL,
    "amendment_version_id" "text" NOT NULL,
    "approval_date" "date",
    "effective_date" "date" NOT NULL,
    "summary" "text" NOT NULL,
    "amendment_status" "public"."amendment_status_enum" DEFAULT 'Draft'::"public"."amendment_status_enum" NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "organization_sk" bigint NOT NULL,
    "amendment_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    CONSTRAINT "chk_amendment_dates" CHECK ((("approval_date" IS NULL) OR ("effective_date" >= "approval_date")))
);


ALTER TABLE "public"."dim_amendment" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_amendment" IS 'Stores protocol amendments for a study. Why: Models how a study plan changes over time, often triggering the creation of new "What-If" scenarios to analyze the financial and operational impact.';



COMMENT ON COLUMN "public"."dim_amendment"."amendment_sk" IS 'Surrogate key for the amendment.';



COMMENT ON COLUMN "public"."dim_amendment"."study_sk" IS 'Links this amendment to the parent study it modifies, providing a clear historical record of protocol changes.';



COMMENT ON COLUMN "public"."dim_amendment"."amendment_version_id" IS 'The unique, human-readable identifier for the amendment (e.g., "v2.0"), unique per study.';



COMMENT ON COLUMN "public"."dim_amendment"."approval_date" IS 'The date the amendment was officially approved.';



COMMENT ON COLUMN "public"."dim_amendment"."effective_date" IS 'The date the amendment takes effect, driving the timing of associated plan changes.';



COMMENT ON COLUMN "public"."dim_amendment"."summary" IS 'A summary of the changes introduced by this amendment.';



COMMENT ON COLUMN "public"."dim_amendment"."amendment_status" IS 'Tracks the lifecycle stage of the amendment (''Draft'', ''In Review'', ''Approved'', ''Implemented''), which controls its visibility and impact on scenario generation.';



COMMENT ON COLUMN "public"."dim_amendment"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_amendment"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_amendment"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_amendment"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_amendment"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_amendment"."organization_sk" IS 'The RLS key for tenant isolation, inherited from the parent study.';



COMMENT ON COLUMN "public"."dim_amendment"."amendment_bk" IS 'Globally unique business key for the amendment, supporting the "Adoption" workflow.';



COMMENT ON CONSTRAINT "chk_amendment_dates" ON "public"."dim_amendment" IS 'CHECK constraint to ensure that if an approval_date is specified, it must be on or before the effective_date.';



ALTER TABLE "public"."dim_amendment" ALTER COLUMN "amendment_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_amendment_amendment_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_budget_category" (
    "budget_category_sk" bigint NOT NULL,
    "category_name" "text" NOT NULL,
    "parent_category_sk" bigint,
    "is_rollup_target" boolean DEFAULT false NOT NULL,
    "description" "text",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "organization_sk" bigint NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "budget_category_bk" "text" NOT NULL,
    "parent_category_bk" "text"
);


ALTER TABLE "public"."dim_budget_category" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_budget_category" IS 'The hierarchical chart of accounts. Why: Provides the foundational structure for financial rollups and reporting, acting as the financial taxonomy for all costs. How: Linked hierarchically via `parent_category_sk`.';



COMMENT ON COLUMN "public"."dim_budget_category"."budget_category_sk" IS 'Surrogate key for the budget category.';



COMMENT ON COLUMN "public"."dim_budget_category"."category_name" IS 'The human-readable name of the budget category (e.g., "Site Pass-Through Costs").';



COMMENT ON COLUMN "public"."dim_budget_category"."parent_category_sk" IS 'Links this category to its parent in the budget hierarchy, enabling financial rollups. A NULL value indicates a top-level category.';



COMMENT ON COLUMN "public"."dim_budget_category"."is_rollup_target" IS 'A flag indicating that this category is a significant endpoint for financial summary reports.';



COMMENT ON COLUMN "public"."dim_budget_category"."description" IS 'A detailed description of the types of costs included in this category.';



COMMENT ON COLUMN "public"."dim_budget_category"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_budget_category"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."dim_budget_category"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_category"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_category"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_category"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_category"."budget_category_bk" IS 'Globally unique business key for the budget category, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."dim_budget_category"."parent_category_bk" IS 'Denormalized business key of the parent category. Why: Simplifies hierarchical creation and reporting without requiring extra joins.';



ALTER TABLE "public"."dim_budget_category" ALTER COLUMN "budget_category_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_budget_category_budget_category_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_budget_scenario" (
    "scenario_sk" bigint NOT NULL,
    "scenario_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    "scenario_name" "text" NOT NULL,
    "scenario_type" "public"."budget_scenario_type_enum" NOT NULL,
    "version" integer DEFAULT 1 NOT NULL,
    "triggering_amendment_sk" bigint,
    "scenario_status" "public"."budget_scenario_status_enum" DEFAULT 'Draft'::"public"."budget_scenario_status_enum" NOT NULL,
    "is_locked" boolean DEFAULT false NOT NULL,
    "description" "text",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "organization_sk" bigint NOT NULL,
    CONSTRAINT "dim_budget_scenario_version_check" CHECK (("version" > 0))
);


ALTER TABLE "public"."dim_budget_scenario" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_budget_scenario" IS 'The catalog of all planning scenarios. Why: Enables the comparison of different plan types (e.g., Forecast vs. Budget vs. Baseline) for a study. How: Its status governs mutability, and it can be locked via an RPC to finalize a plan.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."scenario_sk" IS 'Surrogate key for the scenario.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."scenario_bk" IS 'Globally unique business key for the scenario, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."scenario_name" IS 'The human-readable name of the scenario (e.g., "2024 Approved Budget").';



COMMENT ON COLUMN "public"."dim_budget_scenario"."scenario_type" IS 'Classifies the business purpose of the scenario (e.g., ''Baseline'' for the original plan, ''Forecast'' for a rolling projection, ''What-If'' for modeling changes). This is a primary dimension for BI and variance analysis.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."version" IS 'The version number of the scenario, typically incremented for new iterations of the same type.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."triggering_amendment_sk" IS 'An optional link to an amendment that caused this scenario to be created, providing clear lineage for plan changes.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."scenario_status" IS 'The workflow status of the scenario (e.g., ''Draft'', ''Approved''). Controls business logic like locking.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."is_locked" IS 'A flag indicating that the scenario and its entire blueprint are immutable. Why: Protects finalized plans (e.g., an approved baseline) from accidental changes.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."description" IS 'A detailed description of the assumptions and purpose of this scenario.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_budget_scenario"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON CONSTRAINT "dim_budget_scenario_version_check" ON "public"."dim_budget_scenario" IS 'CHECK constraint to ensure the scenario version number is a positive integer.';



ALTER TABLE "public"."dim_budget_scenario" ALTER COLUMN "scenario_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_budget_scenario_scenario_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_date" (
    "date_sk" "date" NOT NULL,
    "full_date" timestamp with time zone NOT NULL,
    "year" integer NOT NULL,
    "quarter" integer NOT NULL,
    "month" integer NOT NULL,
    "month_name" "text" NOT NULL,
    "week_of_year" integer NOT NULL,
    "day_of_week" integer NOT NULL,
    "day_name" "text" NOT NULL,
    "day_of_year" integer NOT NULL,
    "is_weekend" boolean NOT NULL,
    "is_holiday" boolean DEFAULT false NOT NULL
);


ALTER TABLE "public"."dim_date" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_date" IS 'A standard date dimension table. Why: Provides a conformed dimension for all time-series analysis, enabling consistent grouping and filtering by year, quarter, month, etc. How: Populated by the `populate_dim_date` utility function.';



COMMENT ON COLUMN "public"."dim_date"."date_sk" IS 'The surrogate key for the date dimension, represented as a DATE. This is the primary key used for joining with fact tables.';



COMMENT ON COLUMN "public"."dim_date"."full_date" IS 'The full date and time, preserving timezone information.';



COMMENT ON COLUMN "public"."dim_date"."year" IS 'The calendar year of the date.';



COMMENT ON COLUMN "public"."dim_date"."quarter" IS 'The calendar quarter (1-4) of the date.';



COMMENT ON COLUMN "public"."dim_date"."month" IS 'The month number (1-12) of the date.';



COMMENT ON COLUMN "public"."dim_date"."month_name" IS 'The full name of the month (e.g., "January").';



COMMENT ON COLUMN "public"."dim_date"."week_of_year" IS 'The ISO week number of the year.';



COMMENT ON COLUMN "public"."dim_date"."day_of_week" IS 'The day of the week (1 for Monday, 7 for Sunday).';



COMMENT ON COLUMN "public"."dim_date"."day_name" IS 'The full name of the day (e.g., "Monday").';



COMMENT ON COLUMN "public"."dim_date"."day_of_year" IS 'The day number within the year (1-366).';



COMMENT ON COLUMN "public"."dim_date"."is_weekend" IS 'A boolean flag indicating if the day is a weekend.';



COMMENT ON COLUMN "public"."dim_date"."is_holiday" IS 'A boolean flag indicating if the day is a recognized holiday (functionality can be extended).';



CREATE TABLE IF NOT EXISTS "public"."dim_forecast_calculation_config" (
    "forecast_config_sk" bigint NOT NULL,
    "config_name" "text" NOT NULL,
    "activity_sk" bigint,
    "calculation_method" "public"."calculation_method_enum" NOT NULL,
    "cost_unit" "public"."cost_unit_enum",
    "notes" "text",
    "is_active" boolean DEFAULT true NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "organization_sk" bigint NOT NULL,
    "curve_points" "jsonb",
    "enrollment_curve_type" "text",
    "is_group_definition" boolean DEFAULT false NOT NULL,
    "forecast_config_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    "scenario_configuration_sk" bigint NOT NULL,
    CONSTRAINT "chk_dfcc_method_activity_scope" CHECK ((("calculation_method" <> ALL (ARRAY['DIRECT_MONTHLY'::"public"."calculation_method_enum", 'ENROLLMENT_BASED'::"public"."calculation_method_enum"])) OR ("activity_sk" IS NOT NULL)))
);


ALTER TABLE "public"."dim_forecast_calculation_config" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_forecast_calculation_config" IS 'Defines the rules that drive the forecast "compiler". Why: Controls the calculation method (schedule-based, enrollment-based, etc.) and scoping for generating costs. How: Evaluated by the `recalculate_forecast_details` engine.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."forecast_config_sk" IS 'Surrogate key for the forecast rule.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."config_name" IS 'The human-readable name of the rule.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."activity_sk" IS 'An optional link to a specific activity, scoping the rule''s application.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."calculation_method" IS 'The core logic for the forecast engine: ''DIRECT_MONTHLY'', ''ENROLLMENT_BASED'', ''SCHEDULE_BASED'', or ''ENROLLMENT_CURVE_BASED''.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."cost_unit" IS 'An optional filter to apply this rule only to costs with a matching cost unit.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."notes" IS 'Optional notes providing context for the rule.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."is_active" IS 'A flag to enable or disable the rule without deleting it.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."curve_points" IS 'A JSONB array defining the percentage of total enrollment to be achieved each month for curve-based forecasting. Must sum to 100.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."enrollment_curve_type" IS 'A descriptive name for the enrollment curve (e.g., "Standard S-Curve").';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."is_group_definition" IS 'Flag indicating if this rule defines a performance group (e.g., an enrollment curve) that sites can be assigned to.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."forecast_config_bk" IS 'Globally unique business key for the rule, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."dim_forecast_calculation_config"."scenario_configuration_sk" IS 'Links this rule to a specific plan.';



ALTER TABLE "public"."dim_forecast_calculation_config" ALTER COLUMN "forecast_config_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_forecast_calculation_config_forecast_config_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);


CREATE TABLE IF NOT EXISTS "public"."dim_organization" (
    "organization_sk" bigint NOT NULL,
    "organization_bk" "text" DEFAULT "gen_random_uuid"() NOT NULL,
    "organization_name" "text" NOT NULL,
    "organization_type" "public"."organization_type_enum" NOT NULL,
    "country_code" "text",
    "region" "text",
    "org_status" "public"."org_status_enum" DEFAULT 'Active'::"public"."org_status_enum" NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "parent_organization_sk" bigint,
    "is_clerk_managed" boolean DEFAULT false NOT NULL,
    CONSTRAINT "organization_name_not_empty" CHECK ((TRIM(BOTH FROM "organization_name") <> ''::"text"))
);


ALTER TABLE "public"."dim_organization" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_organization" IS 'The directory of all organizations, both real (Clerk-managed) and fictitious (sandbox-specific). Why: Anchors the multi-tenant hierarchy and defines the ecosystem of partners (Sponsors, CROs, Sites, Vendors). How: Access is governed by RLS policies using the `has_organization_hierarchy_access_by_sk` helper.';



COMMENT ON COLUMN "public"."dim_organization"."organization_sk" IS 'Surrogate key for the organization.';



COMMENT ON COLUMN "public"."dim_organization"."organization_bk" IS 'The business key, which maps to the Clerk organization ID for real organizations or a generated ID for fictitious ones.';



COMMENT ON COLUMN "public"."dim_organization"."organization_name" IS 'The human-readable name of the organization.';



COMMENT ON COLUMN "public"."dim_organization"."organization_type" IS 'Classifies the primary business function of the organization (e.g., ''Sponsor'', ''Site'', ''CRO'').';



COMMENT ON COLUMN "public"."dim_organization"."country_code" IS 'The ISO 3166-1 alpha-2 country code for the organization.';



COMMENT ON COLUMN "public"."dim_organization"."region" IS 'The state, province, or region where the organization is located.';



COMMENT ON COLUMN "public"."dim_organization"."org_status" IS 'The operational status of the organization (Active, Inactive).';



COMMENT ON COLUMN "public"."dim_organization"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_organization"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_organization"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_organization"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_organization"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_organization"."parent_organization_sk" IS 'Links a fictitious organization to its parent (the root Clerk-managed org), forming the sandbox hierarchy.';



COMMENT ON COLUMN "public"."dim_organization"."is_clerk_managed" IS 'A critical flag that distinguishes real, root organizations synced from Clerk from the fictitious, user-created organizations within a sandbox.';



COMMENT ON CONSTRAINT "organization_name_not_empty" ON "public"."dim_organization" IS 'CHECK constraint to ensure the organization name is not an empty string.';



ALTER TABLE "public"."dim_organization" ALTER COLUMN "organization_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_organization_organization_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_reimbursement_type" (
    "reimbursement_type_sk" bigint NOT NULL,
    "reimbursement_type_name" "text" NOT NULL,
    "description" "text",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "organization_sk" bigint NOT NULL,
    "reimbursement_type_code" "text" NOT NULL,
    "financial_impact" numeric(10,2) DEFAULT 1.00 NOT NULL,
    "is_default" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "reimbursement_type_bk" "text" NOT NULL
);


ALTER TABLE "public"."dim_reimbursement_type" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_reimbursement_type" IS 'Defines financial reimbursement policies. Why: Models financial scenarios like discounts, holdbacks, or pass-through costs by applying a multiplier to activity costs. How: Referenced by `dim_activity_cost`.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."reimbursement_type_sk" IS 'Surrogate key for the reimbursement type.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."reimbursement_type_name" IS 'The human-readable name of the policy (e.g., "Standard Reimbursement").';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."description" IS 'A detailed description of the policy.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."reimbursement_type_code" IS 'A short, unique code for the reimbursement type (e.g., "STD_REIMB").';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."financial_impact" IS 'A numeric multiplier applied to a cost to calculate the net financial impact (e.g., 1.0 for full cost, 0.8 for a 20% discount).';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."is_default" IS 'Flag indicating if this is the default reimbursement policy for the organization. A trigger enforces that only one can be default.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_reimbursement_type"."reimbursement_type_bk" IS 'Globally unique business key for the reimbursement type, supporting the "Adoption" workflow.';



ALTER TABLE "public"."dim_reimbursement_type" ALTER COLUMN "reimbursement_type_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_reimbursement_type_reimbursement_type_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_site" (
    "site_sk" bigint NOT NULL,
    "site_bk" "text" NOT NULL,
    "scenario_configuration_sk" bigint NOT NULL,
    "site_status" "public"."site_status_enum" DEFAULT 'Planned'::"public"."site_status_enum",
    "activation_date" "date",
    "closeout_date" "date",
    "target_enrollment" integer,
    "organization_sk" bigint NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "site_partner_sk" bigint NOT NULL,
    "site_organization_sk" bigint NOT NULL,
    "performance_group_bk" "text",
    CONSTRAINT "chk_scenario_site_dates" CHECK ((("closeout_date" IS NULL) OR ("activation_date" IS NULL) OR ("closeout_date" > "activation_date")))
);


ALTER TABLE "public"."dim_site" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_site" IS 'Represents the assignment of a clinical site to a specific scenario configuration. Why: Drives enrollment projections and site-specific costs. How: Linked to a study partner and a configuration, with lifecycle dates and enrollment targets.';



COMMENT ON COLUMN "public"."dim_site"."site_sk" IS 'Surrogate key for the site assignment.';



COMMENT ON COLUMN "public"."dim_site"."site_bk" IS 'Globally unique business key for the site assignment, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."dim_site"."scenario_configuration_sk" IS 'Links this site assignment to a specific plan.';



COMMENT ON COLUMN "public"."dim_site"."site_status" IS 'Tracks the operational status of the site within this plan (e.g., ''Planned'', ''Initiating'', ''Active'', ''Enrolling''). This directly impacts enrollment projections.';



COMMENT ON COLUMN "public"."dim_site"."activation_date" IS 'The planned date for site activation.';



COMMENT ON COLUMN "public"."dim_site"."closeout_date" IS 'The planned date for site closeout.';



COMMENT ON COLUMN "public"."dim_site"."target_enrollment" IS 'The number of subjects this site is expected to enroll for this plan.';



COMMENT ON COLUMN "public"."dim_site"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."dim_site"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_site"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_site"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_site"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_site"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_site"."site_partner_sk" IS 'Links this site assignment to the study partner record that represents the site organization.';



COMMENT ON COLUMN "public"."dim_site"."site_organization_sk" IS 'Denormalized surrogate key linking to the organization record for the site. Why: Simplifies joins and improves query performance for site-related data. How: Maintained by the `trg_fn_sync_site_organization_from_partner` trigger.';



COMMENT ON COLUMN "public"."dim_site"."performance_group_bk" IS 'Links this site to an enrollment curve definition, allowing for modeling of different enrollment speeds (e.g., "High-Enrolling", "Standard").';



COMMENT ON CONSTRAINT "chk_scenario_site_dates" ON "public"."dim_site" IS 'CHECK constraint to ensure that if a closeout_date is specified, it must be after the activation_date.';



ALTER TABLE "public"."dim_site" ALTER COLUMN "site_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_site_site_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_study" (
    "study_sk" bigint NOT NULL,
    "study_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    "protocol_number" "text" NOT NULL,
    "study_title" "text" NOT NULL,
    "study_short_name" "text",
    "therapeutic_area" "text",
    "phase" "public"."study_phase_enum",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "organization_sk" bigint NOT NULL
);


ALTER TABLE "public"."dim_study" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_study" IS 'The master list of clinical trial protocols. Why: Serves as the top-level container for all study-related planning, to which scenarios and amendments are linked. This is a core "Blueprint" object.';



COMMENT ON COLUMN "public"."dim_study"."study_sk" IS 'Surrogate key for the study.';



COMMENT ON COLUMN "public"."dim_study"."study_bk" IS 'Globally unique business key for the study, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."dim_study"."protocol_number" IS 'The unique, sponsor-provided identifier for the clinical trial protocol. Acts as a natural key for human-facing reports and integrations, and is the primary business identifier for a study.';



COMMENT ON COLUMN "public"."dim_study"."study_title" IS 'The full, formal title of the clinical trial.';



COMMENT ON COLUMN "public"."dim_study"."study_short_name" IS 'A short, user-friendly name or alias for the study.';



COMMENT ON COLUMN "public"."dim_study"."therapeutic_area" IS 'The medical field the study belongs to (e.g., "Oncology", "Cardiology").';



COMMENT ON COLUMN "public"."dim_study"."phase" IS 'The clinical trial phase (e.g., ''Phase 1'', ''Phase 2'') which is a primary dimension for portfolio analysis and operational planning.';



COMMENT ON COLUMN "public"."dim_study"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_study"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study"."organization_sk" IS 'The RLS key for tenant isolation.';



CREATE TABLE IF NOT EXISTS "public"."dim_study_arm" (
    "arm_sk" bigint NOT NULL,
    "arm_name" "text" NOT NULL,
    "target_enrollment" bigint,
    "description" "text",
    "organization_sk" bigint NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "arm_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    "scenario_configuration_sk" bigint NOT NULL
);


ALTER TABLE "public"."dim_study_arm" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_study_arm" IS 'Defines the treatment, control, or other arms for a scenario. Why: Partitions the study schedule and enrollment targets, enabling weighted forecasting that reflects real-world trial designs. This is a core "Blueprint" object.';



COMMENT ON COLUMN "public"."dim_study_arm"."arm_sk" IS 'Surrogate key for the study arm.';



COMMENT ON COLUMN "public"."dim_study_arm"."arm_name" IS 'The human-readable name of the arm (e.g., "Treatment Arm A", "Control Arm").';



COMMENT ON COLUMN "public"."dim_study_arm"."target_enrollment" IS 'The number of subjects planned to be enrolled in this specific arm. This value is critical for the Weighted Distribution Mandate.';



COMMENT ON COLUMN "public"."dim_study_arm"."description" IS 'A detailed description of the treatment or protocol for this arm.';



COMMENT ON COLUMN "public"."dim_study_arm"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."dim_study_arm"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_study_arm"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_arm"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_arm"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_arm"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_arm"."arm_bk" IS 'Globally unique business key for the study arm, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."dim_study_arm"."scenario_configuration_sk" IS 'Links this study arm to a specific plan.';



CREATE SEQUENCE IF NOT EXISTS "public"."dim_study_arm_arm_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE "public"."dim_study_arm_arm_sk_seq" OWNER TO "postgres";


ALTER SEQUENCE "public"."dim_study_arm_arm_sk_seq" OWNED BY "public"."dim_study_arm"."arm_sk";



ALTER TABLE "public"."dim_study_arm" ALTER COLUMN "arm_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_study_arm_arm_sk_seq1"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_study_epochs" (
    "epoch_sk" bigint NOT NULL,
    "epoch_name" "text" NOT NULL,
    "epoch_order" integer NOT NULL,
    "description" "text",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "organization_sk" bigint NOT NULL,
    "arm_sk" bigint NOT NULL,
    "epoch_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL
);


ALTER TABLE "public"."dim_study_epochs" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_study_epochs" IS 'Defines major time periods within a study arm (e.g., Screening, Treatment, Follow-up). Why: Groups visits into logical phases, providing structure to the visit schedule. This is a core "Blueprint" object.';



COMMENT ON COLUMN "public"."dim_study_epochs"."epoch_sk" IS 'Surrogate key for the study epoch.';



COMMENT ON COLUMN "public"."dim_study_epochs"."epoch_name" IS 'The name of the epoch (e.g., "Screening", "Treatment Phase 1").';



COMMENT ON COLUMN "public"."dim_study_epochs"."epoch_order" IS 'The sequential order of this epoch within its parent arm.';



COMMENT ON COLUMN "public"."dim_study_epochs"."description" IS 'A detailed description of the purpose of this epoch.';



COMMENT ON COLUMN "public"."dim_study_epochs"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_study_epochs"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_epochs"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_epochs"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_epochs"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_epochs"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."dim_study_epochs"."arm_sk" IS 'Links this epoch to its parent study arm.';



COMMENT ON COLUMN "public"."dim_study_epochs"."epoch_bk" IS 'Globally unique business key for the study epoch, supporting the "Adoption" workflow.';



ALTER TABLE "public"."dim_study_epochs" ALTER COLUMN "epoch_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_study_epochs_epoch_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



ALTER TABLE "public"."dim_study" ALTER COLUMN "study_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_study_study_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_study_visits" (
    "visit_sk" bigint NOT NULL,
    "epoch_sk" bigint NOT NULL,
    "visit_name" "text" NOT NULL,
    "visit_order" integer NOT NULL,
    "offset_days" integer NOT NULL,
    "offset_window_days" integer,
    "description" "text",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "organization_sk" bigint NOT NULL,
    "visit_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    CONSTRAINT "dim_study_visits_offset_window_days_check" CHECK (("offset_window_days" >= 0))
);


ALTER TABLE "public"."dim_study_visits" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_study_visits" IS 'Defines the detailed visit schedule within an epoch. Why: Provides the timeline backbone for the Schedule of Activities, specifying the timing of all patient interactions. This is a core "Blueprint" object.';



COMMENT ON COLUMN "public"."dim_study_visits"."visit_sk" IS 'Surrogate key for the study visit.';



COMMENT ON COLUMN "public"."dim_study_visits"."epoch_sk" IS 'Links this visit to its parent epoch.';



COMMENT ON COLUMN "public"."dim_study_visits"."visit_name" IS 'The name of the visit (e.g., "Week 4 Visit").';



COMMENT ON COLUMN "public"."dim_study_visits"."visit_order" IS 'The sequential order of this visit within its parent epoch.';



COMMENT ON COLUMN "public"."dim_study_visits"."offset_days" IS 'The number of days from the start of the epoch that this visit is scheduled to occur.';



COMMENT ON COLUMN "public"."dim_study_visits"."offset_window_days" IS 'The number of days before or after the `offset_days` that the visit can still occur.';



COMMENT ON COLUMN "public"."dim_study_visits"."description" IS 'A detailed description of the purpose and events of this visit.';



COMMENT ON COLUMN "public"."dim_study_visits"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_study_visits"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_visits"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_visits"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_visits"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_study_visits"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."dim_study_visits"."visit_bk" IS 'Globally unique business key for the study visit, supporting the "Adoption" workflow.';



COMMENT ON CONSTRAINT "dim_study_visits_offset_window_days_check" ON "public"."dim_study_visits" IS 'CHECK constraint to ensure the offset window is not negative.';



ALTER TABLE "public"."dim_study_visits" ALTER COLUMN "visit_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_study_visits_visit_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."dim_user" (
    "user_sk" bigint NOT NULL,
    "user_bk" "text" DEFAULT "gen_random_uuid"() NOT NULL,
    "user_name" "text",
    "user_email" "text",
    "user_status" "public"."user_status_enum" DEFAULT 'Active'::"public"."user_status_enum" NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "is_clerk_managed" boolean DEFAULT false NOT NULL,
    "organization_sk" bigint,
    CONSTRAINT "user_sandbox_key_not_null" CHECK (((("is_clerk_managed" = true) AND ("organization_sk" IS NULL)) OR (("is_clerk_managed" = false) AND ("organization_sk" IS NOT NULL))))
);


ALTER TABLE "public"."dim_user" OWNER TO "postgres";


COMMENT ON TABLE "public"."dim_user" IS 'The directory of all users, both real (Clerk-managed) and fictitious (sandbox-specific). Why: Supports staffing assignments, ownership tracking, and approvals. How: Linked to organizations via the `user_organization_membership` table.';



COMMENT ON COLUMN "public"."dim_user"."user_sk" IS 'Surrogate key for the user.';



COMMENT ON COLUMN "public"."dim_user"."user_bk" IS 'The business key, which maps to the Clerk user ID for real users or a generated ID for fictitious ones.';



COMMENT ON COLUMN "public"."dim_user"."user_name" IS 'The full name of the user.';



COMMENT ON COLUMN "public"."dim_user"."user_email" IS 'The email address of the user.';



COMMENT ON COLUMN "public"."dim_user"."user_status" IS 'The status of the user account (e.g., ''Active'').';



COMMENT ON COLUMN "public"."dim_user"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."dim_user"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_user"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_user"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_user"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."dim_user"."is_clerk_managed" IS 'A critical flag that distinguishes real, root users synced from Clerk from the fictitious, user-created users within a sandbox.';



COMMENT ON COLUMN "public"."dim_user"."organization_sk" IS 'For fictitious users, links them to their home sandbox organization. This is NULL for Clerk-managed users.';



COMMENT ON CONSTRAINT "user_sandbox_key_not_null" ON "public"."dim_user" IS 'CHECK constraint to enforce the business rule that fictitious users must belong to a sandbox organization, while Clerk-managed users must not.';



ALTER TABLE "public"."dim_user" ALTER COLUMN "user_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."dim_user_user_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."fact_enrollment" (
    "enrollment_pk" bigint NOT NULL,
    "enrollment_bk" "text" NOT NULL,
    "organization_sk" bigint NOT NULL,
    "snapshot_date_sk" "date" NOT NULL,
    "projected_enrollment_count" bigint DEFAULT 0 NOT NULL,
    "actual_enrollment_count" bigint DEFAULT 0 NOT NULL,
    "site_sk" bigint NOT NULL,
    "site_bk" "text",
    "is_asserted_actual" boolean DEFAULT false NOT NULL,
    "last_edit_source" "text" DEFAULT 'SYSTEM_CALCULATION'::"text" NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "scenario_configuration_sk" bigint NOT NULL,
    "study_bk" "text",
    "scenario_bk" "text",
    "scenario_configuration_bk" "text" NOT NULL,
    "study_sk" bigint NOT NULL,
    "scenario_sk" bigint NOT NULL,
    CONSTRAINT "chk_last_edit_source" CHECK (("last_edit_source" = ANY (ARRAY['SYSTEM_CALCULATION'::"text", 'USER_EDIT'::"text"])))
);


ALTER TABLE "public"."fact_enrollment" OWNER TO "postgres";


COMMENT ON TABLE "public"."fact_enrollment" IS 'Stores time-phased enrollment data, both projected and actual. Why: The core driver for schedule-based forecasting and variance analysis. This is a "bytecode" table populated by the enrollment engine.';



COMMENT ON COLUMN "public"."fact_enrollment"."enrollment_pk" IS 'Surrogate key for the enrollment fact record.';



COMMENT ON COLUMN "public"."fact_enrollment"."enrollment_bk" IS 'Globally unique business key for the enrollment fact, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."fact_enrollment"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."fact_enrollment"."snapshot_date_sk" IS 'The date (typically first of the month) to which this enrollment count applies. Links to `dim_date`.';



COMMENT ON COLUMN "public"."fact_enrollment"."projected_enrollment_count" IS 'The number of subjects forecasted to be enrolled, as calculated by the enrollment engine.';



COMMENT ON COLUMN "public"."fact_enrollment"."actual_enrollment_count" IS 'The number of subjects confirmed to have been enrolled. This value is typically ingested or entered manually.';



COMMENT ON COLUMN "public"."fact_enrollment"."site_sk" IS 'Links this enrollment fact to the specific clinical site where the subjects were enrolled.';



COMMENT ON COLUMN "public"."fact_enrollment"."site_bk" IS 'Denormalized business key of the site for simplified reporting and data portability.';



COMMENT ON COLUMN "public"."fact_enrollment"."is_asserted_actual" IS 'A flag indicating that the `actual_enrollment_count` was provided by a user or an external data feed, overriding any system calculations.';



COMMENT ON COLUMN "public"."fact_enrollment"."last_edit_source" IS 'Tracks whether the record was generated by the system (''SYSTEM_CALCULATION'') or manually entered/overridden (''USER_EDIT''). This is critical for protecting user data from being overwritten by automated processes.';



COMMENT ON COLUMN "public"."fact_enrollment"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."fact_enrollment"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_enrollment"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_enrollment"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_enrollment"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_enrollment"."scenario_configuration_sk" IS 'Links this enrollment fact to the specific plan it belongs to.';



COMMENT ON COLUMN "public"."fact_enrollment"."study_bk" IS 'Denormalized business key of the study for simplified reporting.';



COMMENT ON COLUMN "public"."fact_enrollment"."scenario_bk" IS 'Denormalized business key of the scenario for simplified reporting.';



COMMENT ON COLUMN "public"."fact_enrollment"."scenario_configuration_bk" IS 'Denormalized business key of the scenario configuration for simplified reporting.';



COMMENT ON COLUMN "public"."fact_enrollment"."study_sk" IS 'Denormalized surrogate key of the study for performant joins.';



COMMENT ON COLUMN "public"."fact_enrollment"."scenario_sk" IS 'Denormalized surrogate key of the scenario for performant joins.';



COMMENT ON CONSTRAINT "chk_last_edit_source" ON "public"."fact_enrollment" IS 'CHECK constraint to enforce that the last_edit_source is either ''SYSTEM_CALCULATION'' or ''USER_EDIT''.';



ALTER TABLE "public"."fact_enrollment" ALTER COLUMN "enrollment_pk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."fact_enrollment_enrollment_pk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."fact_forecast_detail" (
    "forecast_detail_pk" bigint NOT NULL,
    "forecast_detail_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    "organization_sk" bigint NOT NULL,
    "snapshot_date_sk" "date" NOT NULL,
    "cost_unit" "text" NOT NULL,
    "unit_cost" numeric(15,2) NOT NULL,
    "forecast_units" numeric(18,2) DEFAULT 0.00 NOT NULL,
    "reimbursement_factor" numeric(10,4) DEFAULT 1.0000,
    "forecast_cost" numeric(18,2) GENERATED ALWAYS AS (("forecast_units" * "unit_cost")) STORED,
    "net_cost" numeric(18,2) GENERATED ALWAYS AS ((("forecast_units" * "unit_cost") * "reimbursement_factor")) STORED,
    "activity_sk" bigint,
    "activity_bk" "text",
    "site_sk" bigint,
    "site_bk" "text",
    "source_visit_sk" bigint,
    "source_visit_bk" "text",
    "source_epoch_sk" bigint,
    "source_epoch_bk" "text",
    "source_enrollment_pk" bigint,
    "source_enrollment_bk" "text",
    "source_forecast_config_sk" bigint,
    "source_forecast_config_bk" "text",
    "is_asserted_actual" boolean DEFAULT false NOT NULL,
    "last_edit_source" "text" DEFAULT 'SYSTEM_CALCULATION'::"text" NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "source_activity_cost_sk" bigint,
    "source_activity_cost_bk" "text",
    "scenario_configuration_sk" bigint NOT NULL,
    "study_bk" "text" NOT NULL,
    "scenario_bk" "text" NOT NULL,
    "scenario_configuration_bk" character varying,
    "study_sk" bigint NOT NULL,
    "scenario_sk" bigint NOT NULL,
    "payer_partner_sk" bigint,
    "payer_partner_bk" "text",
    "payee_partner_sk" bigint,
    "payee_partner_bk" "text",
    "source_arm_sk" bigint,
    "source_arm_bk" "text",
    "source_map_soa_sk" bigint,
    "source_map_soa_bk" "text",
    CONSTRAINT "chk_ffd_last_edit_source" CHECK (("last_edit_source" = ANY (ARRAY['SYSTEM_CALCULATION'::"text", 'USER_EDIT'::"text"]))),
    CONSTRAINT "fact_forecast_detail_forecast_units_check" CHECK (("forecast_units" >= (0)::numeric)),
    CONSTRAINT "fact_forecast_detail_unit_cost_check" CHECK (("unit_cost" >= (0)::numeric))
);


ALTER TABLE "public"."fact_forecast_detail" OWNER TO "postgres";


COMMENT ON TABLE "public"."fact_forecast_detail" IS 'The lowest-grain financial and operational ledger, representing the "compiled bytecode" of a study plan. Populated by the `recalculate_forecast_details` engine. All SKs are hydrated by the `trg_fn_hydrate_forecast_detail` trigger.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."forecast_detail_pk" IS 'Surrogate key for the forecast detail record.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."forecast_detail_bk" IS 'Globally unique business key for the forecast detail record, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."snapshot_date_sk" IS 'The date to which this financial or operational event is attributed.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."cost_unit" IS 'The unit of measure for the `unit_cost` (e.g., ''Per Visit'').';



COMMENT ON COLUMN "public"."fact_forecast_detail"."unit_cost" IS 'The base cost for a single unit. A value of 0.00 indicates a non-billable but operationally significant event.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."forecast_units" IS 'The quantity of units (e.g., number of subjects) forecasted for this event.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."reimbursement_factor" IS 'A multiplier applied to the gross cost to calculate the net cost.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."forecast_cost" IS 'The gross calculated cost for the event. Generated as (forecast_units * unit_cost).';



COMMENT ON COLUMN "public"."fact_forecast_detail"."net_cost" IS 'The final calculated cost for the event. Generated as (forecast_units * unit_cost * reimbursement_factor).';



COMMENT ON COLUMN "public"."fact_forecast_detail"."activity_sk" IS 'Links to the specific activity that this forecast record represents.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."activity_bk" IS 'Denormalized business key of the activity.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."site_sk" IS 'Links to the clinical site where this event occurred. NULL for non-site-specific events.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."site_bk" IS 'Denormalized business key of the site.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_visit_sk" IS 'Lineage: links to the originating visit for schedule-based events.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_visit_bk" IS 'Lineage: denormalized business key of the originating visit.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_epoch_sk" IS 'Lineage: links to the originating epoch.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_epoch_bk" IS 'Lineage: denormalized business key of the originating epoch.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_enrollment_pk" IS 'Lineage: links to the originating enrollment fact record.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_enrollment_bk" IS 'Lineage: denormalized business key of the originating enrollment fact.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_forecast_config_sk" IS 'Lineage: links to the forecast rule that generated this record.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_forecast_config_bk" IS 'Lineage: denormalized business key of the originating forecast rule.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."is_asserted_actual" IS 'A flag indicating that this record represents a historical actual, not a forecast.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."last_edit_source" IS 'Tracks whether the record was generated by the system (''SYSTEM_CALCULATION'') or manually entered/overridden (''USER_EDIT'').';



COMMENT ON COLUMN "public"."fact_forecast_detail"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_activity_cost_sk" IS 'Lineage: links to the specific activity cost record used for pricing.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_activity_cost_bk" IS 'Lineage: denormalized business key of the originating activity cost.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."scenario_configuration_sk" IS 'Links this fact to the specific plan it belongs to.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."study_bk" IS 'Denormalized business key of the study.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."scenario_bk" IS 'Denormalized business key of the scenario.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."scenario_configuration_bk" IS 'Denormalized business key of the scenario configuration.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."study_sk" IS 'Denormalized surrogate key of the study.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."scenario_sk" IS 'Denormalized surrogate key of the scenario.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."payer_partner_sk" IS 'Identifies the study partner (the payer) responsible for paying for this event.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."payer_partner_bk" IS 'Denormalized business key of the payer partner.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."payee_partner_sk" IS 'Identifies the study partner (the payee) who will receive payment for this event.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."payee_partner_bk" IS 'Denormalized business key of the payee partner.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_arm_sk" IS 'Lineage: links to the originating study arm.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_arm_bk" IS 'Lineage: denormalized business key of the originating study arm.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_map_soa_sk" IS 'Lineage: links to the originating SoA mapping record.';



COMMENT ON COLUMN "public"."fact_forecast_detail"."source_map_soa_bk" IS 'Lineage: denormalized business key of the originating SoA mapping.';



COMMENT ON CONSTRAINT "chk_ffd_last_edit_source" ON "public"."fact_forecast_detail" IS 'CHECK constraint to enforce that the last_edit_source is either ''SYSTEM_CALCULATION'' or ''USER_EDIT''.';



COMMENT ON CONSTRAINT "fact_forecast_detail_forecast_units_check" ON "public"."fact_forecast_detail" IS 'CHECK constraint to ensure that forecast units are not negative.';



COMMENT ON CONSTRAINT "fact_forecast_detail_unit_cost_check" ON "public"."fact_forecast_detail" IS 'CHECK constraint to ensure that the unit cost is not negative.';



ALTER TABLE "public"."fact_forecast_detail" ALTER COLUMN "forecast_detail_pk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."fact_forecast_detail_forecast_detail_pk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."map_scenario_configuration" (
    "scenario_configuration_sk" bigint NOT NULL,
    "study_sk" bigint NOT NULL,
    "scenario_sk" bigint NOT NULL,
    "start_date" "date",
    "end_date" "date",
    "target_enrollment" integer,
    "target_sites" integer,
    "study_status" "public"."study_status_enum",
    "is_deleted" boolean DEFAULT false NOT NULL,
    "organization_sk" bigint,
    "created_at" timestamp with time zone DEFAULT "now"() NOT NULL,
    "updated_at" timestamp with time zone DEFAULT "now"() NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "scenario_configuration_bk" "text",
    CONSTRAINT "chk_config_dates" CHECK ((("end_date" IS NULL) OR ("end_date" >= "start_date"))),
    CONSTRAINT "chk_config_target_enrollment" CHECK ((("target_enrollment" IS NULL) OR ("target_enrollment" >= 0))),
    CONSTRAINT "chk_parent_organization_match" CHECK (("public"."get_org_sk_for_study"("study_sk") = "public"."get_org_sk_for_scenario"("scenario_sk")))
);


ALTER TABLE "public"."map_scenario_configuration" OWNER TO "postgres";


COMMENT ON TABLE "public"."map_scenario_configuration" IS 'The central hub linking a study to a scenario. Why: Creates a concrete, versioned plan to which all other blueprint components (arms, sites, costs) are attached, forming the root of a complete forecastable model. This is a core "Blueprint" object.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."scenario_configuration_sk" IS 'Surrogate key for the configuration.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."study_sk" IS 'Links this configuration to a parent study protocol.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."scenario_sk" IS 'Links this configuration to a parent scenario (e.g., "Baseline").';



COMMENT ON COLUMN "public"."map_scenario_configuration"."start_date" IS 'The planned start date for this version of the study plan.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."end_date" IS 'The planned end date for this version of the study plan.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."target_enrollment" IS 'The total number of subjects to be enrolled across all arms in this plan.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."target_sites" IS 'The total number of clinical sites planned for this study scenario.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."study_status" IS 'The operational status of the study within this plan (e.g., ''Planning'', ''Enrolling'').';



COMMENT ON COLUMN "public"."map_scenario_configuration"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."organization_sk" IS 'The RLS key for tenant isolation, inherited from the parent study.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_scenario_configuration"."scenario_configuration_bk" IS 'Globally unique business key for this specific study-scenario plan, supporting the "Adoption" workflow.';



COMMENT ON CONSTRAINT "chk_config_dates" ON "public"."map_scenario_configuration" IS 'CHECK constraint to ensure that if an end_date is specified, it must be on or after the start_date.';



COMMENT ON CONSTRAINT "chk_config_target_enrollment" ON "public"."map_scenario_configuration" IS 'CHECK constraint to ensure the target enrollment is not negative.';



COMMENT ON CONSTRAINT "chk_parent_organization_match" ON "public"."map_scenario_configuration" IS 'CHECK constraint to enforce the business rule that a study and a scenario can only be linked if they belong to the same organization.';



ALTER TABLE "public"."map_scenario_configuration" ALTER COLUMN "scenario_configuration_sk" ADD GENERATED ALWAYS AS IDENTITY (
    SEQUENCE NAME "public"."map_scenario_configuration_scenario_configuration_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."map_study_partners" (
    "study_partner_sk" bigint NOT NULL,
    "partner_organization_sk" bigint NOT NULL,
    "partner_role" "public"."organization_type_enum" NOT NULL,
    "is_primary" boolean DEFAULT false NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "organization_sk" bigint,
    "created_at" timestamp with time zone DEFAULT "now"() NOT NULL,
    "updated_at" timestamp with time zone DEFAULT "now"() NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "scenario_configuration_sk" bigint NOT NULL,
    "study_partner_bk" "text" NOT NULL
);


ALTER TABLE "public"."map_study_partners" OWNER TO "postgres";


COMMENT ON TABLE "public"."map_study_partners" IS 'Assigns roles (e.g., Sponsor, CRO, Site, Vendor) to specific organizations for a given plan. Why: Defines the stakeholders and their financial and operational responsibilities (e.g., payer/payee). How: Links an organization to a scenario configuration.';



COMMENT ON COLUMN "public"."map_study_partners"."study_partner_sk" IS 'Surrogate key for the partner assignment.';



COMMENT ON COLUMN "public"."map_study_partners"."partner_organization_sk" IS 'Links this partner assignment to a specific organization from the organization directory.';



COMMENT ON COLUMN "public"."map_study_partners"."partner_role" IS 'The role this organization plays in the study (e.g., ''Sponsor'', ''CRO'', ''Lab'').';



COMMENT ON COLUMN "public"."map_study_partners"."is_primary" IS 'Flag indicating if this is the primary partner for a given role (e.g., the primary Sponsor).';



COMMENT ON COLUMN "public"."map_study_partners"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."map_study_partners"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."map_study_partners"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_partners"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_partners"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_partners"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_partners"."scenario_configuration_sk" IS 'Links this partner assignment to a specific plan.';



COMMENT ON COLUMN "public"."map_study_partners"."study_partner_bk" IS 'Globally unique business key for the partner assignment, supporting the "Adoption" workflow.';



ALTER TABLE "public"."map_study_partners" ALTER COLUMN "study_partner_sk" ADD GENERATED ALWAYS AS IDENTITY (
    SEQUENCE NAME "public"."map_study_partners_study_partner_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."map_study_staff" (
    "study_staff_sk" bigint NOT NULL,
    "study_staff_bk" "text" NOT NULL,
    "user_sk" bigint NOT NULL,
    "staff_role" "text" NOT NULL,
    "is_primary_contact" boolean DEFAULT false NOT NULL,
    "organization_sk" bigint NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "study_partner_sk" bigint NOT NULL
);


ALTER TABLE "public"."map_study_staff" OWNER TO "postgres";


COMMENT ON TABLE "public"."map_study_staff" IS 'Assigns users to study partners with a specific role. Why: Models the specific personnel (e.g., Principal Investigator, Study Coordinator) involved in the trial and their responsibilities.';



COMMENT ON COLUMN "public"."map_study_staff"."study_staff_sk" IS 'Surrogate key for the staff assignment.';



COMMENT ON COLUMN "public"."map_study_staff"."study_staff_bk" IS 'Globally unique business key for the staff assignment, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."map_study_staff"."user_sk" IS 'Links this staff assignment to a specific user from the user directory.';



COMMENT ON COLUMN "public"."map_study_staff"."staff_role" IS 'The role of the user in this context (e.g., "Principal Investigator").';



COMMENT ON COLUMN "public"."map_study_staff"."is_primary_contact" IS 'Flag indicating if this user is the primary point of contact for their assigned partner.';



COMMENT ON COLUMN "public"."map_study_staff"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."map_study_staff"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."map_study_staff"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_staff"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_staff"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_staff"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_staff"."study_partner_sk" IS 'Links this staff assignment to a specific study partner record.';



ALTER TABLE "public"."map_study_staff" ALTER COLUMN "study_staff_sk" ADD GENERATED BY DEFAULT AS IDENTITY (
    SEQUENCE NAME "public"."map_study_staff_study_staff_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."map_study_visit_activity" (
    "visit_sk" bigint NOT NULL,
    "activity_sk" bigint NOT NULL,
    "is_required" boolean DEFAULT true,
    "notes" "text",
    "created_by_user_sk" bigint,
    "updated_by_user_sk" bigint,
    "organization_sk" bigint NOT NULL,
    "created_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "updated_at" timestamp with time zone DEFAULT CURRENT_TIMESTAMP NOT NULL,
    "is_deleted" boolean DEFAULT false NOT NULL,
    "map_soa_bk" "text" DEFAULT "extensions"."uuid_generate_v4"() NOT NULL,
    "scenario_configuration_sk" bigint NOT NULL,
    "map_soa_sk" bigint NOT NULL
);


ALTER TABLE "public"."map_study_visit_activity" OWNER TO "postgres";


COMMENT ON TABLE "public"."map_study_visit_activity" IS 'The Schedule of Activities (SoA). Why: Defines which activities occur at which visits, forming the core of schedule-based forecasting and operational planning. This is a core "Blueprint" object.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."visit_sk" IS 'Links this SoA entry to a specific visit in the schedule.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."activity_sk" IS 'Links this SoA entry to a specific activity in the catalog.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."is_required" IS 'Flag indicating if the activity is mandatory for this visit.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."notes" IS 'Optional notes about this specific visit-activity combination.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."created_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."updated_by_user_sk" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."organization_sk" IS 'The RLS key for tenant isolation.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."updated_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."is_deleted" IS 'Soft-delete flag.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."map_soa_bk" IS 'Globally unique business key for the SoA mapping, supporting the "Adoption" workflow.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."scenario_configuration_sk" IS 'Links this SoA mapping to a specific plan.';



COMMENT ON COLUMN "public"."map_study_visit_activity"."map_soa_sk" IS 'Surrogate key for the SoA mapping.';



ALTER TABLE "public"."map_study_visit_activity" ALTER COLUMN "map_soa_sk" ADD GENERATED ALWAYS AS IDENTITY (
    SEQUENCE NAME "public"."map_study_visit_activity_map_soa_sk_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1
);



CREATE TABLE IF NOT EXISTS "public"."user_organization_membership" (
    "user_organization_membership_sk" bigint NOT NULL,
    "user_sk" bigint NOT NULL,
    "organization_sk" bigint NOT NULL,
    "role" "public"."user_role_enum",
    "created_at" timestamp with time zone DEFAULT "now"() NOT NULL,
    "updated_at" timestamp with time zone DEFAULT "now"() NOT NULL
);


ALTER TABLE "public"."user_organization_membership" OWNER TO "postgres";


COMMENT ON TABLE "public"."user_organization_membership" IS 'Links users to organizations with specific roles. Why: The core authorization primitive for the application. How: RLS policies on this table ensure users can only see memberships within their own organization''s hierarchy.';



COMMENT ON COLUMN "public"."user_organization_membership"."user_organization_membership_sk" IS 'Surrogate key for the membership record.';



COMMENT ON COLUMN "public"."user_organization_membership"."user_sk" IS 'Links this membership to a specific user.';



COMMENT ON COLUMN "public"."user_organization_membership"."organization_sk" IS 'Links this membership to a specific organization.';



COMMENT ON COLUMN "public"."user_organization_membership"."role" IS 'Defines the user''s permission level within the organization (''Admin'', ''Editor'', ''Viewer'').';



COMMENT ON COLUMN "public"."user_organization_membership"."created_at" IS 'Standard audit trail column.';



COMMENT ON COLUMN "public"."user_organization_membership"."updated_at" IS 'Standard audit trail column.';



CREATE SEQUENCE IF NOT EXISTS "public"."user_organization_membership_user_organization_membership_s_seq"
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE "public"."user_organization_membership_user_organization_membership_s_seq" OWNER TO "postgres";


ALTER SEQUENCE "public"."user_organization_membership_user_organization_membership_s_seq" OWNED BY "public"."user_organization_membership"."user_organization_membership_sk";




ALTER TABLE ONLY "public"."user_organization_membership" ALTER COLUMN "user_organization_membership_sk" SET DEFAULT "nextval"('"public"."user_organization_membership_user_organization_membership_s_seq"'::"regclass");





-- Migration: Partial flags for Budget Forecast facts
-- Adds is_partial boolean and missing_reasons jsonb to public.fact_forecast_detail
-- Also creates partial indexes and performs a one-time backfill based on deterministic rules.

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

-- Add columns (idempotent)
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1
    FROM information_schema.columns
    WHERE table_schema = 'public'
      AND table_name = 'fact_forecast_detail'
      AND column_name = 'is_partial'
  ) THEN
    ALTER TABLE public.fact_forecast_detail
      ADD COLUMN is_partial boolean NOT NULL DEFAULT false;
    COMMENT ON COLUMN public.fact_forecast_detail.is_partial IS 'Row-level completeness flag. true when one or more missing_reasons are present.';
  END IF;
END
$$;

DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1
    FROM information_schema.columns
    WHERE table_schema = 'public'
      AND table_name = 'fact_forecast_detail'
      AND column_name = 'missing_reasons'
  ) THEN
    ALTER TABLE public.fact_forecast_detail
      ADD COLUMN missing_reasons jsonb NOT NULL DEFAULT '[]'::jsonb;
    COMMENT ON COLUMN public.fact_forecast_detail.missing_reasons IS 'JSONB array of reason codes explaining why a row is partial (e.g., MISSING_FORECAST_RULE, MISSING_ACTIVITY_COST, UNIT_COST_ZERO, FORECAST_UNITS_NULL_OR_ZERO, MISSING_PAYER_OR_PAYEE, MISSING_VISIT_MAPPING, OUTSIDE_COST_EFFECTIVE_WINDOW).';
  END IF;
END
$$;

-- Indexes (idempotent)
CREATE INDEX IF NOT EXISTS ffd_is_partial_idx
  ON public.fact_forecast_detail (is_partial)
  WHERE is_partial = true;

CREATE INDEX IF NOT EXISTS ffd_missing_reasons_gin
  ON public.fact_forecast_detail
  USING GIN (missing_reasons jsonb_path_ops);

-- Backfill existing rows using deterministic rules
WITH computed AS (
  SELECT
    fd.forecast_detail_pk,
    (
      '[]'::jsonb
      || CASE WHEN fd.source_forecast_config_sk IS NULL THEN '["MISSING_FORECAST_RULE"]'::jsonb ELSE '[]'::jsonb END
      || CASE WHEN fd.source_activity_cost_sk IS NULL THEN '["MISSING_ACTIVITY_COST"]'::jsonb ELSE '[]'::jsonb END
      || CASE WHEN COALESCE(fd.unit_cost, 0) = 0 AND fd.source_activity_cost_sk IS NOT NULL THEN '["UNIT_COST_ZERO"]'::jsonb ELSE '[]'::jsonb END
      || CASE WHEN COALESCE(fd.forecast_units, 0) = 0 THEN '["FORECAST_UNITS_NULL_OR_ZERO"]'::jsonb ELSE '[]'::jsonb END
      || CASE WHEN fd.payer_partner_sk IS NULL OR fd.payee_partner_sk IS NULL THEN '["MISSING_PAYER_OR_PAYEE"]'::jsonb ELSE '[]'::jsonb END
      || CASE WHEN fd.source_visit_sk IS NOT NULL AND fd.source_map_soa_sk IS NULL THEN '["MISSING_VISIT_MAPPING"]'::jsonb ELSE '[]'::jsonb END
      || CASE
           WHEN fd.source_activity_cost_sk IS NOT NULL
            AND fd.snapshot_date_sk IS NOT NULL
            AND NOT EXISTS (
              SELECT 1
              FROM public.dim_activity_cost ac
              WHERE ac.activity_cost_sk = fd.source_activity_cost_sk
                AND fd.snapshot_date_sk BETWEEN ac.effective_date AND ac.end_date
            )
           THEN '["OUTSIDE_COST_EFFECTIVE_WINDOW"]'::jsonb
           ELSE '[]'::jsonb
         END
    ) AS reasons
  FROM public.fact_forecast_detail fd
  WHERE fd.is_deleted = false
)
UPDATE public.fact_forecast_detail fd
SET missing_reasons = c.reasons,
    is_partial = (jsonb_array_length(c.reasons) > 0)
FROM computed c
WHERE fd.forecast_detail_pk = c.forecast_detail_pk;



ALTER TABLE ONLY "private"."materialized_template_payload"
    ADD CONSTRAINT "materialized_template_payload_pkey" PRIMARY KEY ("id");



COMMENT ON CONSTRAINT "materialized_template_payload_pkey" ON "private"."materialized_template_payload" IS 'PRIMARY KEY constraint to uniquely identify each materialized row in the cache.';



ALTER TABLE ONLY "private"."materialized_template_payload"
    ADD CONSTRAINT "materialized_template_payload_template_table_name_key" UNIQUE ("template_table_name");



COMMENT ON CONSTRAINT "materialized_template_payload_template_table_name_key" ON "private"."materialized_template_payload" IS 'UNIQUE constraint to ensure that only one materialized payload exists per source template table.';



ALTER TABLE ONLY "private"."template_study_partners"
    ADD CONSTRAINT "pk_template_study_partners_final" PRIMARY KEY ("study_partner_bk");



COMMENT ON CONSTRAINT "pk_template_study_partners_final" ON "private"."template_study_partners" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template partner assignment.';



ALTER TABLE ONLY "private"."template_study_staff"
    ADD CONSTRAINT "pk_template_study_staff" PRIMARY KEY ("study_staff_bk");



COMMENT ON CONSTRAINT "pk_template_study_staff" ON "private"."template_study_staff" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template staff assignment.';



ALTER TABLE ONLY "private"."sync_log"
    ADD CONSTRAINT "sync_log_pkey" PRIMARY KEY ("sync_type");



COMMENT ON CONSTRAINT "sync_log_pkey" ON "private"."sync_log" IS 'PRIMARY KEY constraint to uniquely identify each sync job type.';



ALTER TABLE ONLY "private"."template_activities"
    ADD CONSTRAINT "template_activities_pkey" PRIMARY KEY ("activity_bk");



COMMENT ON CONSTRAINT "template_activities_pkey" ON "private"."template_activities" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template activity.';



ALTER TABLE ONLY "private"."template_activity_costs"
    ADD CONSTRAINT "template_activity_costs_clean_pkey" PRIMARY KEY ("cost_bk");



COMMENT ON CONSTRAINT "template_activity_costs_clean_pkey" ON "private"."template_activity_costs" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template activity cost record.';



ALTER TABLE ONLY "private"."template_amendments"
    ADD CONSTRAINT "template_amendments_pkey" PRIMARY KEY ("amendment_bk");



COMMENT ON CONSTRAINT "template_amendments_pkey" ON "private"."template_amendments" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template amendment.';



ALTER TABLE ONLY "private"."template_budget_categories"
    ADD CONSTRAINT "template_budget_categories_pkey" PRIMARY KEY ("budget_category_bk");



COMMENT ON CONSTRAINT "template_budget_categories_pkey" ON "private"."template_budget_categories" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template budget category.';



ALTER TABLE ONLY "private"."template_fact_enrollment"
    ADD CONSTRAINT "template_fact_enrollment_pkey" PRIMARY KEY ("enrollment_bk");



COMMENT ON CONSTRAINT "template_fact_enrollment_pkey" ON "private"."template_fact_enrollment" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template enrollment fact record.';



ALTER TABLE ONLY "private"."template_fact_forecast_detail"
    ADD CONSTRAINT "template_fact_forecast_detail_pkey" PRIMARY KEY ("forecast_detail_bk");



COMMENT ON CONSTRAINT "template_fact_forecast_detail_pkey" ON "private"."template_fact_forecast_detail" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template forecast detail fact record.';



ALTER TABLE ONLY "private"."template_forecast_calculation_configs"
    ADD CONSTRAINT "template_forecast_calculation_configs_pkey" PRIMARY KEY ("config_bk");



COMMENT ON CONSTRAINT "template_forecast_calculation_configs_pkey" ON "private"."template_forecast_calculation_configs" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template forecast rule.';



ALTER TABLE ONLY "private"."template_memberships"
    ADD CONSTRAINT "template_memberships_pkey" PRIMARY KEY ("user_bk", "organization_bk");



COMMENT ON CONSTRAINT "template_memberships_pkey" ON "private"."template_memberships" IS 'PRIMARY KEY constraint to ensure a user can only have one membership (and one role) per organization in the template data.';



ALTER TABLE ONLY "private"."template_organizations"
    ADD CONSTRAINT "template_organizations_pkey" PRIMARY KEY ("organization_bk");



COMMENT ON CONSTRAINT "template_organizations_pkey" ON "private"."template_organizations" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template organization.';



ALTER TABLE ONLY "private"."template_reimbursement_types"
    ADD CONSTRAINT "template_reimbursement_types_pkey" PRIMARY KEY ("reimbursement_type_bk");



COMMENT ON CONSTRAINT "template_reimbursement_types_pkey" ON "private"."template_reimbursement_types" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template reimbursement type.';



ALTER TABLE ONLY "private"."template_reimbursement_types"
    ADD CONSTRAINT "template_reimbursement_types_reimbursement_type_code_key" UNIQUE ("reimbursement_type_code");



COMMENT ON CONSTRAINT "template_reimbursement_types_reimbursement_type_code_key" ON "private"."template_reimbursement_types" IS 'UNIQUE constraint to ensure the short code for a reimbursement type is unique.';



ALTER TABLE ONLY "private"."template_scenario_configurations"
    ADD CONSTRAINT "template_scenario_configurations_pkey" PRIMARY KEY ("scenario_configuration_bk");



COMMENT ON CONSTRAINT "template_scenario_configurations_pkey" ON "private"."template_scenario_configurations" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template scenario configuration.';



ALTER TABLE ONLY "private"."template_scenarios"
    ADD CONSTRAINT "template_scenarios_pkey" PRIMARY KEY ("scenario_bk");



COMMENT ON CONSTRAINT "template_scenarios_pkey" ON "private"."template_scenarios" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template scenario.';



ALTER TABLE ONLY "private"."template_sites"
    ADD CONSTRAINT "template_sites_pkey" PRIMARY KEY ("site_bk");



COMMENT ON CONSTRAINT "template_sites_pkey" ON "private"."template_sites" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template site assignment.';



ALTER TABLE ONLY "private"."template_soa_mappings"
    ADD CONSTRAINT "template_soa_mappings_pkey" PRIMARY KEY ("map_soa_bk");



COMMENT ON CONSTRAINT "template_soa_mappings_pkey" ON "private"."template_soa_mappings" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template SoA mapping.';



ALTER TABLE ONLY "private"."template_studies"
    ADD CONSTRAINT "template_studies_pkey" PRIMARY KEY ("study_bk");



COMMENT ON CONSTRAINT "template_studies_pkey" ON "private"."template_studies" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template study.';



ALTER TABLE ONLY "private"."template_study_arms"
    ADD CONSTRAINT "template_study_arms_pkey" PRIMARY KEY ("arm_bk");



COMMENT ON CONSTRAINT "template_study_arms_pkey" ON "private"."template_study_arms" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template study arm.';



ALTER TABLE ONLY "private"."template_study_epochs"
    ADD CONSTRAINT "template_study_epochs_pkey" PRIMARY KEY ("epoch_bk");



COMMENT ON CONSTRAINT "template_study_epochs_pkey" ON "private"."template_study_epochs" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template epoch.';



ALTER TABLE ONLY "private"."template_study_visits"
    ADD CONSTRAINT "template_study_visits_pkey" PRIMARY KEY ("visit_bk");



COMMENT ON CONSTRAINT "template_study_visits_pkey" ON "private"."template_study_visits" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template visit.';



ALTER TABLE ONLY "private"."template_users"
    ADD CONSTRAINT "template_users_pkey" PRIMARY KEY ("user_bk");



COMMENT ON CONSTRAINT "template_users_pkey" ON "private"."template_users" IS 'PRIMARY KEY constraint using the business key to uniquely identify each template user.';



ALTER TABLE ONLY "private"."template_forecast_calculation_configs"
    ADD CONSTRAINT "uq_config_per_activity_scenario_config" UNIQUE ("parent_scenario_configuration_bk", "activity_bk");



COMMENT ON CONSTRAINT "uq_config_per_activity_scenario_config" ON "private"."template_forecast_calculation_configs" IS 'UNIQUE constraint to prevent defining more than one forecast rule for the same activity within the same scenario configuration.';



ALTER TABLE ONLY "private"."template_activity_costs"
    ADD CONSTRAINT "uq_cost_per_config_activity" UNIQUE ("parent_scenario_configuration_bk", "activity_bk");



COMMENT ON CONSTRAINT "uq_cost_per_config_activity" ON "private"."template_activity_costs" IS 'UNIQUE constraint to prevent defining more than one cost for the same activity within the same scenario configuration, ensuring a single source of price.';



ALTER TABLE ONLY "private"."template_soa_mappings"
    ADD CONSTRAINT "uq_soa_mapping_per_config" UNIQUE ("parent_scenario_configuration_bk", "visit_bk", "activity_bk");



COMMENT ON CONSTRAINT "uq_soa_mapping_per_config" ON "private"."template_soa_mappings" IS 'UNIQUE constraint to prevent scheduling the same activity at the same visit more than once within a single plan.';



ALTER TABLE ONLY "private"."template_study_arms"
    ADD CONSTRAINT "uq_template_arm_name_per_scenario_config" UNIQUE ("parent_scenario_configuration_bk", "arm_name");



COMMENT ON CONSTRAINT "uq_template_arm_name_per_scenario_config" ON "private"."template_study_arms" IS 'UNIQUE constraint to ensure arm names are unique within a single plan.';



ALTER TABLE ONLY "private"."template_study_epochs"
    ADD CONSTRAINT "uq_template_epoch_name_per_arm" UNIQUE ("parent_arm_bk", "epoch_name");



COMMENT ON CONSTRAINT "uq_template_epoch_name_per_arm" ON "private"."template_study_epochs" IS 'UNIQUE constraint to ensure epoch names are unique within a single arm.';



ALTER TABLE ONLY "private"."template_study_epochs"
    ADD CONSTRAINT "uq_template_epoch_order_per_arm" UNIQUE ("parent_arm_bk", "epoch_order");



COMMENT ON CONSTRAINT "uq_template_epoch_order_per_arm" ON "private"."template_study_epochs" IS 'UNIQUE constraint to ensure epoch order is unique within a single arm.';



ALTER TABLE ONLY "private"."template_scenario_configurations"
    ADD CONSTRAINT "uq_template_scenario_study_pair" UNIQUE ("parent_scenario_bk", "parent_study_bk");



COMMENT ON CONSTRAINT "uq_template_scenario_study_pair" ON "private"."template_scenario_configurations" IS 'UNIQUE constraint to ensure that a study can only be linked to a specific scenario once, preventing duplicate plans.';



ALTER TABLE ONLY "private"."template_study_visits"
    ADD CONSTRAINT "uq_template_visit_name_per_epoch" UNIQUE ("parent_epoch_bk", "visit_name");



COMMENT ON CONSTRAINT "uq_template_visit_name_per_epoch" ON "private"."template_study_visits" IS 'UNIQUE constraint to ensure visit names are unique within a single epoch.';



ALTER TABLE ONLY "private"."template_study_visits"
    ADD CONSTRAINT "uq_template_visit_order_per_epoch" UNIQUE ("parent_epoch_bk", "visit_order");



COMMENT ON CONSTRAINT "uq_template_visit_order_per_epoch" ON "private"."template_study_visits" IS 'UNIQUE constraint to ensure visit order is unique within a single epoch.';



ALTER TABLE ONLY "public"."agent_memory_log"
    ADD CONSTRAINT "agent_memory_log_pkey" PRIMARY KEY ("log_id");



COMMENT ON CONSTRAINT "agent_memory_log_pkey" ON "public"."agent_memory_log" IS 'PRIMARY KEY constraint to uniquely identify each agent log entry.';



ALTER TABLE ONLY "public"."checkpoint_blobs"
    ADD CONSTRAINT "checkpoint_blobs_pkey" PRIMARY KEY ("thread_id", "checkpoint_ns", "channel", "version");



COMMENT ON CONSTRAINT "checkpoint_blobs_pkey" ON "public"."checkpoint_blobs" IS 'PRIMARY KEY constraint to uniquely identify a specific version of a blob for a given channel in a thread.';



ALTER TABLE ONLY "public"."checkpoint_migrations"
    ADD CONSTRAINT "checkpoint_migrations_pkey" PRIMARY KEY ("v");



COMMENT ON CONSTRAINT "checkpoint_migrations_pkey" ON "public"."checkpoint_migrations" IS 'PRIMARY KEY constraint to uniquely identify each checkpoint schema migration version.';



ALTER TABLE ONLY "public"."checkpoint_writes"
    ADD CONSTRAINT "checkpoint_writes_pkey" PRIMARY KEY ("thread_id", "checkpoint_ns", "checkpoint_id", "task_id", "idx");



COMMENT ON CONSTRAINT "checkpoint_writes_pkey" ON "public"."checkpoint_writes" IS 'PRIMARY KEY constraint to uniquely identify a single write operation within a task of a checkpoint.';



ALTER TABLE ONLY "public"."checkpoints"
    ADD CONSTRAINT "checkpoints_pkey" PRIMARY KEY ("thread_id", "checkpoint_ns", "checkpoint_id");



COMMENT ON CONSTRAINT "checkpoints_pkey" ON "public"."checkpoints" IS 'PRIMARY KEY constraint to uniquely identify a checkpoint within a specific thread and namespace.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "dim_activity_cost_pkey" PRIMARY KEY ("activity_cost_sk");



COMMENT ON CONSTRAINT "dim_activity_cost_pkey" ON "public"."dim_activity_cost" IS 'PRIMARY KEY constraint to uniquely identify each activity cost record.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "dim_activity_pkey" PRIMARY KEY ("activity_sk");



COMMENT ON CONSTRAINT "dim_activity_pkey" ON "public"."dim_activity" IS 'PRIMARY KEY constraint to uniquely identify each activity record.';



ALTER TABLE ONLY "public"."dim_amendment"
    ADD CONSTRAINT "dim_amendment_pkey" PRIMARY KEY ("amendment_sk");



COMMENT ON CONSTRAINT "dim_amendment_pkey" ON "public"."dim_amendment" IS 'PRIMARY KEY constraint to uniquely identify each amendment record.';



ALTER TABLE ONLY "public"."dim_budget_category"
    ADD CONSTRAINT "dim_budget_category_pkey" PRIMARY KEY ("budget_category_sk");



COMMENT ON CONSTRAINT "dim_budget_category_pkey" ON "public"."dim_budget_category" IS 'PRIMARY KEY constraint to uniquely identify each budget category record.';



ALTER TABLE ONLY "public"."dim_budget_scenario"
    ADD CONSTRAINT "dim_budget_scenario_pkey" PRIMARY KEY ("scenario_sk");



COMMENT ON CONSTRAINT "dim_budget_scenario_pkey" ON "public"."dim_budget_scenario" IS 'PRIMARY KEY constraint to uniquely identify each budget scenario record.';



ALTER TABLE ONLY "public"."dim_date"
    ADD CONSTRAINT "dim_date_pkey" PRIMARY KEY ("date_sk");



COMMENT ON CONSTRAINT "dim_date_pkey" ON "public"."dim_date" IS 'PRIMARY KEY constraint to uniquely identify each day in the calendar.';



ALTER TABLE ONLY "public"."dim_forecast_calculation_config"
    ADD CONSTRAINT "dim_forecast_calculation_config_pkey" PRIMARY KEY ("forecast_config_sk");



COMMENT ON CONSTRAINT "dim_forecast_calculation_config_pkey" ON "public"."dim_forecast_calculation_config" IS 'PRIMARY KEY constraint to uniquely identify each forecast rule record.';



ALTER TABLE ONLY "public"."dim_model_costs"
    ADD CONSTRAINT "dim_model_costs_pkey" PRIMARY KEY ("model_name");



COMMENT ON CONSTRAINT "dim_model_costs_pkey" ON "public"."dim_model_costs" IS 'PRIMARY KEY constraint to uniquely identify the pricing for each LLM model.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "dim_organization_bk_key_sandbox" UNIQUE ("organization_bk", "parent_organization_sk");



COMMENT ON CONSTRAINT "dim_organization_bk_key_sandbox" ON "public"."dim_organization" IS 'UNIQUE constraint to ensure that fictitious (sandbox) organization business keys are unique within their parent organization.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "dim_organization_organization_bk_key" UNIQUE ("organization_bk");



COMMENT ON CONSTRAINT "dim_organization_organization_bk_key" ON "public"."dim_organization" IS 'UNIQUE constraint to ensure that organization business keys are globally unique. This is primarily for Clerk-managed root organizations.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "dim_organization_pkey" PRIMARY KEY ("organization_sk");



COMMENT ON CONSTRAINT "dim_organization_pkey" ON "public"."dim_organization" IS 'PRIMARY KEY constraint to uniquely identify each organization record.';



ALTER TABLE ONLY "public"."dim_plan_limits"
    ADD CONSTRAINT "dim_plan_limits_pkey" PRIMARY KEY ("plan");



COMMENT ON CONSTRAINT "dim_plan_limits_pkey" ON "public"."dim_plan_limits" IS 'PRIMARY KEY constraint to uniquely identify the limits for each subscription plan tier.';



ALTER TABLE ONLY "public"."dim_reimbursement_type"
    ADD CONSTRAINT "dim_reimbursement_type_pkey" PRIMARY KEY ("reimbursement_type_sk");



COMMENT ON CONSTRAINT "dim_reimbursement_type_pkey" ON "public"."dim_reimbursement_type" IS 'PRIMARY KEY constraint to uniquely identify each reimbursement type record.';



ALTER TABLE ONLY "public"."dim_site"
    ADD CONSTRAINT "dim_site_pkey" PRIMARY KEY ("site_sk");



COMMENT ON CONSTRAINT "dim_site_pkey" ON "public"."dim_site" IS 'PRIMARY KEY constraint to uniquely identify each site assignment record.';



ALTER TABLE ONLY "public"."dim_study_arm"
    ADD CONSTRAINT "dim_study_arm_pkey" PRIMARY KEY ("arm_sk");



COMMENT ON CONSTRAINT "dim_study_arm_pkey" ON "public"."dim_study_arm" IS 'PRIMARY KEY constraint to uniquely identify each study arm record.';



ALTER TABLE ONLY "public"."dim_study_epochs"
    ADD CONSTRAINT "dim_study_epochs_pkey" PRIMARY KEY ("epoch_sk");



COMMENT ON CONSTRAINT "dim_study_epochs_pkey" ON "public"."dim_study_epochs" IS 'PRIMARY KEY constraint to uniquely identify each study epoch record.';



ALTER TABLE ONLY "public"."dim_study"
    ADD CONSTRAINT "dim_study_pkey" PRIMARY KEY ("study_sk");



COMMENT ON CONSTRAINT "dim_study_pkey" ON "public"."dim_study" IS 'PRIMARY KEY constraint to uniquely identify each study record.';



ALTER TABLE ONLY "public"."dim_study_visits"
    ADD CONSTRAINT "dim_study_visits_pkey" PRIMARY KEY ("visit_sk");



COMMENT ON CONSTRAINT "dim_study_visits_pkey" ON "public"."dim_study_visits" IS 'PRIMARY KEY constraint to uniquely identify each study visit record.';



ALTER TABLE ONLY "public"."dim_user"
    ADD CONSTRAINT "dim_user_pkey" PRIMARY KEY ("user_sk");



COMMENT ON CONSTRAINT "dim_user_pkey" ON "public"."dim_user" IS 'PRIMARY KEY constraint to uniquely identify each user record.';



ALTER TABLE ONLY "public"."fact_enrollment"
    ADD CONSTRAINT "fact_enrollment_pkey" PRIMARY KEY ("enrollment_pk");



COMMENT ON CONSTRAINT "fact_enrollment_pkey" ON "public"."fact_enrollment" IS 'PRIMARY KEY constraint to uniquely identify each enrollment fact record.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fact_forecast_detail_pkey" PRIMARY KEY ("forecast_detail_pk");



COMMENT ON CONSTRAINT "fact_forecast_detail_pkey" ON "public"."fact_forecast_detail" IS 'PRIMARY KEY constraint to uniquely identify each forecast detail fact record.';



ALTER TABLE ONLY "public"."fact_user_token_usage"
    ADD CONSTRAINT "fact_user_token_usage_pkey" PRIMARY KEY ("run_id");



COMMENT ON CONSTRAINT "fact_user_token_usage_pkey" ON "public"."fact_user_token_usage" IS 'PRIMARY KEY constraint to uniquely identify each token usage record by its run ID.';



ALTER TABLE ONLY "public"."fact_user_token_usage"
    ADD CONSTRAINT "fact_user_token_usage_run_id_key" UNIQUE ("run_id");



COMMENT ON CONSTRAINT "fact_user_token_usage_run_id_key" ON "public"."fact_user_token_usage" IS 'UNIQUE constraint on the run_id to enforce it as the primary business key.';



ALTER TABLE ONLY "public"."map_scenario_configuration"
    ADD CONSTRAINT "map_scenario_configuration_pkey" PRIMARY KEY ("scenario_configuration_sk");



COMMENT ON CONSTRAINT "map_scenario_configuration_pkey" ON "public"."map_scenario_configuration" IS 'PRIMARY KEY constraint to uniquely identify each scenario configuration record.';



ALTER TABLE ONLY "public"."map_shared_scenario"
    ADD CONSTRAINT "map_shared_scenario_pkey" PRIMARY KEY ("shared_scenario_sk");



COMMENT ON CONSTRAINT "map_shared_scenario_pkey" ON "public"."map_shared_scenario" IS 'PRIMARY KEY constraint to uniquely identify each scenario sharing record.';



ALTER TABLE ONLY "public"."map_shared_scenario"
    ADD CONSTRAINT "map_shared_scenario_sponsor_organization_sk_vendor_organiza_key" UNIQUE ("sponsor_organization_sk", "vendor_organization_sk", "budget_scenario_sk");



COMMENT ON CONSTRAINT "map_shared_scenario_sponsor_organization_sk_vendor_organiza_key" ON "public"."map_shared_scenario" IS 'UNIQUE constraint to prevent sharing the same scenario with the same vendor more than once.';



ALTER TABLE ONLY "public"."map_study_partners"
    ADD CONSTRAINT "map_study_partners_pkey" PRIMARY KEY ("study_partner_sk");



COMMENT ON CONSTRAINT "map_study_partners_pkey" ON "public"."map_study_partners" IS 'PRIMARY KEY constraint to uniquely identify each partner assignment record.';



ALTER TABLE ONLY "public"."map_study_staff"
    ADD CONSTRAINT "map_study_staff_pkey" PRIMARY KEY ("study_staff_sk");



COMMENT ON CONSTRAINT "map_study_staff_pkey" ON "public"."map_study_staff" IS 'PRIMARY KEY constraint to uniquely identify each staff assignment record.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "pk_map_study_visit_activity" PRIMARY KEY ("map_soa_sk");



COMMENT ON CONSTRAINT "pk_map_study_visit_activity" ON "public"."map_study_visit_activity" IS 'PRIMARY KEY constraint to uniquely identify each SoA mapping record.';



ALTER TABLE ONLY "public"."schema_version"
    ADD CONSTRAINT "schema_version_pkey" PRIMARY KEY ("version_id");



COMMENT ON CONSTRAINT "schema_version_pkey" ON "public"."schema_version" IS 'PRIMARY KEY constraint to uniquely identify each schema version.';



ALTER TABLE ONLY "public"."subscriptions"
    ADD CONSTRAINT "subscriptions_organization_bk_key" UNIQUE ("organization_bk");



COMMENT ON CONSTRAINT "subscriptions_organization_bk_key" ON "public"."subscriptions" IS 'UNIQUE constraint to ensure only one subscription record exists per organization.';



ALTER TABLE ONLY "public"."subscriptions"
    ADD CONSTRAINT "subscriptions_pkey" PRIMARY KEY ("subscription_id");



COMMENT ON CONSTRAINT "subscriptions_pkey" ON "public"."subscriptions" IS 'PRIMARY KEY constraint to uniquely identify each subscription record.';



ALTER TABLE ONLY "public"."subscriptions"
    ADD CONSTRAINT "subscriptions_stripe_customer_id_key" UNIQUE ("stripe_customer_id");



COMMENT ON CONSTRAINT "subscriptions_stripe_customer_id_key" ON "public"."subscriptions" IS 'UNIQUE constraint on the Stripe customer ID.';



ALTER TABLE ONLY "public"."subscriptions"
    ADD CONSTRAINT "subscriptions_stripe_subscription_id_key" UNIQUE ("stripe_subscription_id");



COMMENT ON CONSTRAINT "subscriptions_stripe_subscription_id_key" ON "public"."subscriptions" IS 'UNIQUE constraint on the Stripe subscription ID.';



ALTER TABLE ONLY "public"."synced_clerk_invitations"
    ADD CONSTRAINT "synced_clerk_invitations_pkey" PRIMARY KEY ("invitation_id");



COMMENT ON CONSTRAINT "synced_clerk_invitations_pkey" ON "public"."synced_clerk_invitations" IS 'PRIMARY KEY constraint to uniquely identify each synced invitation record using the ID from Clerk.';



ALTER TABLE ONLY "public"."threads"
    ADD CONSTRAINT "threads_pkey" PRIMARY KEY ("thread_id");



COMMENT ON CONSTRAINT "threads_pkey" ON "public"."threads" IS 'PRIMARY KEY constraint to uniquely identify each agent conversation thread.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "uq_activity_cost_bk" UNIQUE ("organization_sk", "activity_cost_bk");



COMMENT ON CONSTRAINT "uq_activity_cost_bk" ON "public"."dim_activity_cost" IS 'UNIQUE constraint to ensure that activity cost business keys are unique within a single organization.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "uq_activity_org_name" UNIQUE ("organization_sk", "activity_name");



COMMENT ON CONSTRAINT "uq_activity_org_name" ON "public"."dim_activity" IS 'UNIQUE constraint to ensure that activity names are unique within a single organization.';



ALTER TABLE ONLY "public"."dim_study_arm"
    ADD CONSTRAINT "uq_arm_name_per_scenario_config" UNIQUE ("scenario_configuration_sk", "arm_name");



COMMENT ON CONSTRAINT "uq_arm_name_per_scenario_config" ON "public"."dim_study_arm" IS 'UNIQUE constraint to ensure arm names are unique within a single plan.';



ALTER TABLE ONLY "public"."dim_budget_category"
    ADD CONSTRAINT "uq_category_name_per_parent" UNIQUE ("organization_sk", "parent_category_sk", "category_name");



COMMENT ON CONSTRAINT "uq_category_name_per_parent" ON "public"."dim_budget_category" IS 'UNIQUE constraint to ensure that sibling categories in the hierarchy have unique names.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "uq_dim_activity_bk" UNIQUE ("organization_sk", "activity_bk");



COMMENT ON CONSTRAINT "uq_dim_activity_bk" ON "public"."dim_activity" IS 'UNIQUE constraint to ensure that activity business keys are unique within a single organization.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "uq_dim_organization_fictitious_name_per_parent" UNIQUE ("parent_organization_sk", "organization_name");



COMMENT ON CONSTRAINT "uq_dim_organization_fictitious_name_per_parent" ON "public"."dim_organization" IS 'UNIQUE constraint to ensure that fictitious organization names are unique within their parent organization.';



ALTER TABLE ONLY "public"."dim_study_epochs"
    ADD CONSTRAINT "uq_epoch_name_per_arm" UNIQUE ("arm_sk", "epoch_name");



COMMENT ON CONSTRAINT "uq_epoch_name_per_arm" ON "public"."dim_study_epochs" IS 'UNIQUE constraint to ensure epoch names are unique within a single arm.';



ALTER TABLE ONLY "public"."dim_study_epochs"
    ADD CONSTRAINT "uq_epoch_order_per_arm" UNIQUE ("arm_sk", "epoch_order");



COMMENT ON CONSTRAINT "uq_epoch_order_per_arm" ON "public"."dim_study_epochs" IS 'UNIQUE constraint to ensure epoch order is unique within a single arm.';



ALTER TABLE ONLY "public"."fact_enrollment"
    ADD CONSTRAINT "uq_fact_enrollment_bk" UNIQUE ("organization_sk", "enrollment_bk");



COMMENT ON CONSTRAINT "uq_fact_enrollment_bk" ON "public"."fact_enrollment" IS 'UNIQUE constraint to ensure enrollment business keys are unique within an organization.';



ALTER TABLE ONLY "public"."fact_enrollment"
    ADD CONSTRAINT "uq_fact_enrollment_grain_new" UNIQUE ("organization_sk", "snapshot_date_sk", "site_sk", "scenario_configuration_sk");



COMMENT ON CONSTRAINT "uq_fact_enrollment_grain_new" ON "public"."fact_enrollment" IS 'UNIQUE constraint defining the natural key of an enrollment fact: one record per date, per site, per plan.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "uq_fact_forecast_detail_bk" UNIQUE ("organization_sk", "forecast_detail_bk");



COMMENT ON CONSTRAINT "uq_fact_forecast_detail_bk" ON "public"."fact_forecast_detail" IS 'UNIQUE constraint to ensure forecast detail business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "uq_fictitious_org_bk_per_parent" UNIQUE ("parent_organization_sk", "organization_bk");



COMMENT ON CONSTRAINT "uq_fictitious_org_bk_per_parent" ON "public"."dim_organization" IS 'UNIQUE constraint to ensure that fictitious organization business keys are unique within their parent organization.';



ALTER TABLE ONLY "public"."dim_user"
    ADD CONSTRAINT "uq_fictitious_user_bk_global" UNIQUE ("user_bk");



COMMENT ON CONSTRAINT "uq_fictitious_user_bk_global" ON "public"."dim_user" IS 'UNIQUE constraint to ensure that all user business keys (both Clerk-managed and fictitious) are globally unique.';



ALTER TABLE ONLY "public"."dim_user"
    ADD CONSTRAINT "uq_fictitious_user_bk_per_org" UNIQUE ("organization_sk", "user_bk");



COMMENT ON CONSTRAINT "uq_fictitious_user_bk_per_org" ON "public"."dim_user" IS 'UNIQUE constraint to ensure fictitious user business keys are unique within their sandbox organization.';



ALTER TABLE ONLY "public"."dim_forecast_calculation_config"
    ADD CONSTRAINT "uq_forecast_config_name_per_scenario" UNIQUE ("scenario_configuration_sk", "config_name");



COMMENT ON CONSTRAINT "uq_forecast_config_name_per_scenario" ON "public"."dim_forecast_calculation_config" IS 'UNIQUE constraint to ensure forecast rule names are unique within a single plan.';



ALTER TABLE ONLY "public"."map_scenario_configuration"
    ADD CONSTRAINT "uq_map_scenario_config_bk" UNIQUE ("organization_sk", "scenario_configuration_bk");



COMMENT ON CONSTRAINT "uq_map_scenario_config_bk" ON "public"."map_scenario_configuration" IS 'UNIQUE constraint to ensure scenario configuration business keys are unique within an organization.';



ALTER TABLE ONLY "public"."map_scenario_configuration"
    ADD CONSTRAINT "uq_map_scenario_config_parents" UNIQUE ("scenario_sk", "study_sk");



COMMENT ON CONSTRAINT "uq_map_scenario_config_parents" ON "public"."map_scenario_configuration" IS 'UNIQUE constraint to ensure that a study can only be linked to a specific scenario once, preventing duplicate plans.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "uq_map_soa_bk" UNIQUE ("organization_sk", "map_soa_bk");



COMMENT ON CONSTRAINT "uq_map_soa_bk" ON "public"."map_study_visit_activity" IS 'UNIQUE constraint to ensure SoA mapping business keys are unique within an organization.';



ALTER TABLE ONLY "public"."map_study_partners"
    ADD CONSTRAINT "uq_map_study_partners_bk" UNIQUE ("organization_sk", "study_partner_bk");



COMMENT ON CONSTRAINT "uq_map_study_partners_bk" ON "public"."map_study_partners" IS 'UNIQUE constraint to ensure partner assignment business keys are unique within an organization.';



ALTER TABLE ONLY "public"."map_study_staff"
    ADD CONSTRAINT "uq_map_study_staff_bk" UNIQUE ("organization_sk", "study_staff_bk");



COMMENT ON CONSTRAINT "uq_map_study_staff_bk" ON "public"."map_study_staff" IS 'UNIQUE constraint to ensure staff assignment business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_study_arm"
    ADD CONSTRAINT "uq_org_arm_bk" UNIQUE ("organization_sk", "arm_bk");



COMMENT ON CONSTRAINT "uq_org_arm_bk" ON "public"."dim_study_arm" IS 'UNIQUE constraint to ensure arm business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_budget_category"
    ADD CONSTRAINT "uq_org_budget_category_bk" UNIQUE ("organization_sk", "budget_category_bk");



COMMENT ON CONSTRAINT "uq_org_budget_category_bk" ON "public"."dim_budget_category" IS 'UNIQUE constraint to ensure budget category business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_study_epochs"
    ADD CONSTRAINT "uq_org_epoch_bk" UNIQUE ("organization_sk", "epoch_bk");



COMMENT ON CONSTRAINT "uq_org_epoch_bk" ON "public"."dim_study_epochs" IS 'UNIQUE constraint to ensure epoch business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_forecast_calculation_config"
    ADD CONSTRAINT "uq_org_forecast_config_bk" UNIQUE ("organization_sk", "forecast_config_bk");



COMMENT ON CONSTRAINT "uq_org_forecast_config_bk" ON "public"."dim_forecast_calculation_config" IS 'UNIQUE constraint to ensure forecast rule business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_reimbursement_type"
    ADD CONSTRAINT "uq_org_reimbursement_type_bk" UNIQUE ("organization_sk", "reimbursement_type_bk");



COMMENT ON CONSTRAINT "uq_org_reimbursement_type_bk" ON "public"."dim_reimbursement_type" IS 'UNIQUE constraint to ensure reimbursement type business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_budget_scenario"
    ADD CONSTRAINT "uq_org_scenario_bk" UNIQUE ("organization_sk", "scenario_bk");



COMMENT ON CONSTRAINT "uq_org_scenario_bk" ON "public"."dim_budget_scenario" IS 'UNIQUE constraint to ensure scenario business keys are unique within an organization.';



ALTER TABLE ONLY "public"."map_study_partners"
    ADD CONSTRAINT "uq_partner_role_per_config" UNIQUE ("scenario_configuration_sk", "partner_organization_sk", "partner_role");



COMMENT ON CONSTRAINT "uq_partner_role_per_config" ON "public"."map_study_partners" IS 'UNIQUE constraint to prevent assigning the same organization the same role more than once within a single plan.';



ALTER TABLE ONLY "public"."dim_study"
    ADD CONSTRAINT "uq_protocol_per_org" UNIQUE ("organization_sk", "protocol_number");



COMMENT ON CONSTRAINT "uq_protocol_per_org" ON "public"."dim_study" IS 'UNIQUE constraint to ensure that protocol numbers are unique within an organization, acting as a natural key.';



ALTER TABLE ONLY "public"."dim_reimbursement_type"
    ADD CONSTRAINT "uq_reimbursement_type_name" UNIQUE ("reimbursement_type_name", "organization_sk");



COMMENT ON CONSTRAINT "uq_reimbursement_type_name" ON "public"."dim_reimbursement_type" IS 'UNIQUE constraint to ensure reimbursement type names are unique within an organization.';



ALTER TABLE ONLY "public"."dim_site"
    ADD CONSTRAINT "uq_site_bk_per_org" UNIQUE ("organization_sk", "site_bk");



COMMENT ON CONSTRAINT "uq_site_bk_per_org" ON "public"."dim_site" IS 'UNIQUE constraint to ensure site business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_site"
    ADD CONSTRAINT "uq_site_org_per_config" UNIQUE ("scenario_configuration_sk", "site_organization_sk");



COMMENT ON CONSTRAINT "uq_site_org_per_config" ON "public"."dim_site" IS 'UNIQUE constraint to prevent assigning the same site organization to the same plan more than once.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "uq_soa_mapping_per_config_sk" UNIQUE ("scenario_configuration_sk", "visit_sk", "activity_sk");



COMMENT ON CONSTRAINT "uq_soa_mapping_per_config_sk" ON "public"."map_study_visit_activity" IS 'UNIQUE constraint to prevent scheduling the same activity at the same visit more than once within a single plan.';



ALTER TABLE ONLY "public"."map_study_staff"
    ADD CONSTRAINT "uq_staff_role_per_partner" UNIQUE ("study_partner_sk", "user_sk", "staff_role");



COMMENT ON CONSTRAINT "uq_staff_role_per_partner" ON "public"."map_study_staff" IS 'UNIQUE constraint to prevent assigning the same user the same role for the same partner more than once.';



ALTER TABLE ONLY "public"."dim_amendment"
    ADD CONSTRAINT "uq_study_amendment_bk" UNIQUE ("study_sk", "amendment_bk");



COMMENT ON CONSTRAINT "uq_study_amendment_bk" ON "public"."dim_amendment" IS 'UNIQUE constraint to ensure amendment business keys are unique per study.';



ALTER TABLE ONLY "public"."dim_amendment"
    ADD CONSTRAINT "uq_study_amendment_version" UNIQUE ("study_sk", "amendment_version_id");



COMMENT ON CONSTRAINT "uq_study_amendment_version" ON "public"."dim_amendment" IS 'UNIQUE constraint to ensure amendment version IDs are unique per study.';



ALTER TABLE ONLY "public"."dim_study"
    ADD CONSTRAINT "uq_study_bk_per_org" UNIQUE ("organization_sk", "study_bk");



COMMENT ON CONSTRAINT "uq_study_bk_per_org" ON "public"."dim_study" IS 'UNIQUE constraint to ensure study business keys are unique within an organization.';



ALTER TABLE ONLY "public"."subscriptions"
    ADD CONSTRAINT "uq_subscriptions_organization_sk" UNIQUE ("organization_sk");



COMMENT ON CONSTRAINT "uq_subscriptions_organization_sk" ON "public"."subscriptions" IS 'UNIQUE constraint to ensure only one subscription record exists per organization, enforced on the surrogate key.';



ALTER TABLE ONLY "public"."dim_study"
    ADD CONSTRAINT "uq_title_per_org" UNIQUE ("organization_sk", "study_title");



COMMENT ON CONSTRAINT "uq_title_per_org" ON "public"."dim_study" IS 'UNIQUE constraint to ensure study titles are unique within an organization.';



ALTER TABLE ONLY "public"."dim_study_visits"
    ADD CONSTRAINT "uq_visit_bk" UNIQUE ("organization_sk", "visit_bk");



COMMENT ON CONSTRAINT "uq_visit_bk" ON "public"."dim_study_visits" IS 'UNIQUE constraint to ensure visit business keys are unique within an organization.';



ALTER TABLE ONLY "public"."dim_study_visits"
    ADD CONSTRAINT "uq_visit_name_per_epoch" UNIQUE ("epoch_sk", "visit_name");



COMMENT ON CONSTRAINT "uq_visit_name_per_epoch" ON "public"."dim_study_visits" IS 'UNIQUE constraint to ensure visit names are unique within a single epoch.';



ALTER TABLE ONLY "public"."dim_study_visits"
    ADD CONSTRAINT "uq_visit_order_per_epoch" UNIQUE ("epoch_sk", "visit_order");



COMMENT ON CONSTRAINT "uq_visit_order_per_epoch" ON "public"."dim_study_visits" IS 'UNIQUE constraint to ensure visit order is unique within a single epoch.';



ALTER TABLE ONLY "public"."user_organization_membership"
    ADD CONSTRAINT "user_organization_membership_pkey" PRIMARY KEY ("user_organization_membership_sk");



COMMENT ON CONSTRAINT "user_organization_membership_pkey" ON "public"."user_organization_membership" IS 'PRIMARY KEY constraint to uniquely identify each membership record.';



ALTER TABLE ONLY "public"."user_organization_membership"
    ADD CONSTRAINT "user_organization_membership_user_sk_organization_sk_key" UNIQUE ("user_sk", "organization_sk");



COMMENT ON CONSTRAINT "user_organization_membership_user_sk_organization_sk_key" ON "public"."user_organization_membership" IS 'UNIQUE constraint to ensure a user can only have one membership (and one role) per organization.';



CREATE INDEX "idx_materialized_payload_stage_number" ON "private"."materialized_template_payload" USING "btree" ("stage_number");



COMMENT ON INDEX "private"."idx_materialized_payload_stage_number" IS 'Accelerates lookups by the seeder orchestrator, which fetches payloads stage by stage to ensure correct insertion order.';



CREATE INDEX "idx_template_activities_include" ON "private"."template_activities" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_activities_reimb_type_bk" ON "private"."template_activities" USING "btree" ("reimbursement_type_bk");



CREATE INDEX "idx_template_amendments_include" ON "private"."template_amendments" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_amendments_parent_study_bk" ON "private"."template_amendments" USING "btree" ("parent_study_bk");



CREATE INDEX "idx_template_arms_include" ON "private"."template_study_arms" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_budget_cat_include" ON "private"."template_budget_categories" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_budget_cat_parent_bk" ON "private"."template_budget_categories" USING "btree" ("parent_category_bk");



CREATE INDEX "idx_template_configs_activity_bk" ON "private"."template_forecast_calculation_configs" USING "btree" ("activity_bk");



CREATE INDEX "idx_template_configs_include" ON "private"."template_forecast_calculation_configs" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_costs_activity_bk" ON "private"."template_activity_costs" USING "btree" ("activity_bk");



CREATE INDEX "idx_template_costs_include" ON "private"."template_activity_costs" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_costs_parent_scen_config_bk" ON "private"."template_activity_costs" USING "btree" ("parent_scenario_configuration_bk");



CREATE INDEX "idx_template_costs_reimb_type_bk" ON "private"."template_activity_costs" USING "btree" ("reimbursement_type_bk");



CREATE INDEX "idx_template_epochs_include" ON "private"."template_study_epochs" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_epochs_parent_arm_bk" ON "private"."template_study_epochs" USING "btree" ("parent_arm_bk");



CREATE INDEX "idx_template_memberships_include" ON "private"."template_memberships" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_memberships_org_bk" ON "private"."template_memberships" USING "btree" ("organization_bk");



CREATE INDEX "idx_template_memberships_user_bk" ON "private"."template_memberships" USING "btree" ("user_bk");



CREATE INDEX "idx_template_orgs_include" ON "private"."template_organizations" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_partners_include" ON "private"."template_study_partners" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_partners_org_bk" ON "private"."template_study_partners" USING "btree" ("partner_organization_bk");



CREATE INDEX "idx_template_reimb_types_include" ON "private"."template_reimbursement_types" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_scen_configs_include" ON "private"."template_scenario_configurations" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_scen_configs_scen_bk" ON "private"."template_scenario_configurations" USING "btree" ("parent_scenario_bk");



CREATE INDEX "idx_template_scen_configs_study_bk" ON "private"."template_scenario_configurations" USING "btree" ("parent_study_bk");



CREATE INDEX "idx_template_scen_sites_include" ON "private"."template_sites" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_scen_sites_perf_group_bk" ON "private"."template_sites" USING "btree" ("performance_group_config_bk");



CREATE INDEX "idx_template_scenarios_include" ON "private"."template_scenarios" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_soa_activity_bk" ON "private"."template_soa_mappings" USING "btree" ("activity_bk");



CREATE INDEX "idx_template_soa_include" ON "private"."template_soa_mappings" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_soa_parent_scen_config_bk" ON "private"."template_soa_mappings" USING "btree" ("parent_scenario_configuration_bk");



CREATE INDEX "idx_template_soa_visit_bk" ON "private"."template_soa_mappings" USING "btree" ("visit_bk");



CREATE INDEX "idx_template_studies_include" ON "private"."template_studies" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_users_include" ON "private"."template_users" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_visits_include" ON "private"."template_study_visits" USING "btree" ("include_in_json") WHERE ("include_in_json" = true);



CREATE INDEX "idx_template_visits_parent_epoch_bk" ON "private"."template_study_visits" USING "btree" ("parent_epoch_bk");



CREATE UNIQUE INDEX "uq_idx_private_template_metadata_dict_tbl_name" ON "private"."template_metadata_dictionary" USING "btree" ("template_table_name");



COMMENT ON INDEX "private"."uq_idx_private_template_metadata_dict_tbl_name" IS 'UNIQUE index on the materialized view to enforce the primary key constraint and ensure fast lookups by table name.';



CREATE INDEX "idx_checkpoint_writes_lookup" ON "public"."checkpoint_writes" USING "btree" ("thread_id", "checkpoint_ns", "checkpoint_id", "task_id", "organization_sk");



COMMENT ON INDEX "public"."idx_checkpoint_writes_lookup" IS 'Accelerates lookups for specific write operations within a task of a checkpoint, used by the agent framework.';



CREATE INDEX "idx_checkpoint_writes_thread_org" ON "public"."checkpoint_writes" USING "btree" ("thread_id", "checkpoint_id", "organization_sk");



COMMENT ON INDEX "public"."idx_checkpoint_writes_thread_org" IS 'Accelerates queries that retrieve all write operations for a given thread within an organization.';



CREATE INDEX "idx_checkpoints_lookup" ON "public"."checkpoints" USING "btree" ("thread_id", "checkpoint_ns", "checkpoint_id", "organization_sk");



COMMENT ON INDEX "public"."idx_checkpoints_lookup" IS 'Accelerates lookups for a specific checkpoint by its full composite key, essential for resuming workflows.';



CREATE INDEX "idx_checkpoints_thread_org" ON "public"."checkpoints" USING "btree" ("thread_id", "organization_sk", "checkpoint_ns");



COMMENT ON INDEX "public"."idx_checkpoints_thread_org" IS 'Accelerates queries that retrieve all checkpoints for a given thread within an organization, used for fetching history.';



CREATE INDEX "idx_da_created_by" ON "public"."dim_activity" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_da_created_by" IS 'Accelerates queries filtering activities by their creator.';



CREATE INDEX "idx_da_reimbursement_type_sk" ON "public"."dim_activity" USING "btree" ("reimbursement_type_sk");



COMMENT ON INDEX "public"."idx_da_reimbursement_type_sk" IS 'Accelerates lookups for activities linked to a specific default reimbursement policy.';



CREATE INDEX "idx_da_updated_by" ON "public"."dim_activity" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_da_updated_by" IS 'Accelerates queries filtering activities by their last updater.';



CREATE INDEX "idx_dac_activity_sk" ON "public"."dim_activity_cost" USING "btree" ("activity_sk");



COMMENT ON INDEX "public"."idx_dac_activity_sk" IS 'Accelerates lookups for all costs associated with a specific activity.';



CREATE INDEX "idx_dac_created_by" ON "public"."dim_activity_cost" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_dac_created_by" IS 'Accelerates queries filtering activity costs by their creator.';



CREATE INDEX "idx_dac_organization_sk" ON "public"."dim_activity_cost" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dac_organization_sk" IS 'The primary index for enforcing RLS on activity costs.';



CREATE INDEX "idx_dac_reimbursement_type_sk" ON "public"."dim_activity_cost" USING "btree" ("reimbursement_type_sk");



COMMENT ON INDEX "public"."idx_dac_reimbursement_type_sk" IS 'Accelerates lookups for all costs using a specific reimbursement policy.';



CREATE INDEX "idx_dac_updated_by" ON "public"."dim_activity_cost" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_dac_updated_by" IS 'Accelerates queries filtering activity costs by their last updater.';



CREATE INDEX "idx_dam_created_by" ON "public"."dim_amendment" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_dam_created_by" IS 'Accelerates queries filtering amendments by their creator.';



CREATE INDEX "idx_dam_updated_by" ON "public"."dim_amendment" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_dam_updated_by" IS 'Accelerates queries filtering amendments by their last updater.';



CREATE INDEX "idx_dbc_created_by" ON "public"."dim_budget_category" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_dbc_created_by" IS 'Accelerates queries filtering budget categories by their creator.';



CREATE INDEX "idx_dbc_organization_sk" ON "public"."dim_budget_category" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dbc_organization_sk" IS 'The primary index for enforcing RLS on budget categories.';



CREATE INDEX "idx_dbc_parent_category_sk" ON "public"."dim_budget_category" USING "btree" ("parent_category_sk");



COMMENT ON INDEX "public"."idx_dbc_parent_category_sk" IS 'Accelerates traversal of the budget category hierarchy.';



CREATE INDEX "idx_dbc_updated_by" ON "public"."dim_budget_category" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_dbc_updated_by" IS 'Accelerates queries filtering budget categories by their last updater.';



CREATE INDEX "idx_dbs_created_by" ON "public"."dim_budget_scenario" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_dbs_created_by" IS 'Accelerates queries filtering scenarios by their creator.';



CREATE INDEX "idx_dbs_triggering_amendment_sk" ON "public"."dim_budget_scenario" USING "btree" ("triggering_amendment_sk");



COMMENT ON INDEX "public"."idx_dbs_triggering_amendment_sk" IS 'Accelerates lookups for scenarios triggered by a specific amendment.';



CREATE INDEX "idx_dbs_updated_by" ON "public"."dim_budget_scenario" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_dbs_updated_by" IS 'Accelerates queries filtering scenarios by their last updater.';



CREATE INDEX "idx_dfcc_created_by" ON "public"."dim_forecast_calculation_config" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_dfcc_created_by" IS 'Accelerates queries filtering forecast rules by their creator.';



CREATE INDEX "idx_dfcc_updated_by" ON "public"."dim_forecast_calculation_config" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_dfcc_updated_by" IS 'Accelerates queries filtering forecast rules by their last updater.';



CREATE INDEX "idx_dim_activity_budget_category_sk" ON "public"."dim_activity" USING "btree" ("budget_category_sk");



COMMENT ON INDEX "public"."idx_dim_activity_budget_category_sk" IS 'Accelerates lookups for all activities belonging to a specific budget category.';



CREATE INDEX "idx_dim_activity_cost_by_config_and_activity" ON "public"."dim_activity_cost" USING "btree" ("scenario_configuration_sk", "activity_sk");



COMMENT ON INDEX "public"."idx_dim_activity_cost_by_config_and_activity" IS 'A composite index to accelerate lookups for the cost of a specific activity within a specific plan.';



CREATE INDEX "idx_dim_activity_cost_cost_bearing_partner_sk" ON "public"."dim_activity_cost" USING "btree" ("cost_bearing_partner_sk");



COMMENT ON INDEX "public"."idx_dim_activity_cost_cost_bearing_partner_sk" IS 'Accelerates queries filtering costs by the payee.';



CREATE INDEX "idx_dim_activity_cost_payer_partner_sk" ON "public"."dim_activity_cost" USING "btree" ("payer_partner_sk");



COMMENT ON INDEX "public"."idx_dim_activity_cost_payer_partner_sk" IS 'Accelerates queries filtering costs by the payer.';



CREATE INDEX "idx_dim_activity_cost_scenario_configuration_sk" ON "public"."dim_activity_cost" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_dim_activity_cost_scenario_configuration_sk" IS 'Accelerates lookups for all costs associated with a specific plan.';



CREATE INDEX "idx_dim_activity_template_finder" ON "public"."dim_activity" USING "btree" ("organization_sk", "activity_bk") WHERE ("activity_bk" ~~ 'TPL-%'::"text");



COMMENT ON INDEX "public"."idx_dim_activity_template_finder" IS 'A partial index to quickly find all template activities within an organization''s sandbox for adoption or clearing.';



CREATE INDEX "idx_dim_amendment_organization_sk" ON "public"."dim_amendment" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_amendment_organization_sk" IS 'The primary index for enforcing RLS on amendments.';



CREATE INDEX "idx_dim_amendment_study_sk" ON "public"."dim_amendment" USING "btree" ("study_sk");



COMMENT ON INDEX "public"."idx_dim_amendment_study_sk" IS 'Accelerates lookups for all amendments belonging to a specific study.';



CREATE INDEX "idx_dim_budget_category_template_finder" ON "public"."dim_budget_category" USING "btree" ("organization_sk", "budget_category_bk") WHERE ("budget_category_bk" ~~ 'TPL-%'::"text");



COMMENT ON INDEX "public"."idx_dim_budget_category_template_finder" IS 'A partial index to quickly find all template budget categories within an organization''s sandbox.';



CREATE INDEX "idx_dim_budget_scenario_organization_sk" ON "public"."dim_budget_scenario" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_budget_scenario_organization_sk" IS 'The primary index for enforcing RLS on scenarios.';



CREATE INDEX "idx_dim_forecast_calculation_config_activity_sk" ON "public"."dim_forecast_calculation_config" USING "btree" ("activity_sk");



COMMENT ON INDEX "public"."idx_dim_forecast_calculation_config_activity_sk" IS 'Accelerates lookups for rules associated with a specific activity.';



CREATE INDEX "idx_dim_forecast_calculation_config_organization_sk" ON "public"."dim_forecast_calculation_config" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_forecast_calculation_config_organization_sk" IS 'The primary index for enforcing RLS on forecast rules.';



CREATE INDEX "idx_dim_forecast_calculation_config_scenario_config_sk" ON "public"."dim_forecast_calculation_config" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_dim_forecast_calculation_config_scenario_config_sk" IS 'Accelerates lookups for all rules associated with a specific plan.';



CREATE INDEX "idx_dim_organization_template_finder" ON "public"."dim_organization" USING "btree" ("parent_organization_sk", "organization_bk") WHERE ("organization_bk" ~~ 'TPL-%'::"text");



COMMENT ON INDEX "public"."idx_dim_organization_template_finder" IS 'A partial index to quickly find all template organizations within an organization''s sandbox.';



CREATE INDEX "idx_dim_reimbursement_type_template_finder" ON "public"."dim_reimbursement_type" USING "btree" ("organization_sk", "reimbursement_type_bk") WHERE ("reimbursement_type_bk" ~~ 'TPL-%'::"text");



COMMENT ON INDEX "public"."idx_dim_reimbursement_type_template_finder" IS 'A partial index to quickly find all template reimbursement types within an organization''s sandbox.';



CREATE INDEX "idx_dim_site_by_config_and_org" ON "public"."dim_site" USING "btree" ("scenario_configuration_sk", "site_organization_sk");



COMMENT ON INDEX "public"."idx_dim_site_by_config_and_org" IS 'A composite index to accelerate lookups for a specific site organization within a specific plan.';



CREATE INDEX "idx_dim_site_organization_sk" ON "public"."dim_site" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_site_organization_sk" IS 'The primary index for enforcing RLS on site assignments.';



CREATE INDEX "idx_dim_site_scenario_configuration_sk" ON "public"."dim_site" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_dim_site_scenario_configuration_sk" IS 'Accelerates lookups for all sites assigned to a specific plan.';



CREATE INDEX "idx_dim_site_site_organization_sk" ON "public"."dim_site" USING "btree" ("site_organization_sk");



COMMENT ON INDEX "public"."idx_dim_site_site_organization_sk" IS 'Accelerates lookups for all assignments of a specific site organization across all plans.';



CREATE INDEX "idx_dim_site_site_partner_sk" ON "public"."dim_site" USING "btree" ("site_partner_sk");



COMMENT ON INDEX "public"."idx_dim_site_site_partner_sk" IS 'Accelerates lookups on the site-to-partner link.';



CREATE INDEX "idx_dim_study_arm_organization_sk" ON "public"."dim_study_arm" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_study_arm_organization_sk" IS 'The primary index for enforcing RLS on study arms.';



CREATE INDEX "idx_dim_study_arm_scenario_configuration_sk" ON "public"."dim_study_arm" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_dim_study_arm_scenario_configuration_sk" IS 'Accelerates lookups for all arms belonging to a specific plan.';



CREATE INDEX "idx_dim_study_epochs_arm_sk" ON "public"."dim_study_epochs" USING "btree" ("arm_sk");



COMMENT ON INDEX "public"."idx_dim_study_epochs_arm_sk" IS 'Accelerates lookups for all epochs belonging to a specific arm.';



CREATE INDEX "idx_dim_study_epochs_organization_sk" ON "public"."dim_study_epochs" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_study_epochs_organization_sk" IS 'The primary index for enforcing RLS on epochs.';



CREATE INDEX "idx_dim_study_organization_sk" ON "public"."dim_study" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_study_organization_sk" IS 'The primary index for enforcing RLS on studies.';



CREATE INDEX "idx_dim_study_template_finder" ON "public"."dim_study" USING "btree" ("organization_sk", "study_bk") WHERE ("study_bk" ~~ 'TPL-%'::"text");



COMMENT ON INDEX "public"."idx_dim_study_template_finder" IS 'A partial index to quickly find all template studies within an organization''s sandbox.';



CREATE INDEX "idx_dim_study_visits_epoch_sk" ON "public"."dim_study_visits" USING "btree" ("epoch_sk");



COMMENT ON INDEX "public"."idx_dim_study_visits_epoch_sk" IS 'Accelerates lookups for all visits belonging to a specific epoch.';



CREATE INDEX "idx_dim_study_visits_organization_sk" ON "public"."dim_study_visits" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_dim_study_visits_organization_sk" IS 'The primary index for enforcing RLS on visits.';



CREATE INDEX "idx_dim_user_template_finder" ON "public"."dim_user" USING "btree" ("organization_sk", "user_bk") WHERE ("user_bk" ~~ 'TPL-%'::"text");



COMMENT ON INDEX "public"."idx_dim_user_template_finder" IS 'A partial index to quickly find all template users within an organization''s sandbox.';



CREATE INDEX "idx_ds_created_by" ON "public"."dim_study" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_ds_created_by" IS 'Accelerates queries filtering studies by their creator.';



CREATE INDEX "idx_ds_updated_by" ON "public"."dim_study" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_ds_updated_by" IS 'Accelerates queries filtering studies by their last updater.';



CREATE INDEX "idx_fact_enrollment_analytics_by_plan_time" ON "public"."fact_enrollment" USING "btree" ("scenario_configuration_sk", "snapshot_date_sk");



COMMENT ON INDEX "public"."idx_fact_enrollment_analytics_by_plan_time" IS 'A composite index to accelerate time-series analysis of enrollment data for a specific plan.';



CREATE INDEX "idx_fact_enrollment_bk_lookup" ON "public"."fact_enrollment" USING "btree" ("enrollment_bk", "organization_sk");



COMMENT ON INDEX "public"."idx_fact_enrollment_bk_lookup" IS 'Accelerates lookups for enrollment facts by their business key.';



CREATE INDEX "idx_fact_enrollment_organization_sk" ON "public"."fact_enrollment" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_fact_enrollment_organization_sk" IS 'The primary index for enforcing RLS on enrollment facts.';



CREATE INDEX "idx_fact_enrollment_scenario_configuration_sk" ON "public"."fact_enrollment" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_fact_enrollment_scenario_configuration_sk" IS 'Accelerates queries filtering enrollment facts by plan.';



CREATE INDEX "idx_fact_enrollment_scenario_sk" ON "public"."fact_enrollment" USING "btree" ("scenario_sk");



COMMENT ON INDEX "public"."idx_fact_enrollment_scenario_sk" IS 'Accelerates queries filtering enrollment facts by scenario.';



CREATE INDEX "idx_fact_enrollment_snapshot_date_sk" ON "public"."fact_enrollment" USING "btree" ("snapshot_date_sk");



COMMENT ON INDEX "public"."idx_fact_enrollment_snapshot_date_sk" IS 'The primary index for time-series queries on enrollment data.';



CREATE INDEX "idx_fact_enrollment_study_sk" ON "public"."fact_enrollment" USING "btree" ("study_sk");



COMMENT ON INDEX "public"."idx_fact_enrollment_study_sk" IS 'Accelerates queries filtering enrollment facts by study.';



CREATE INDEX "idx_fact_forecast_detail_analytics_by_plan_activity_time" ON "public"."fact_forecast_detail" USING "btree" ("scenario_sk", "activity_sk", "snapshot_date_sk");



COMMENT ON INDEX "public"."idx_fact_forecast_detail_analytics_by_plan_activity_time" IS 'A composite index to accelerate time-series analysis of costs for a specific activity within a specific scenario.';



CREATE INDEX "idx_fact_forecast_detail_analytics_by_scenario_payee_activity" ON "public"."fact_forecast_detail" USING "btree" ("scenario_sk", "payee_partner_sk", "activity_sk") INCLUDE ("net_cost", "forecast_units");



COMMENT ON INDEX "public"."idx_fact_forecast_detail_analytics_by_scenario_payee_activity" IS 'A composite index to accelerate financial rollups by payee and activity for a specific scenario.';



CREATE INDEX "idx_fact_forecast_detail_payee_partner_sk" ON "public"."fact_forecast_detail" USING "btree" ("payee_partner_sk");



COMMENT ON INDEX "public"."idx_fact_forecast_detail_payee_partner_sk" IS 'Accelerates queries filtering forecast details by the payee.';



CREATE INDEX "idx_fact_forecast_detail_payer_partner_sk" ON "public"."fact_forecast_detail" USING "btree" ("payer_partner_sk");



COMMENT ON INDEX "public"."idx_fact_forecast_detail_payer_partner_sk" IS 'Accelerates queries filtering forecast details by the payer.';



CREATE INDEX "idx_fact_forecast_detail_scenario_configuration_sk" ON "public"."fact_forecast_detail" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_fact_forecast_detail_scenario_configuration_sk" IS 'Accelerates queries filtering forecast details by plan.';



CREATE INDEX "idx_fact_forecast_detail_scenario_sk" ON "public"."fact_forecast_detail" USING "btree" ("scenario_sk");



COMMENT ON INDEX "public"."idx_fact_forecast_detail_scenario_sk" IS 'Accelerates queries filtering forecast details by scenario.';



CREATE INDEX "idx_fact_forecast_detail_study_sk" ON "public"."fact_forecast_detail" USING "btree" ("study_sk");



COMMENT ON INDEX "public"."idx_fact_forecast_detail_study_sk" IS 'Accelerates queries filtering forecast details by study.';



CREATE INDEX "idx_fe_site_sk" ON "public"."fact_enrollment" USING "btree" ("site_sk");



COMMENT ON INDEX "public"."idx_fe_site_sk" IS 'Accelerates queries filtering enrollment facts by site.';



CREATE INDEX "idx_ffd_activity_sk" ON "public"."fact_forecast_detail" USING "btree" ("activity_sk");



COMMENT ON INDEX "public"."idx_ffd_activity_sk" IS 'Accelerates queries filtering forecast details by activity.';



CREATE INDEX "idx_ffd_created_by" ON "public"."fact_forecast_detail" USING "btree" ("created_by_user_sk");



COMMENT ON INDEX "public"."idx_ffd_created_by" IS 'Accelerates queries filtering forecast details by their creator.';



CREATE INDEX "idx_ffd_organization_sk" ON "public"."fact_forecast_detail" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_ffd_organization_sk" IS 'The primary index for enforcing RLS on forecast details.';



CREATE INDEX "idx_ffd_site_sk" ON "public"."fact_forecast_detail" USING "btree" ("site_sk");



COMMENT ON INDEX "public"."idx_ffd_site_sk" IS 'Accelerates queries filtering forecast details by site.';



CREATE INDEX "idx_ffd_snapshot_date_sk" ON "public"."fact_forecast_detail" USING "btree" ("snapshot_date_sk");



COMMENT ON INDEX "public"."idx_ffd_snapshot_date_sk" IS 'The primary index for time-series queries on forecast data.';



CREATE INDEX "idx_ffd_source_activity_cost_sk" ON "public"."fact_forecast_detail" USING "btree" ("source_activity_cost_sk");



COMMENT ON INDEX "public"."idx_ffd_source_activity_cost_sk" IS 'Accelerates lineage tracing from a forecast fact back to its source cost record.';



CREATE INDEX "idx_ffd_source_arm_sk" ON "public"."fact_forecast_detail" USING "btree" ("source_arm_sk");



COMMENT ON INDEX "public"."idx_ffd_source_arm_sk" IS 'Accelerates lineage tracing from a forecast fact back to its source arm.';



CREATE INDEX "idx_ffd_source_enrollment_pk" ON "public"."fact_forecast_detail" USING "btree" ("source_enrollment_pk");



COMMENT ON INDEX "public"."idx_ffd_source_enrollment_pk" IS 'Accelerates lineage tracing from a forecast fact back to its source enrollment record.';



CREATE INDEX "idx_ffd_source_epoch_sk" ON "public"."fact_forecast_detail" USING "btree" ("source_epoch_sk");



COMMENT ON INDEX "public"."idx_ffd_source_epoch_sk" IS 'Accelerates lineage tracing from a forecast fact back to its source epoch.';



CREATE INDEX "idx_ffd_source_forecast_config_sk" ON "public"."fact_forecast_detail" USING "btree" ("source_forecast_config_sk");



COMMENT ON INDEX "public"."idx_ffd_source_forecast_config_sk" IS 'Accelerates lineage tracing from a forecast fact back to its source forecast rule.';



CREATE INDEX "idx_ffd_source_map_soa_sk" ON "public"."fact_forecast_detail" USING "btree" ("source_map_soa_sk");



COMMENT ON INDEX "public"."idx_ffd_source_map_soa_sk" IS 'Accelerates lineage tracing from a forecast fact back to its source SoA mapping.';



CREATE INDEX "idx_ffd_source_visit_sk" ON "public"."fact_forecast_detail" USING "btree" ("source_visit_sk");



COMMENT ON INDEX "public"."idx_ffd_source_visit_sk" IS 'Accelerates lineage tracing from a forecast fact back to its source visit.';



CREATE INDEX "idx_ffd_updated_by" ON "public"."fact_forecast_detail" USING "btree" ("updated_by_user_sk");



COMMENT ON INDEX "public"."idx_ffd_updated_by" IS 'Accelerates queries filtering forecast details by their last updater.';



CREATE INDEX "idx_futu_org_created" ON "public"."fact_user_token_usage" USING "btree" ("organization_sk", "created_at" DESC);



CREATE INDEX "idx_futu_org_user_created" ON "public"."fact_user_token_usage" USING "btree" ("organization_sk", "user_sk", "created_at" DESC);



CREATE INDEX "idx_map_config_scenario_sk" ON "public"."map_scenario_configuration" USING "btree" ("scenario_sk");



COMMENT ON INDEX "public"."idx_map_config_scenario_sk" IS 'Accelerates lookups for all plans associated with a specific scenario.';



CREATE INDEX "idx_map_partner_org_sk" ON "public"."map_study_partners" USING "btree" ("partner_organization_sk");



COMMENT ON INDEX "public"."idx_map_partner_org_sk" IS 'Accelerates lookups for all partner assignments for a specific organization.';



CREATE INDEX "idx_map_scenario_configuration_organization_sk" ON "public"."map_scenario_configuration" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_map_scenario_configuration_organization_sk" IS 'The primary index for enforcing RLS on scenario configurations.';



CREATE INDEX "idx_map_scenario_configuration_study_sk" ON "public"."map_scenario_configuration" USING "btree" ("study_sk");



COMMENT ON INDEX "public"."idx_map_scenario_configuration_study_sk" IS 'Accelerates lookups for all plans associated with a specific study.';



CREATE INDEX "idx_map_study_partners_organization_sk" ON "public"."map_study_partners" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_map_study_partners_organization_sk" IS 'The primary index for enforcing RLS on partner assignments.';



CREATE INDEX "idx_map_study_partners_scenario_configuration_sk" ON "public"."map_study_partners" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_map_study_partners_scenario_configuration_sk" IS 'Accelerates lookups for all partners assigned to a specific plan.';



CREATE INDEX "idx_map_study_staff_organization_sk" ON "public"."map_study_staff" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_map_study_staff_organization_sk" IS 'The primary index for enforcing RLS on staff assignments.';



CREATE INDEX "idx_map_study_staff_study_partner_sk" ON "public"."map_study_staff" USING "btree" ("study_partner_sk");



COMMENT ON INDEX "public"."idx_map_study_staff_study_partner_sk" IS 'Accelerates lookups for all staff assigned to a specific partner.';



CREATE INDEX "idx_map_study_staff_user_sk" ON "public"."map_study_staff" USING "btree" ("user_sk");



COMMENT ON INDEX "public"."idx_map_study_staff_user_sk" IS 'Accelerates lookups for all assignments for a specific user.';



CREATE INDEX "idx_map_study_visit_activity_organization_sk" ON "public"."map_study_visit_activity" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_map_study_visit_activity_organization_sk" IS 'The primary index for enforcing RLS on SoA mappings.';



CREATE INDEX "idx_map_study_visit_activity_scenario_configuration_sk" ON "public"."map_study_visit_activity" USING "btree" ("scenario_configuration_sk");



COMMENT ON INDEX "public"."idx_map_study_visit_activity_scenario_configuration_sk" IS 'Accelerates lookups for all SoA mappings within a specific plan.';



CREATE INDEX "idx_map_study_visit_activity_visit_sk" ON "public"."map_study_visit_activity" USING "btree" ("visit_sk");



COMMENT ON INDEX "public"."idx_map_study_visit_activity_visit_sk" IS 'Accelerates lookups for all activities scheduled for a specific visit.';



CREATE INDEX "idx_msva_activity_sk" ON "public"."map_study_visit_activity" USING "btree" ("activity_sk");



COMMENT ON INDEX "public"."idx_msva_activity_sk" IS 'Accelerates lookups for all visits where a specific activity is scheduled.';



CREATE INDEX "idx_org_parent_sk" ON "public"."dim_organization" USING "btree" ("parent_organization_sk");



COMMENT ON INDEX "public"."idx_org_parent_sk" IS 'Accelerates traversal of the organization hierarchy.';



CREATE INDEX "idx_org_status" ON "public"."dim_organization" USING "btree" ("org_status");



COMMENT ON INDEX "public"."idx_org_status" IS 'Accelerates queries filtering organizations by their status.';



CREATE INDEX "idx_org_type" ON "public"."dim_organization" USING "btree" ("organization_type");



COMMENT ON INDEX "public"."idx_org_type" IS 'Accelerates queries filtering organizations by their type.';



CREATE INDEX "idx_synced_clerk_invitations_org_bk" ON "public"."synced_clerk_invitations" USING "btree" ("organization_bk");



COMMENT ON INDEX "public"."idx_synced_clerk_invitations_org_bk" IS 'Accelerates lookups for all pending invitations for a specific organization.';



CREATE INDEX "idx_threads_organization_sk" ON "public"."threads" USING "btree" ("organization_sk");



COMMENT ON INDEX "public"."idx_threads_organization_sk" IS 'The primary index for enforcing RLS on agent conversation threads.';



CREATE UNIQUE INDEX "uq_activity_cost_definitive_key" ON "public"."dim_activity_cost" USING "btree" ("scenario_configuration_sk", "activity_sk", "cost_bearing_partner_sk", "effective_date", COALESCE("payer_partner_sk", ('-1'::integer)::bigint));



COMMENT ON INDEX "public"."uq_activity_cost_definitive_key" IS 'UNIQUE index defining the natural key for a cost: a unique price for an activity, for a specific payee, within a specific plan, for a given effective date. This is critical for preventing ambiguous pricing.';



CREATE UNIQUE INDEX "uq_clerk_managed_organization_bk" ON "public"."dim_organization" USING "btree" ("organization_bk") WHERE ("is_clerk_managed" = true);



COMMENT ON INDEX "public"."uq_clerk_managed_organization_bk" IS 'UNIQUE index to enforce that business keys for Clerk-managed (root) organizations are globally unique. This is a critical rule for tenant identification.';



CREATE UNIQUE INDEX "uq_clerk_managed_user_bk" ON "public"."dim_user" USING "btree" ("user_bk") WHERE ("is_clerk_managed" = true);



COMMENT ON INDEX "public"."uq_clerk_managed_user_bk" IS 'UNIQUE index to enforce that business keys for Clerk-managed (root) users are globally unique. This is a critical rule for identity management.';



CREATE UNIQUE INDEX "uq_idx_dim_organization_clerk_managed_name" ON "public"."dim_organization" USING "btree" ("organization_name") WHERE ("is_clerk_managed" = true);



COMMENT ON INDEX "public"."uq_idx_dim_organization_clerk_managed_name" IS 'UNIQUE index to enforce that names for Clerk-managed (root) organizations are globally unique, preventing ambiguity between tenants.';



CREATE UNIQUE INDEX "uq_one_default_reimbursement_type_per_org" ON "public"."dim_reimbursement_type" USING "btree" ("organization_sk") WHERE (("is_default" = true) AND ("is_deleted" = false));



COMMENT ON INDEX "public"."uq_one_default_reimbursement_type_per_org" IS 'UNIQUE index to enforce the business rule that only one active reimbursement type can be marked as the default per organization.';



CREATE OR REPLACE TRIGGER "trg_cascade_from_activity" AFTER UPDATE ON "private"."template_activities" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_activity"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_arm" AFTER UPDATE ON "private"."template_study_arms" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_arm"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_budget_category" AFTER UPDATE ON "private"."template_budget_categories" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_budget_category"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_epoch" AFTER UPDATE ON "private"."template_study_epochs" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_epoch"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_organization" AFTER UPDATE ON "private"."template_organizations" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_organization"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_partner" AFTER UPDATE ON "private"."template_study_partners" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_partner"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_reimbursement_type" AFTER UPDATE ON "private"."template_reimbursement_types" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_reimbursement_type"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_scenario" AFTER UPDATE ON "private"."template_scenarios" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_scenario"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_scenario_config" AFTER UPDATE ON "private"."template_scenario_configurations" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_scenario_config"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_study" AFTER UPDATE ON "private"."template_studies" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_study"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_user" AFTER UPDATE ON "private"."template_users" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_user_to_membership"();



CREATE OR REPLACE TRIGGER "trg_cascade_from_visit" AFTER UPDATE ON "private"."template_study_visits" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_from_visit"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_arm" AFTER UPDATE ON "private"."template_study_arms" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_arm"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_budget_category" AFTER UPDATE ON "private"."template_budget_categories" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_budget_category"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_cost" AFTER UPDATE ON "private"."template_activity_costs" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_cost"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_epoch" AFTER UPDATE ON "private"."template_study_epochs" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_epoch"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_partner" AFTER UPDATE ON "private"."template_study_partners" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_partner"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_scenario_config" AFTER UPDATE ON "private"."template_scenario_configurations" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_scenario_config"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_site" AFTER UPDATE ON "private"."template_sites" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_site"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_soa" AFTER UPDATE ON "private"."template_soa_mappings" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_soa"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_study_staff" AFTER UPDATE ON "private"."template_study_staff" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_study_staff"();



CREATE OR REPLACE TRIGGER "trg_cascade_up_from_visit" AFTER UPDATE ON "private"."template_study_visits" FOR EACH ROW WHEN (("old"."include_in_json" IS DISTINCT FROM "new"."include_in_json")) EXECUTE FUNCTION "private"."trg_fn_cascade_up_from_visit"();



CREATE OR REPLACE TRIGGER "trg_set_initial_membership_state" BEFORE INSERT ON "private"."template_memberships" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_set_initial_membership_inclusion"();



CREATE OR REPLACE TRIGGER "trg_set_stage_number" BEFORE INSERT ON "private"."materialized_template_payload" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_set_payload_stage_number"();



CREATE OR REPLACE TRIGGER "trg_sync_cost_state" BEFORE INSERT ON "private"."template_activity_costs" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_cost_inclusion_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_forecast_calculation_configs" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_config_child_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_scenario_configurations" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_scenario_config_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_sites" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_config_child_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_soa_mappings" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_soa_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_study_arms" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_config_child_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_study_epochs" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_arm_child_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_study_partners" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_config_child_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_study_staff" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_study_staff_state"();



CREATE OR REPLACE TRIGGER "trg_sync_state_on_insert" BEFORE INSERT ON "private"."template_study_visits" FOR EACH ROW EXECUTE FUNCTION "private"."trg_fn_sync_epoch_child_state"();



CREATE OR REPLACE TRIGGER "bupd_dim_activity_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_activity" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_budget_category_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_budget_category" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_budget_scenario_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_budget_scenario" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_forecast_calc_cfg_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_forecast_calculation_config" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_organization_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_organization" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_reimb_type_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_reimbursement_type" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_study_arm_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_study_arm" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_study_epochs_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_study_epochs" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_study_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_study" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_study_visits_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_study_visits" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "bupd_dim_user_strip_tpl_on_adopt" BEFORE UPDATE ON "public"."dim_user" FOR EACH ROW EXECUTE FUNCTION "public"."trg_strip_tpl_prefix_on_adopt"();



CREATE OR REPLACE TRIGGER "trg_c_update_fact_forecast_detail_timestamp" BEFORE INSERT OR UPDATE ON "public"."fact_forecast_detail" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_c_update_fact_forecast_detail_timestamp" ON "public"."fact_forecast_detail" IS 'Automatically updates the updated_at timestamp on any modification to a forecast detail record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_activity" AFTER UPDATE ON "public"."dim_activity" FOR EACH ROW WHEN (("old"."activity_bk" IS DISTINCT FROM "new"."activity_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_activity_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_activity" ON "public"."dim_activity" IS 'Cascades updates of an activity''s business key to the denormalized `activity_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_activity_cost" AFTER UPDATE ON "public"."dim_activity_cost" FOR EACH ROW WHEN (("old"."activity_cost_bk" IS DISTINCT FROM "new"."activity_cost_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_activity_cost_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_activity_cost" ON "public"."dim_activity_cost" IS 'Cascades updates of an activity cost''s business key to the denormalized `source_activity_cost_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_budget_category" AFTER UPDATE ON "public"."dim_budget_category" FOR EACH ROW WHEN (("old"."budget_category_bk" IS DISTINCT FROM "new"."budget_category_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_budget_category_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_budget_category" ON "public"."dim_budget_category" IS 'Cascades updates of a budget category''s business key to the denormalized `parent_category_bk` column for all its children.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_budget_scenario" AFTER UPDATE ON "public"."dim_budget_scenario" FOR EACH ROW WHEN (("old"."scenario_bk" IS DISTINCT FROM "new"."scenario_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_scenario_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_budget_scenario" ON "public"."dim_budget_scenario" IS 'Cascades updates of a scenario''s business key to the denormalized `scenario_bk` column in fact tables.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_forecast_calculation_config" AFTER UPDATE ON "public"."dim_forecast_calculation_config" FOR EACH ROW WHEN (("old"."forecast_config_bk" IS DISTINCT FROM "new"."forecast_config_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_forecast_config_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_forecast_calculation_config" ON "public"."dim_forecast_calculation_config" IS 'Cascades updates of a forecast rule''s business key to the denormalized `source_forecast_config_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_site" AFTER UPDATE ON "public"."dim_site" FOR EACH ROW WHEN (("old"."site_bk" IS DISTINCT FROM "new"."site_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_site_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_site" ON "public"."dim_site" IS 'Cascades updates of a site''s business key to the denormalized `site_bk` column in fact tables.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_study" AFTER UPDATE ON "public"."dim_study" FOR EACH ROW WHEN (("old"."study_bk" IS DISTINCT FROM "new"."study_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_study_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_study" ON "public"."dim_study" IS 'Cascades updates of a study''s business key to the denormalized `study_bk` column in fact tables.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_study_arm" AFTER UPDATE ON "public"."dim_study_arm" FOR EACH ROW WHEN (("old"."arm_bk" IS DISTINCT FROM "new"."arm_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_arm_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_study_arm" ON "public"."dim_study_arm" IS 'Cascades updates of a study arm''s business key to the denormalized `source_arm_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_study_epochs" AFTER UPDATE ON "public"."dim_study_epochs" FOR EACH ROW WHEN (("old"."epoch_bk" IS DISTINCT FROM "new"."epoch_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_epoch_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_study_epochs" ON "public"."dim_study_epochs" IS 'Cascades updates of an epoch''s business key to the denormalized `source_epoch_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_dim_study_visits" AFTER UPDATE ON "public"."dim_study_visits" FOR EACH ROW WHEN (("old"."visit_bk" IS DISTINCT FROM "new"."visit_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_visit_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_dim_study_visits" ON "public"."dim_study_visits" IS 'Cascades updates of a visit''s business key to the denormalized `source_visit_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_fact_enrollment" AFTER UPDATE ON "public"."fact_enrollment" FOR EACH ROW WHEN (("old"."enrollment_bk" IS DISTINCT FROM "new"."enrollment_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_enrollment_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_fact_enrollment" ON "public"."fact_enrollment" IS 'Cascades updates of an enrollment fact''s business key to the denormalized `source_enrollment_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_map_scenario_configuration" AFTER UPDATE ON "public"."map_scenario_configuration" FOR EACH ROW WHEN (("old"."scenario_configuration_bk" IS DISTINCT FROM "new"."scenario_configuration_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_scen_config_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_map_scenario_configuration" ON "public"."map_scenario_configuration" IS 'Cascades updates of a scenario configuration''s business key to the denormalized `scenario_configuration_bk` column in fact tables.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_map_soa" AFTER UPDATE ON "public"."map_study_visit_activity" FOR EACH ROW WHEN (("old"."map_soa_bk" IS DISTINCT FROM "new"."map_soa_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_soa_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_map_soa" ON "public"."map_study_visit_activity" IS 'Cascades updates of a SoA mapping''s business key to the denormalized `source_map_soa_bk` column in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_on_map_study_partners" AFTER UPDATE ON "public"."map_study_partners" FOR EACH ROW WHEN (("old"."study_partner_bk" IS DISTINCT FROM "new"."study_partner_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_study_partner_bk_update"();



COMMENT ON TRIGGER "trg_cascade_bk_update_on_map_study_partners" ON "public"."map_study_partners" IS 'Cascades updates of a study partner''s business key to the denormalized `payer_partner_bk` and `payee_partner_bk` columns in `fact_forecast_detail`.';



CREATE OR REPLACE TRIGGER "trg_cascade_bk_update_to_subscriptions" AFTER UPDATE OF "organization_bk" ON "public"."dim_organization" FOR EACH ROW WHEN (("old"."organization_bk" IS DISTINCT FROM "new"."organization_bk")) EXECUTE FUNCTION "public"."trg_fn_cascade_org_bk_update_to_subscriptions"();



COMMENT ON TRIGGER "trg_cascade_bk_update_to_subscriptions" ON "public"."dim_organization" IS 'Cascades updates of an organization''s business key to the corresponding record in the subscriptions table.';



CREATE OR REPLACE TRIGGER "trg_hydrate_enrollment_fact" BEFORE INSERT OR UPDATE ON "public"."fact_enrollment" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_hydrate_enrollment_fact"();



COMMENT ON TRIGGER "trg_hydrate_enrollment_fact" ON "public"."fact_enrollment" IS 'Ensures all foreign keys and ownership keys are correctly populated on an enrollment fact before insert or update.';



CREATE OR REPLACE TRIGGER "trg_hydrate_forecast_detail" BEFORE INSERT OR UPDATE ON "public"."fact_forecast_detail" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_hydrate_forecast_detail"();



COMMENT ON TRIGGER "trg_hydrate_forecast_detail" ON "public"."fact_forecast_detail" IS 'Ensures all foreign keys and ownership keys are correctly populated on a forecast detail fact before insert or update.';



CREATE OR REPLACE TRIGGER "trg_hydrate_parent_sk_on_insert" BEFORE INSERT ON "public"."dim_budget_category" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_hydrate_budget_category_parent"();



COMMENT ON TRIGGER "trg_hydrate_parent_sk_on_insert" ON "public"."dim_budget_category" IS 'On insert, resolves the `parent_category_sk` from the provided `parent_category_bk` to enable hierarchical creation via business keys.';



CREATE OR REPLACE TRIGGER "trg_sync_org_keys" BEFORE INSERT OR UPDATE ON "public"."subscriptions" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_subscription_org_keys"();



COMMENT ON TRIGGER "trg_sync_org_keys" ON "public"."subscriptions" IS 'Ensures organization_sk and organization_bk are synchronized before a subscription record is written.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_for_activity_cost" BEFORE INSERT OR UPDATE ON "public"."dim_activity_cost" FOR EACH ROW EXECUTE FUNCTION "public"."sync_ownership_for_activity_cost"();



COMMENT ON TRIGGER "trg_sync_ownership_for_activity_cost" ON "public"."dim_activity_cost" IS 'Ensures an activity cost record always inherits the `organization_sk` from its parent scenario configuration.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_for_soa" BEFORE INSERT OR UPDATE ON "public"."map_study_visit_activity" FOR EACH ROW EXECUTE FUNCTION "public"."sync_ownership_for_soa_mapping"();



COMMENT ON TRIGGER "trg_sync_ownership_for_soa" ON "public"."map_study_visit_activity" IS 'Ensures a Schedule of Activities (SoA) mapping always inherits the `organization_sk` from its parent scenario configuration.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_arm" BEFORE INSERT OR UPDATE ON "public"."dim_study_epochs" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_ownership_from_arm"();



COMMENT ON TRIGGER "trg_sync_ownership_from_arm" ON "public"."dim_study_epochs" IS 'Ensures a study epoch record always inherits the `organization_sk` from its parent study arm.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_epoch" BEFORE INSERT OR UPDATE ON "public"."dim_study_visits" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_ownership_from_epoch"();



COMMENT ON TRIGGER "trg_sync_ownership_from_epoch" ON "public"."dim_study_visits" IS 'Ensures a study visit record always inherits the `organization_sk` from its parent study epoch.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_scenario_config" BEFORE INSERT OR UPDATE ON "public"."dim_forecast_calculation_config" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_ownership_from_scenario_config"();



COMMENT ON TRIGGER "trg_sync_ownership_from_scenario_config" ON "public"."dim_forecast_calculation_config" IS 'Ensures a forecast rule record always inherits the `organization_sk` from its parent scenario configuration.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_scenario_config" BEFORE INSERT OR UPDATE ON "public"."dim_site" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_ownership_from_scenario_config"();



COMMENT ON TRIGGER "trg_sync_ownership_from_scenario_config" ON "public"."dim_site" IS 'Ensures a site assignment record always inherits the `organization_sk` from its parent scenario configuration.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_scenario_config" BEFORE INSERT OR UPDATE ON "public"."dim_study_arm" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_ownership_from_scenario_config"();



COMMENT ON TRIGGER "trg_sync_ownership_from_scenario_config" ON "public"."dim_study_arm" IS 'Ensures a study arm record always inherits the `organization_sk` from its parent scenario configuration.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_scenario_config" BEFORE INSERT OR UPDATE ON "public"."map_study_partners" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_ownership_from_scenario_config"();



COMMENT ON TRIGGER "trg_sync_ownership_from_scenario_config" ON "public"."map_study_partners" IS 'Ensures a study partner record always inherits the `organization_sk` from its parent scenario configuration.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_study" BEFORE INSERT OR UPDATE ON "public"."dim_amendment" FOR EACH ROW EXECUTE FUNCTION "public"."sync_ownership_from_study"();



COMMENT ON TRIGGER "trg_sync_ownership_from_study" ON "public"."dim_amendment" IS 'Ensures an amendment record always inherits the `organization_sk` from its parent study.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_study" BEFORE INSERT OR UPDATE ON "public"."map_scenario_configuration" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_config_ownership_from_study"();



COMMENT ON TRIGGER "trg_sync_ownership_from_study" ON "public"."map_scenario_configuration" IS 'Ensures a scenario configuration record always inherits the `organization_sk` from its parent study.';



CREATE OR REPLACE TRIGGER "trg_sync_ownership_from_study_partner" BEFORE INSERT OR UPDATE ON "public"."map_study_staff" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_ownership_from_study_partner"();



COMMENT ON TRIGGER "trg_sync_ownership_from_study_partner" ON "public"."map_study_staff" IS 'Ensures a study staff record always inherits the `organization_sk` from its parent study partner.';



CREATE OR REPLACE TRIGGER "trg_sync_site_organization" BEFORE INSERT OR UPDATE ON "public"."dim_site" FOR EACH ROW EXECUTE FUNCTION "public"."trg_fn_sync_site_organization_from_partner"();



COMMENT ON TRIGGER "trg_sync_site_organization" ON "public"."dim_site" IS 'Ensures the denormalized `site_organization_sk` is always consistent with the `partner_organization_sk` from the linked study partner record.';



CREATE OR REPLACE TRIGGER "trg_update_dim_activity_cost_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_activity_cost" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_activity_cost_timestamp" ON "public"."dim_activity_cost" IS 'Automatically updates the updated_at timestamp on any modification to an activity cost record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_activity_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_activity" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_activity_timestamp" ON "public"."dim_activity" IS 'Automatically updates the updated_at timestamp on any modification to an activity record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_amendment_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_amendment" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_amendment_timestamp" ON "public"."dim_amendment" IS 'Automatically updates the updated_at timestamp on any modification to an amendment record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_budget_category_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_budget_category" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_budget_category_timestamp" ON "public"."dim_budget_category" IS 'Automatically updates the updated_at timestamp on any modification to a budget category record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_budget_scenario_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_budget_scenario" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_budget_scenario_timestamp" ON "public"."dim_budget_scenario" IS 'Automatically updates the updated_at timestamp on any modification to a budget scenario record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_forecast_calculation_config_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_forecast_calculation_config" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_forecast_calculation_config_timestamp" ON "public"."dim_forecast_calculation_config" IS 'Automatically updates the updated_at timestamp on any modification to a forecast rule record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_organization_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_organization" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_organization_timestamp" ON "public"."dim_organization" IS 'Automatically updates the updated_at timestamp on any modification to an organization record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_reimbursement_type_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_reimbursement_type" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_reimbursement_type_timestamp" ON "public"."dim_reimbursement_type" IS 'Automatically updates the updated_at timestamp on any modification to a reimbursement type record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_study_arm_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_study_arm" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_study_arm_timestamp" ON "public"."dim_study_arm" IS 'Automatically updates the updated_at timestamp on any modification to a study arm record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_study_epochs_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_study_epochs" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_study_epochs_timestamp" ON "public"."dim_study_epochs" IS 'Automatically updates the updated_at timestamp on any modification to a study epoch record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_study_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_study" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_study_timestamp" ON "public"."dim_study" IS 'Automatically updates the updated_at timestamp on any modification to a study record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_study_visits_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_study_visits" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_study_visits_timestamp" ON "public"."dim_study_visits" IS 'Automatically updates the updated_at timestamp on any modification to a study visit record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_dim_user_timestamp" BEFORE INSERT OR UPDATE ON "public"."dim_user" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_dim_user_timestamp" ON "public"."dim_user" IS 'Automatically updates the updated_at timestamp on any modification to a user record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_fact_enrollment_timestamp" BEFORE INSERT OR UPDATE ON "public"."fact_enrollment" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_fact_enrollment_timestamp" ON "public"."fact_enrollment" IS 'Automatically updates the updated_at timestamp on any modification to an enrollment fact record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_map_study_staff_timestamp" BEFORE INSERT OR UPDATE ON "public"."map_study_staff" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_map_study_staff_timestamp" ON "public"."map_study_staff" IS 'Automatically updates the updated_at timestamp on any modification to a study staff record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "trg_update_map_study_visit_activity_timestamp" BEFORE UPDATE ON "public"."map_study_visit_activity" FOR EACH ROW EXECUTE FUNCTION "public"."update_timestamp"();



COMMENT ON TRIGGER "trg_update_map_study_visit_activity_timestamp" ON "public"."map_study_visit_activity" IS 'Automatically updates the updated_at timestamp on any modification to a SoA mapping record, ensuring a reliable audit trail.';



CREATE OR REPLACE TRIGGER "validate_parent_organization" BEFORE INSERT OR UPDATE ON "public"."dim_organization" FOR EACH ROW EXECUTE FUNCTION "public"."validate_parent_organization_ref"();



COMMENT ON TRIGGER "validate_parent_organization" ON "public"."dim_organization" IS 'Enforces business rules for organizations, such as requiring a parent for fictitious orgs and ensuring only `org_*` BKs are Clerk-managed.';



ALTER TABLE ONLY "private"."template_activities"
    ADD CONSTRAINT "fk_activity_to_budget_category" FOREIGN KEY ("budget_category_bk") REFERENCES "private"."template_budget_categories"("budget_category_bk") ON DELETE RESTRICT;



COMMENT ON CONSTRAINT "fk_activity_to_budget_category" ON "private"."template_activities" IS 'FOREIGN KEY constraint linking an activity to its financial budget category, establishing the core link between operations and finance.';



ALTER TABLE ONLY "private"."template_activities"
    ADD CONSTRAINT "fk_activity_to_reimbursement_type" FOREIGN KEY ("reimbursement_type_bk") REFERENCES "private"."template_reimbursement_types"("reimbursement_type_bk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_activity_to_reimbursement_type" ON "private"."template_activities" IS 'FOREIGN KEY constraint linking an activity to a default reimbursement policy.';



ALTER TABLE ONLY "private"."template_amendments"
    ADD CONSTRAINT "fk_amendment_to_study" FOREIGN KEY ("parent_study_bk") REFERENCES "private"."template_studies"("study_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_amendment_to_study" ON "private"."template_amendments" IS 'FOREIGN KEY constraint linking an amendment to the parent study it modifies.';



ALTER TABLE ONLY "private"."template_study_arms"
    ADD CONSTRAINT "fk_arm_to_scenario_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_arm_to_scenario_config" ON "private"."template_study_arms" IS 'FOREIGN KEY constraint linking a study arm to a specific plan.';



ALTER TABLE ONLY "private"."template_budget_categories"
    ADD CONSTRAINT "fk_budget_category_to_parent" FOREIGN KEY ("parent_category_bk") REFERENCES "private"."template_budget_categories"("budget_category_bk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_budget_category_to_parent" ON "private"."template_budget_categories" IS 'FOREIGN KEY constraint establishing the self-referencing hierarchy for the chart of accounts.';



ALTER TABLE ONLY "private"."template_forecast_calculation_configs"
    ADD CONSTRAINT "fk_config_to_activity" FOREIGN KEY ("activity_bk") REFERENCES "private"."template_activities"("activity_bk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_config_to_activity" ON "private"."template_forecast_calculation_configs" IS 'FOREIGN KEY constraint linking a forecast rule to a specific activity.';



ALTER TABLE ONLY "private"."template_scenario_configurations"
    ADD CONSTRAINT "fk_config_to_scenario" FOREIGN KEY ("parent_scenario_bk") REFERENCES "private"."template_scenarios"("scenario_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_config_to_scenario" ON "private"."template_scenario_configurations" IS 'FOREIGN KEY constraint linking a configuration to its parent scenario.';



ALTER TABLE ONLY "private"."template_forecast_calculation_configs"
    ADD CONSTRAINT "fk_config_to_scenario_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_config_to_scenario_config" ON "private"."template_forecast_calculation_configs" IS 'FOREIGN KEY constraint linking a forecast rule to a specific plan.';



ALTER TABLE ONLY "private"."template_scenario_configurations"
    ADD CONSTRAINT "fk_config_to_study" FOREIGN KEY ("parent_study_bk") REFERENCES "private"."template_studies"("study_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_config_to_study" ON "private"."template_scenario_configurations" IS 'FOREIGN KEY constraint linking a configuration to its parent study.';



ALTER TABLE ONLY "private"."template_activity_costs"
    ADD CONSTRAINT "fk_cost_to_activity" FOREIGN KEY ("activity_bk") REFERENCES "private"."template_activities"("activity_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_cost_to_activity" ON "private"."template_activity_costs" IS 'FOREIGN KEY constraint linking a cost record to the specific activity being priced.';



ALTER TABLE ONLY "private"."template_activity_costs"
    ADD CONSTRAINT "fk_cost_to_reimbursement_type" FOREIGN KEY ("reimbursement_type_bk") REFERENCES "private"."template_reimbursement_types"("reimbursement_type_bk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_cost_to_reimbursement_type" ON "private"."template_activity_costs" IS 'FOREIGN KEY constraint linking a cost to a reimbursement policy that may override the activity''s default.';



ALTER TABLE ONLY "private"."template_activity_costs"
    ADD CONSTRAINT "fk_cost_to_scenario_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_cost_to_scenario_config" ON "private"."template_activity_costs" IS 'FOREIGN KEY constraint linking a cost record to a specific plan, ensuring pricing is versioned and scoped correctly.';



ALTER TABLE ONLY "private"."template_fact_enrollment"
    ADD CONSTRAINT "fk_enrollment_to_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_enrollment_to_config" ON "private"."template_fact_enrollment" IS 'FOREIGN KEY constraint linking an enrollment fact to the specific scenario configuration (plan) it belongs to.';



ALTER TABLE ONLY "private"."template_study_epochs"
    ADD CONSTRAINT "fk_epoch_to_arm" FOREIGN KEY ("parent_arm_bk") REFERENCES "private"."template_study_arms"("arm_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_epoch_to_arm" ON "private"."template_study_epochs" IS 'FOREIGN KEY constraint linking an epoch to its parent study arm.';



ALTER TABLE ONLY "private"."template_fact_forecast_detail"
    ADD CONSTRAINT "fk_forecast_to_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_forecast_to_config" ON "private"."template_fact_forecast_detail" IS 'FOREIGN KEY constraint linking a forecast detail fact to the specific scenario configuration (plan) it belongs to.';



ALTER TABLE ONLY "private"."template_memberships"
    ADD CONSTRAINT "fk_membership_to_organization" FOREIGN KEY ("organization_bk") REFERENCES "private"."template_organizations"("organization_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_membership_to_organization" ON "private"."template_memberships" IS 'FOREIGN KEY constraint linking a membership to an organization.';



ALTER TABLE ONLY "private"."template_memberships"
    ADD CONSTRAINT "fk_membership_to_user" FOREIGN KEY ("user_bk") REFERENCES "private"."template_users"("user_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_membership_to_user" ON "private"."template_memberships" IS 'FOREIGN KEY constraint linking a membership to a user.';



ALTER TABLE ONLY "private"."template_study_partners"
    ADD CONSTRAINT "fk_partner_to_organization" FOREIGN KEY ("partner_organization_bk") REFERENCES "private"."template_organizations"("organization_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_partner_to_organization" ON "private"."template_study_partners" IS 'FOREIGN KEY constraint linking a partner assignment to an organization.';



ALTER TABLE ONLY "private"."template_study_partners"
    ADD CONSTRAINT "fk_partner_to_scenario_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_partner_to_scenario_config" ON "private"."template_study_partners" IS 'FOREIGN KEY constraint linking a partner assignment to a specific plan.';



ALTER TABLE ONLY "private"."template_scenarios"
    ADD CONSTRAINT "fk_scenario_to_amendment" FOREIGN KEY ("triggering_amendment_bk") REFERENCES "private"."template_amendments"("amendment_bk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_scenario_to_amendment" ON "private"."template_scenarios" IS 'FOREIGN KEY constraint linking a scenario to the amendment that may have triggered its creation.';



ALTER TABLE ONLY "private"."template_sites"
    ADD CONSTRAINT "fk_site_to_performance_group" FOREIGN KEY ("performance_group_config_bk") REFERENCES "private"."template_forecast_calculation_configs"("config_bk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_site_to_performance_group" ON "private"."template_sites" IS 'FOREIGN KEY constraint linking a site to an enrollment performance group (an enrollment curve rule).';



ALTER TABLE ONLY "private"."template_sites"
    ADD CONSTRAINT "fk_site_to_scenario_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_site_to_scenario_config" ON "private"."template_sites" IS 'FOREIGN KEY constraint linking a site assignment to a specific plan.';



ALTER TABLE ONLY "private"."template_soa_mappings"
    ADD CONSTRAINT "fk_soa_to_activity" FOREIGN KEY ("activity_bk") REFERENCES "private"."template_activities"("activity_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_soa_to_activity" ON "private"."template_soa_mappings" IS 'FOREIGN KEY constraint linking a SoA mapping to an activity.';



ALTER TABLE ONLY "private"."template_soa_mappings"
    ADD CONSTRAINT "fk_soa_to_scenario_config" FOREIGN KEY ("parent_scenario_configuration_bk") REFERENCES "private"."template_scenario_configurations"("scenario_configuration_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_soa_to_scenario_config" ON "private"."template_soa_mappings" IS 'FOREIGN KEY constraint linking a SoA mapping to a specific plan.';



ALTER TABLE ONLY "private"."template_soa_mappings"
    ADD CONSTRAINT "fk_soa_to_visit" FOREIGN KEY ("visit_bk") REFERENCES "private"."template_study_visits"("visit_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_soa_to_visit" ON "private"."template_soa_mappings" IS 'FOREIGN KEY constraint linking a SoA mapping to a visit.';



ALTER TABLE ONLY "private"."template_study_staff"
    ADD CONSTRAINT "fk_study_staff_to_partner" FOREIGN KEY ("parent_study_partner_bk") REFERENCES "private"."template_study_partners"("study_partner_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_study_staff_to_partner" ON "private"."template_study_staff" IS 'FOREIGN KEY constraint linking a staff assignment to a study partner.';



ALTER TABLE ONLY "private"."template_study_staff"
    ADD CONSTRAINT "fk_study_staff_to_user" FOREIGN KEY ("staff_user_bk") REFERENCES "private"."template_users"("user_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_study_staff_to_user" ON "private"."template_study_staff" IS 'FOREIGN KEY constraint linking a staff assignment to a user.';



ALTER TABLE ONLY "private"."template_study_visits"
    ADD CONSTRAINT "fk_visit_to_epoch" FOREIGN KEY ("parent_epoch_bk") REFERENCES "private"."template_study_epochs"("epoch_bk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_visit_to_epoch" ON "private"."template_study_visits" IS 'FOREIGN KEY constraint linking a visit to its parent epoch.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "dim_activity_cost_reimbursement_type_sk_fkey" FOREIGN KEY ("reimbursement_type_sk") REFERENCES "public"."dim_reimbursement_type"("reimbursement_type_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "dim_activity_cost_reimbursement_type_sk_fkey" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint linking a cost to a specific reimbursement policy.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "dim_organization_parent_organization_sk_fkey" FOREIGN KEY ("parent_organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "dim_organization_parent_organization_sk_fkey" ON "public"."dim_organization" IS 'FOREIGN KEY constraint establishing the self-referencing hierarchy for sandbox organizations.';



ALTER TABLE ONLY "public"."fact_user_token_usage"
    ADD CONSTRAINT "fact_user_token_usage_organization_sk_fkey" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk");



COMMENT ON CONSTRAINT "fact_user_token_usage_organization_sk_fkey" ON "public"."fact_user_token_usage" IS 'FOREIGN KEY constraint linking the usage record to an organization.';



ALTER TABLE ONLY "public"."fact_user_token_usage"
    ADD CONSTRAINT "fact_user_token_usage_user_sk_fkey" FOREIGN KEY ("user_sk") REFERENCES "public"."dim_user"("user_sk");



COMMENT ON CONSTRAINT "fact_user_token_usage_user_sk_fkey" ON "public"."fact_user_token_usage" IS 'FOREIGN KEY constraint linking the usage record to a user.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "fk_activity_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_activity_created_by" ON "public"."dim_activity" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "fk_activity_organization" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk");



COMMENT ON CONSTRAINT "fk_activity_organization" ON "public"."dim_activity" IS 'FOREIGN KEY constraint linking the activity to its owning organization, enforcing tenant isolation.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "fk_activity_reimbursement_type" FOREIGN KEY ("reimbursement_type_sk") REFERENCES "public"."dim_reimbursement_type"("reimbursement_type_sk");



COMMENT ON CONSTRAINT "fk_activity_reimbursement_type" ON "public"."dim_activity" IS 'FOREIGN KEY constraint linking an activity to its default reimbursement policy.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "fk_activity_to_budget_category" FOREIGN KEY ("budget_category_sk") REFERENCES "public"."dim_budget_category"("budget_category_sk") ON DELETE RESTRICT;



COMMENT ON CONSTRAINT "fk_activity_to_budget_category" ON "public"."dim_activity" IS 'FOREIGN KEY constraint linking an activity to its financial budget category.';



ALTER TABLE ONLY "public"."dim_activity"
    ADD CONSTRAINT "fk_activity_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_activity_updated_by" ON "public"."dim_activity" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "fk_activitycost_activity" FOREIGN KEY ("activity_sk") REFERENCES "public"."dim_activity"("activity_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_activitycost_activity" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint linking a cost to the activity being priced.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "fk_activitycost_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_activitycost_created_by" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "fk_activitycost_organization" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk");



COMMENT ON CONSTRAINT "fk_activitycost_organization" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint linking the cost to its owning organization.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "fk_activitycost_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_activitycost_updated_by" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."agent_memory_log"
    ADD CONSTRAINT "fk_agent_log_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_agent_log_created_by" ON "public"."agent_memory_log" IS 'FOREIGN KEY constraint linking the log entry to the user who created it, providing an audit trail.';



ALTER TABLE ONLY "public"."agent_memory_log"
    ADD CONSTRAINT "fk_agent_log_organization" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_agent_log_organization" ON "public"."agent_memory_log" IS 'FOREIGN KEY constraint linking the log entry to an organization, enforcing tenant isolation.';



ALTER TABLE ONLY "public"."agent_memory_log"
    ADD CONSTRAINT "fk_agent_log_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_agent_log_updated_by" ON "public"."agent_memory_log" IS 'FOREIGN KEY constraint linking the log entry to the user who last updated it.';



ALTER TABLE ONLY "public"."agent_memory_log"
    ADD CONSTRAINT "fk_agent_memory_log_study" FOREIGN KEY ("study_sk") REFERENCES "public"."dim_study"("study_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_agent_memory_log_study" ON "public"."agent_memory_log" IS 'FOREIGN KEY constraint linking the log entry to a specific study, providing operational context.';



ALTER TABLE ONLY "public"."dim_amendment"
    ADD CONSTRAINT "fk_amendment_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_amendment_created_by" ON "public"."dim_amendment" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_amendment"
    ADD CONSTRAINT "fk_amendment_study" FOREIGN KEY ("study_sk") REFERENCES "public"."dim_study"("study_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_amendment_study" ON "public"."dim_amendment" IS 'FOREIGN KEY constraint linking an amendment to the parent study it modifies.';



ALTER TABLE ONLY "public"."dim_amendment"
    ADD CONSTRAINT "fk_amendment_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_amendment_updated_by" ON "public"."dim_amendment" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_study_arm"
    ADD CONSTRAINT "fk_arm_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_arm_created_by" ON "public"."dim_study_arm" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_study_arm"
    ADD CONSTRAINT "fk_arm_to_scenario_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_arm_to_scenario_config" ON "public"."dim_study_arm" IS 'FOREIGN KEY constraint linking a study arm to a specific plan.';



ALTER TABLE ONLY "public"."dim_study_arm"
    ADD CONSTRAINT "fk_arm_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_arm_updated_by" ON "public"."dim_study_arm" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_budget_category"
    ADD CONSTRAINT "fk_budget_category_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_budget_category_created_by" ON "public"."dim_budget_category" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_budget_category"
    ADD CONSTRAINT "fk_budget_category_parent" FOREIGN KEY ("parent_category_sk") REFERENCES "public"."dim_budget_category"("budget_category_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_budget_category_parent" ON "public"."dim_budget_category" IS 'FOREIGN KEY constraint establishing the self-referencing hierarchy for the chart of accounts.';



ALTER TABLE ONLY "public"."dim_budget_category"
    ADD CONSTRAINT "fk_budget_category_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_budget_category_updated_by" ON "public"."dim_budget_category" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."checkpoint_blobs"
    ADD CONSTRAINT "fk_checkpoint_blobs_organization_sk" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_checkpoint_blobs_organization_sk" ON "public"."checkpoint_blobs" IS 'FOREIGN KEY constraint linking the blob to an organization for tenant isolation.';



ALTER TABLE ONLY "public"."checkpoint_migrations"
    ADD CONSTRAINT "fk_checkpoint_migrations_organization_sk" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_checkpoint_migrations_organization_sk" ON "public"."checkpoint_migrations" IS 'FOREIGN KEY constraint linking the migration record to an organization for tenant isolation.';



ALTER TABLE ONLY "public"."checkpoint_writes"
    ADD CONSTRAINT "fk_checkpoint_writes_organization_sk" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_checkpoint_writes_organization_sk" ON "public"."checkpoint_writes" IS 'FOREIGN KEY constraint linking the write log to an organization for tenant isolation.';



ALTER TABLE ONLY "public"."checkpoints"
    ADD CONSTRAINT "fk_checkpoints_organization_sk" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_checkpoints_organization_sk" ON "public"."checkpoints" IS 'FOREIGN KEY constraint linking the checkpoint to an organization for tenant isolation.';



ALTER TABLE ONLY "public"."dim_forecast_calculation_config"
    ADD CONSTRAINT "fk_config_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_config_created_by" ON "public"."dim_forecast_calculation_config" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."map_scenario_configuration"
    ADD CONSTRAINT "fk_config_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_config_created_by" ON "public"."map_scenario_configuration" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."map_scenario_configuration"
    ADD CONSTRAINT "fk_config_scenario" FOREIGN KEY ("scenario_sk") REFERENCES "public"."dim_budget_scenario"("scenario_sk") ON DELETE RESTRICT;



ALTER TABLE ONLY "public"."map_scenario_configuration"
    ADD CONSTRAINT "fk_config_study" FOREIGN KEY ("study_sk") REFERENCES "public"."dim_study"("study_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_config_study" ON "public"."map_scenario_configuration" IS 'FOREIGN KEY constraint linking a configuration to its parent study.';



ALTER TABLE ONLY "public"."dim_forecast_calculation_config"
    ADD CONSTRAINT "fk_config_to_scenario_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_config_to_scenario_config" ON "public"."dim_forecast_calculation_config" IS 'FOREIGN KEY constraint linking a forecast rule to a specific plan.';



ALTER TABLE ONLY "public"."dim_forecast_calculation_config"
    ADD CONSTRAINT "fk_config_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_config_updated_by" ON "public"."dim_forecast_calculation_config" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."map_scenario_configuration"
    ADD CONSTRAINT "fk_config_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_config_updated_by" ON "public"."map_scenario_configuration" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "fk_cost_bearer_to_partner" FOREIGN KEY ("cost_bearing_partner_sk") REFERENCES "public"."map_study_partners"("study_partner_sk");



COMMENT ON CONSTRAINT "fk_cost_bearer_to_partner" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint identifying the study partner who is the payee for this cost.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "fk_cost_to_scenario_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_cost_to_scenario_config" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint linking the cost to a specific plan.';



ALTER TABLE ONLY "public"."dim_budget_category"
    ADD CONSTRAINT "fk_dim_budget_category_organization_sk" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_dim_budget_category_organization_sk" ON "public"."dim_budget_category" IS 'FOREIGN KEY constraint linking the budget category to its owning organization.';



ALTER TABLE ONLY "public"."fact_enrollment"
    ADD CONSTRAINT "fk_enrollment_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_enrollment_created_by" ON "public"."fact_enrollment" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."fact_enrollment"
    ADD CONSTRAINT "fk_enrollment_date" FOREIGN KEY ("snapshot_date_sk") REFERENCES "public"."dim_date"("date_sk");



COMMENT ON CONSTRAINT "fk_enrollment_date" ON "public"."fact_enrollment" IS 'FOREIGN KEY constraint linking the enrollment fact to the date dimension.';



ALTER TABLE ONLY "public"."fact_enrollment"
    ADD CONSTRAINT "fk_enrollment_to_scenario_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_enrollment_to_scenario_config" ON "public"."fact_enrollment" IS 'FOREIGN KEY constraint linking an enrollment fact to the specific plan it belongs to.';



ALTER TABLE ONLY "public"."fact_enrollment"
    ADD CONSTRAINT "fk_enrollment_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_enrollment_updated_by" ON "public"."fact_enrollment" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_study_epochs"
    ADD CONSTRAINT "fk_epoch_to_arm" FOREIGN KEY ("arm_sk") REFERENCES "public"."dim_study_arm"("arm_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_epoch_to_arm" ON "public"."dim_study_epochs" IS 'FOREIGN KEY constraint linking an epoch to its parent study arm.';



ALTER TABLE ONLY "public"."dim_study_epochs"
    ADD CONSTRAINT "fk_epochs_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_epochs_created_by" ON "public"."dim_study_epochs" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_study_epochs"
    ADD CONSTRAINT "fk_epochs_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_epochs_updated_by" ON "public"."dim_study_epochs" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_activity" FOREIGN KEY ("activity_sk") REFERENCES "public"."dim_activity"("activity_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_ffd_activity" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint linking a forecast fact to the activity it represents.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk");



COMMENT ON CONSTRAINT "fk_ffd_created_by" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_date" FOREIGN KEY ("snapshot_date_sk") REFERENCES "public"."dim_date"("date_sk");



COMMENT ON CONSTRAINT "fk_ffd_date" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint linking the forecast fact to the date dimension.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_organization" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk");



COMMENT ON CONSTRAINT "fk_ffd_organization" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint linking the forecast fact to its owning organization.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_source_activity_cost" FOREIGN KEY ("source_activity_cost_sk") REFERENCES "public"."dim_activity_cost"("activity_cost_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_ffd_source_activity_cost" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint providing lineage to the specific activity cost record used for pricing.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_source_arm" FOREIGN KEY ("source_arm_sk") REFERENCES "public"."dim_study_arm"("arm_sk");



COMMENT ON CONSTRAINT "fk_ffd_source_arm" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint providing lineage to the originating study arm.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_source_config" FOREIGN KEY ("source_forecast_config_sk") REFERENCES "public"."dim_forecast_calculation_config"("forecast_config_sk");



COMMENT ON CONSTRAINT "fk_ffd_source_config" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint providing lineage to the forecast rule that generated this record.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_source_enrollment" FOREIGN KEY ("source_enrollment_pk") REFERENCES "public"."fact_enrollment"("enrollment_pk");



COMMENT ON CONSTRAINT "fk_ffd_source_enrollment" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint providing lineage to the originating enrollment fact record.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_source_epoch" FOREIGN KEY ("source_epoch_sk") REFERENCES "public"."dim_study_epochs"("epoch_sk");



COMMENT ON CONSTRAINT "fk_ffd_source_epoch" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint providing lineage to the originating epoch.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_source_soa" FOREIGN KEY ("source_map_soa_sk") REFERENCES "public"."map_study_visit_activity"("map_soa_sk");



COMMENT ON CONSTRAINT "fk_ffd_source_soa" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint providing lineage to the originating SoA mapping record.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_source_visit" FOREIGN KEY ("source_visit_sk") REFERENCES "public"."dim_study_visits"("visit_sk");



COMMENT ON CONSTRAINT "fk_ffd_source_visit" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint providing lineage to the originating visit for schedule-based events.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_to_scenario_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_ffd_to_scenario_config" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint linking a forecast fact to the specific plan it belongs to.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_ffd_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk");



COMMENT ON CONSTRAINT "fk_ffd_updated_by" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_forecast_calculation_config"
    ADD CONSTRAINT "fk_forecast_config_activity" FOREIGN KEY ("activity_sk") REFERENCES "public"."dim_activity"("activity_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_forecast_config_activity" ON "public"."dim_forecast_calculation_config" IS 'FOREIGN KEY constraint linking a forecast rule to a specific activity.';



ALTER TABLE ONLY "public"."dim_site"
    ADD CONSTRAINT "fk_map_site_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_map_site_created_by" ON "public"."dim_site" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_site"
    ADD CONSTRAINT "fk_map_site_to_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_map_site_to_config" ON "public"."dim_site" IS 'FOREIGN KEY constraint linking a site assignment to a specific plan.';



ALTER TABLE ONLY "public"."dim_site"
    ADD CONSTRAINT "fk_map_site_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_map_site_updated_by" ON "public"."dim_site" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "fk_map_visit_act_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_map_visit_act_created_by" ON "public"."map_study_visit_activity" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "fk_map_visit_act_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_map_visit_act_updated_by" ON "public"."map_study_visit_activity" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "fk_mapvisit_activity" FOREIGN KEY ("activity_sk") REFERENCES "public"."dim_activity"("activity_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_mapvisit_activity" ON "public"."map_study_visit_activity" IS 'FOREIGN KEY constraint linking a SoA mapping to an activity.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "fk_mapvisit_visit" FOREIGN KEY ("visit_sk") REFERENCES "public"."dim_study_visits"("visit_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_mapvisit_visit" ON "public"."map_study_visit_activity" IS 'FOREIGN KEY constraint linking a SoA mapping to a visit.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "fk_org_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_org_created_by" ON "public"."dim_organization" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_organization"
    ADD CONSTRAINT "fk_org_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_org_updated_by" ON "public"."dim_organization" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."map_study_partners"
    ADD CONSTRAINT "fk_partner_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_partner_created_by" ON "public"."map_study_partners" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."map_study_partners"
    ADD CONSTRAINT "fk_partner_organization" FOREIGN KEY ("partner_organization_sk") REFERENCES "public"."dim_organization"("organization_sk");



COMMENT ON CONSTRAINT "fk_partner_organization" ON "public"."map_study_partners" IS 'FOREIGN KEY constraint linking a partner assignment to an organization.';



ALTER TABLE ONLY "public"."map_study_partners"
    ADD CONSTRAINT "fk_partner_to_scenario_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_partner_to_scenario_config" ON "public"."map_study_partners" IS 'FOREIGN KEY constraint linking a partner assignment to a specific plan.';



ALTER TABLE ONLY "public"."map_study_partners"
    ADD CONSTRAINT "fk_partner_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_partner_updated_by" ON "public"."map_study_partners" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_payee_to_partner" FOREIGN KEY ("payee_partner_sk") REFERENCES "public"."map_study_partners"("study_partner_sk");



COMMENT ON CONSTRAINT "fk_payee_to_partner" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint identifying the study partner who is the payee for this cost.';



ALTER TABLE ONLY "public"."dim_activity_cost"
    ADD CONSTRAINT "fk_payer_to_partner" FOREIGN KEY ("payer_partner_sk") REFERENCES "public"."map_study_partners"("study_partner_sk");



COMMENT ON CONSTRAINT "fk_payer_to_partner" ON "public"."dim_activity_cost" IS 'FOREIGN KEY constraint identifying the study partner who is the payer for this cost.';



ALTER TABLE ONLY "public"."fact_forecast_detail"
    ADD CONSTRAINT "fk_payer_to_partner" FOREIGN KEY ("payer_partner_sk") REFERENCES "public"."map_study_partners"("study_partner_sk");



COMMENT ON CONSTRAINT "fk_payer_to_partner" ON "public"."fact_forecast_detail" IS 'FOREIGN KEY constraint identifying the study partner who is the payer for this cost.';



ALTER TABLE ONLY "public"."dim_reimbursement_type"
    ADD CONSTRAINT "fk_reimbursement_type_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_reimbursement_type_created_by" ON "public"."dim_reimbursement_type" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_reimbursement_type"
    ADD CONSTRAINT "fk_reimbursement_type_organization" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_reimbursement_type_organization" ON "public"."dim_reimbursement_type" IS 'FOREIGN KEY constraint linking the reimbursement type to its owning organization.';



ALTER TABLE ONLY "public"."dim_reimbursement_type"
    ADD CONSTRAINT "fk_reimbursement_type_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_reimbursement_type_updated_by" ON "public"."dim_reimbursement_type" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_budget_scenario"
    ADD CONSTRAINT "fk_scenario_amendment" FOREIGN KEY ("triggering_amendment_sk") REFERENCES "public"."dim_amendment"("amendment_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_scenario_amendment" ON "public"."dim_budget_scenario" IS 'FOREIGN KEY constraint linking a scenario to the amendment that may have triggered its creation.';



ALTER TABLE ONLY "public"."dim_budget_scenario"
    ADD CONSTRAINT "fk_scenario_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_scenario_created_by" ON "public"."dim_budget_scenario" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_budget_scenario"
    ADD CONSTRAINT "fk_scenario_organization" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_scenario_organization" ON "public"."dim_budget_scenario" IS 'FOREIGN KEY constraint linking the scenario to its owning organization.';



ALTER TABLE ONLY "public"."dim_budget_scenario"
    ADD CONSTRAINT "fk_scenario_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_scenario_updated_by" ON "public"."dim_budget_scenario" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_site"
    ADD CONSTRAINT "fk_site_to_partner" FOREIGN KEY ("site_partner_sk") REFERENCES "public"."map_study_partners"("study_partner_sk");



COMMENT ON CONSTRAINT "fk_site_to_partner" ON "public"."dim_site" IS 'FOREIGN KEY constraint linking a site assignment to the study partner record that represents the site organization.';



ALTER TABLE ONLY "public"."map_study_visit_activity"
    ADD CONSTRAINT "fk_soa_to_scenario_config" FOREIGN KEY ("scenario_configuration_sk") REFERENCES "public"."map_scenario_configuration"("scenario_configuration_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_soa_to_scenario_config" ON "public"."map_study_visit_activity" IS 'FOREIGN KEY constraint linking a SoA mapping to a specific plan.';



ALTER TABLE ONLY "public"."map_study_staff"
    ADD CONSTRAINT "fk_staff_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_staff_created_by" ON "public"."map_study_staff" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."map_study_staff"
    ADD CONSTRAINT "fk_staff_to_partner" FOREIGN KEY ("study_partner_sk") REFERENCES "public"."map_study_partners"("study_partner_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_staff_to_partner" ON "public"."map_study_staff" IS 'FOREIGN KEY constraint linking a staff assignment to a study partner.';



ALTER TABLE ONLY "public"."map_study_staff"
    ADD CONSTRAINT "fk_staff_to_user" FOREIGN KEY ("user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_staff_to_user" ON "public"."map_study_staff" IS 'FOREIGN KEY constraint linking a staff assignment to a user.';



ALTER TABLE ONLY "public"."map_study_staff"
    ADD CONSTRAINT "fk_staff_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_staff_updated_by" ON "public"."map_study_staff" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."dim_study"
    ADD CONSTRAINT "fk_study_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_study_created_by" ON "public"."dim_study" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_study"
    ADD CONSTRAINT "fk_study_organization" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_study_organization" ON "public"."dim_study" IS 'FOREIGN KEY constraint linking the study to its owning organization.';



ALTER TABLE ONLY "public"."dim_study"
    ADD CONSTRAINT "fk_study_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_study_updated_by" ON "public"."dim_study" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."subscriptions"
    ADD CONSTRAINT "fk_subscriptions_organization_sk" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_subscriptions_organization_sk" ON "public"."subscriptions" IS 'FOREIGN KEY constraint linking a subscription to its owning organization.';



ALTER TABLE ONLY "public"."dim_user"
    ADD CONSTRAINT "fk_user_sandbox_org" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_user_sandbox_org" ON "public"."dim_user" IS 'FOREIGN KEY constraint linking a fictitious user to their home sandbox organization.';



ALTER TABLE ONLY "public"."dim_study_visits"
    ADD CONSTRAINT "fk_visit_to_epoch" FOREIGN KEY ("epoch_sk") REFERENCES "public"."dim_study_epochs"("epoch_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "fk_visit_to_epoch" ON "public"."dim_study_visits" IS 'FOREIGN KEY constraint linking a visit to its parent epoch.';



ALTER TABLE ONLY "public"."dim_study_visits"
    ADD CONSTRAINT "fk_visits_created_by" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_visits_created_by" ON "public"."dim_study_visits" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."dim_study_visits"
    ADD CONSTRAINT "fk_visits_updated_by" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE SET NULL;



COMMENT ON CONSTRAINT "fk_visits_updated_by" ON "public"."dim_study_visits" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."map_shared_scenario"
    ADD CONSTRAINT "map_shared_scenario_budget_scenario_sk_fkey" FOREIGN KEY ("budget_scenario_sk") REFERENCES "public"."dim_budget_scenario"("scenario_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "map_shared_scenario_budget_scenario_sk_fkey" ON "public"."map_shared_scenario" IS 'FOREIGN KEY constraint linking to the specific budget scenario being shared.';



ALTER TABLE ONLY "public"."map_shared_scenario"
    ADD CONSTRAINT "map_shared_scenario_created_by_user_sk_fkey" FOREIGN KEY ("created_by_user_sk") REFERENCES "public"."dim_user"("user_sk");



COMMENT ON CONSTRAINT "map_shared_scenario_created_by_user_sk_fkey" ON "public"."map_shared_scenario" IS 'FOREIGN KEY constraint for audit trail, linking to the user who created the record.';



ALTER TABLE ONLY "public"."map_shared_scenario"
    ADD CONSTRAINT "map_shared_scenario_sponsor_organization_sk_fkey" FOREIGN KEY ("sponsor_organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "map_shared_scenario_sponsor_organization_sk_fkey" ON "public"."map_shared_scenario" IS 'FOREIGN KEY constraint linking to the organization that owns and is sharing the scenario.';



ALTER TABLE ONLY "public"."map_shared_scenario"
    ADD CONSTRAINT "map_shared_scenario_updated_by_user_sk_fkey" FOREIGN KEY ("updated_by_user_sk") REFERENCES "public"."dim_user"("user_sk");



COMMENT ON CONSTRAINT "map_shared_scenario_updated_by_user_sk_fkey" ON "public"."map_shared_scenario" IS 'FOREIGN KEY constraint for audit trail, linking to the user who last updated the record.';



ALTER TABLE ONLY "public"."map_shared_scenario"
    ADD CONSTRAINT "map_shared_scenario_vendor_organization_sk_fkey" FOREIGN KEY ("vendor_organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "map_shared_scenario_vendor_organization_sk_fkey" ON "public"."map_shared_scenario" IS 'FOREIGN KEY constraint linking to the organization with whom the scenario is being shared.';



ALTER TABLE ONLY "public"."threads"
    ADD CONSTRAINT "threads_organization_sk_fkey" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk");



COMMENT ON CONSTRAINT "threads_organization_sk_fkey" ON "public"."threads" IS 'FOREIGN KEY constraint linking a thread to its owning organization.';



ALTER TABLE ONLY "public"."user_organization_membership"
    ADD CONSTRAINT "user_organization_membership_organization_sk_fkey" FOREIGN KEY ("organization_sk") REFERENCES "public"."dim_organization"("organization_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "user_organization_membership_organization_sk_fkey" ON "public"."user_organization_membership" IS 'FOREIGN KEY constraint linking a membership to an organization.';



ALTER TABLE ONLY "public"."user_organization_membership"
    ADD CONSTRAINT "user_organization_membership_user_sk_fkey" FOREIGN KEY ("user_sk") REFERENCES "public"."dim_user"("user_sk") ON DELETE CASCADE;



COMMENT ON CONSTRAINT "user_organization_membership_user_sk_fkey" ON "public"."user_organization_membership" IS 'FOREIGN KEY constraint linking a membership to a user.';


CREATE POLICY "Template Read-Only Access" ON "private"."template_activities" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_amendments" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_budget_categories" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_forecast_calculation_configs" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_memberships" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_organizations" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_reimbursement_types" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_scenarios" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_soa_mappings" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_studies" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_study_arms" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_study_epochs" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_study_visits" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Template Read-Only Access" ON "private"."template_users" FOR SELECT TO "authenticated" USING (true);



ALTER TABLE "private"."template_activities" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_amendments" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_budget_categories" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_forecast_calculation_configs" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_memberships" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_organizations" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_reimbursement_types" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_scenarios" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_soa_mappings" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_studies" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_study_arms" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_study_epochs" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_study_visits" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "private"."template_users" ENABLE ROW LEVEL SECURITY;


CREATE POLICY "Activity Cost Sandbox Policy" ON "public"."dim_activity_cost" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "dim_activity_cost"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Activity Sandbox Policy" ON "public"."dim_activity" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Agent Memory Log Sandbox Policy" ON "public"."agent_memory_log" FOR SELECT TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Amendment Sandbox Policy" ON "public"."dim_amendment" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."dim_study" "s"
  WHERE (("s"."study_sk" = "dim_amendment"."study_sk") AND ("s"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Budget Category Sandbox Policy" ON "public"."dim_budget_category" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Budget Scenario Sandbox Policy" ON "public"."dim_budget_scenario" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Checkpoint Blobs Sandbox Policy" ON "public"."checkpoint_blobs" FOR SELECT TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Checkpoint Blobs Write Protection Policy" ON "public"."checkpoint_blobs" USING (false) WITH CHECK (false);



CREATE POLICY "Checkpoint Migrations Sandbox Policy" ON "public"."checkpoint_migrations" FOR SELECT TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Checkpoint Migrations Write Protection Policy" ON "public"."checkpoint_migrations" USING (false) WITH CHECK (false);



CREATE POLICY "Checkpoint Sandbox Policy" ON "public"."checkpoints" FOR SELECT TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Checkpoint Write Protection Policy" ON "public"."checkpoints" USING (false) WITH CHECK (false);



CREATE POLICY "Checkpoint Writes Protection Policy" ON "public"."checkpoint_writes" USING (false) WITH CHECK (false);



CREATE POLICY "Checkpoint Writes Sandbox Policy" ON "public"."checkpoint_writes" FOR SELECT TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Enrollment Fact Sandbox Policy" ON "public"."fact_enrollment" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "fact_enrollment"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Forecast Config Sandbox Policy" ON "public"."dim_forecast_calculation_config" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "dim_forecast_calculation_config"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Forecast Detail Sandbox Policy" ON "public"."fact_forecast_detail" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "fact_forecast_detail"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Invitation Sandbox Policy" ON "public"."synced_clerk_invitations" FOR SELECT TO "authenticated" USING ("public"."has_organization_hierarchy_access_by_sk"("public"."get_current_organization_sk"(), "public"."get_organization_sk_from_bk"("organization_bk")));



CREATE POLICY "Membership Sandbox Policy" ON "public"."user_organization_membership" TO "authenticated" USING ("public"."has_organization_hierarchy_access_by_sk"("public"."get_current_organization_sk"(), "organization_sk")) WITH CHECK ("public"."has_organization_hierarchy_access_by_sk"("public"."get_current_organization_sk"(), "organization_sk"));



CREATE POLICY "Organization Sandbox Policy" ON "public"."dim_organization" TO "authenticated" USING ("public"."has_organization_hierarchy_access_by_sk"("public"."get_current_organization_sk"(), "organization_sk")) WITH CHECK ((("is_clerk_managed" = true) OR "public"."has_organization_hierarchy_access_by_sk"("public"."get_current_organization_sk"(), "parent_organization_sk")));



CREATE POLICY "Public Read Access" ON "public"."dim_date" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Public Read Access" ON "public"."dim_model_costs" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Public Read Access" ON "public"."dim_plan_limits" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Public Read Access" ON "public"."schema_version" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "Reimbursement Type Sandbox Policy" ON "public"."dim_reimbursement_type" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Scenario Configuration Sandbox Policy" ON "public"."map_scenario_configuration" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."dim_study" "s"
  WHERE (("s"."study_sk" = "map_scenario_configuration"."study_sk") AND ("s"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Shared Scenario Sandbox Policy" ON "public"."map_shared_scenario" TO "authenticated" USING ((("sponsor_organization_sk" = "public"."get_current_organization_sk"()) OR ("vendor_organization_sk" = "public"."get_current_organization_sk"()))) WITH CHECK (("sponsor_organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Site Sandbox Policy" ON "public"."dim_site" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "dim_site"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "SoA Mapping Sandbox Policy" ON "public"."map_study_visit_activity" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "map_study_visit_activity"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Study Arm Sandbox Policy" ON "public"."dim_study_arm" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "dim_study_arm"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Study Epoch Sandbox Policy" ON "public"."dim_study_epochs" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."dim_study_arm" "dsa"
  WHERE (("dsa"."arm_sk" = "dim_study_epochs"."arm_sk") AND ("dsa"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Study Partner Sandbox Policy" ON "public"."map_study_partners" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_scenario_configuration" "msc"
  WHERE (("msc"."scenario_configuration_sk" = "map_study_partners"."scenario_configuration_sk") AND ("msc"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Study Sandbox Policy" ON "public"."dim_study" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Study Staff Sandbox Policy" ON "public"."map_study_staff" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."map_study_partners" "msp"
  WHERE (("msp"."study_partner_sk" = "map_study_staff"."study_partner_sk") AND ("msp"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Study Visit Sandbox Policy" ON "public"."dim_study_visits" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK ((EXISTS ( SELECT 1
   FROM "public"."dim_study_epochs" "dse"
  WHERE (("dse"."epoch_sk" = "dim_study_visits"."epoch_sk") AND ("dse"."organization_sk" = "public"."get_current_organization_sk"())))));



CREATE POLICY "Subscription Sandbox Policy" ON "public"."subscriptions" FOR SELECT TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "Subscription Write Protection Policy" ON "public"."subscriptions" USING (false) WITH CHECK (false);



CREATE POLICY "Thread Sandbox Policy" ON "public"."threads" TO "authenticated" USING (("organization_sk" = "public"."get_current_organization_sk"())) WITH CHECK (("organization_sk" = "public"."get_current_organization_sk"()));



CREATE POLICY "User Read Org Usage Policy" ON "public"."fact_user_token_usage" FOR SELECT TO "authenticated" USING ((("organization_sk" = "public"."get_current_organization_sk"()) AND ("public"."is_current_user_admin"() OR ("user_sk" = "public"."get_current_user_sk"()))));



CREATE POLICY "User Sandbox Policy" ON "public"."dim_user" TO "authenticated" USING (((("is_clerk_managed" = true) AND ("user_bk" = NULLIF((("current_setting"('request.jwt.claims'::"text", true))::json ->> 'sub'::"text"), ''::"text"))) OR (("is_clerk_managed" = false) AND ("organization_sk" = "public"."get_current_organization_sk"())) OR ("public"."is_current_user_admin"() AND (EXISTS ( SELECT 1
   FROM "public"."user_organization_membership" "mem"
  WHERE (("mem"."user_sk" = "dim_user"."user_sk") AND ("mem"."organization_sk" = "public"."get_current_organization_sk"()))))))) WITH CHECK ((("is_clerk_managed" = false) AND ("organization_sk" = "public"."get_current_organization_sk"())));



CREATE POLICY "User Write Protection Policy" ON "public"."fact_user_token_usage" USING (false) WITH CHECK (false);



ALTER TABLE "public"."agent_memory_log" ENABLE ROW LEVEL SECURITY;


CREATE POLICY "allow_insert_usage_own_context" ON "public"."fact_user_token_usage" FOR INSERT TO "authenticated" WITH CHECK ((("public"."get_current_organization_bk"() = "organization_bk") AND ("public"."get_current_user_bk"() = "user_bk")));



CREATE POLICY "allow_select_plan_limits" ON "public"."dim_plan_limits" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "allow_select_usage_own_context" ON "public"."fact_user_token_usage" FOR SELECT TO "authenticated" USING ((("public"."get_current_organization_bk"() = "organization_bk") AND ("public"."get_current_user_bk"() = "user_bk")));



ALTER TABLE "public"."checkpoint_blobs" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."checkpoint_migrations" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."checkpoint_writes" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."checkpoints" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_activity" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_activity_cost" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_amendment" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_budget_category" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_budget_scenario" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_date" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_forecast_calculation_config" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_model_costs" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_organization" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_plan_limits" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_reimbursement_type" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_site" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_study" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_study_arm" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_study_epochs" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_study_visits" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."dim_user" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."fact_enrollment" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."fact_forecast_detail" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."fact_user_token_usage" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."map_scenario_configuration" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."map_shared_scenario" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."map_study_partners" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."map_study_staff" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."map_study_visit_activity" ENABLE ROW LEVEL SECURITY;


CREATE POLICY "public_read_model_costs" ON "public"."dim_model_costs" FOR SELECT TO "authenticated" USING (true);



CREATE POLICY "public_read_plan_limits" ON "public"."dim_plan_limits" FOR SELECT TO "authenticated" USING (true);



ALTER TABLE "public"."schema_version" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."subscriptions" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."synced_clerk_invitations" ENABLE ROW LEVEL SECURITY;


ALTER TABLE "public"."threads" ENABLE ROW LEVEL SECURITY;


CREATE POLICY "user_insert_own_usage" ON "public"."fact_user_token_usage" FOR INSERT TO "authenticated" WITH CHECK ((("user_sk" = "public"."get_current_user_sk"()) AND ("organization_sk" = "public"."get_current_organization_sk"())));



ALTER TABLE "public"."user_organization_membership" ENABLE ROW LEVEL SECURITY;


CREATE POLICY "user_read_org_usage" ON "public"."fact_user_token_usage" FOR SELECT TO "authenticated" USING ((("organization_sk" = "public"."get_current_organization_sk"()) AND ("private"."is_user_admin_in_org"("public"."get_current_user_sk"(), "public"."get_current_organization_sk"()) OR ("user_sk" = "public"."get_current_user_sk"()))));





ALTER PUBLICATION "supabase_realtime" OWNER TO "postgres";








GRANT USAGE ON SCHEMA "private" TO "service_role";



REVOKE USAGE ON SCHEMA "public" FROM PUBLIC;
GRANT ALL ON SCHEMA "public" TO PUBLIC;



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."materialized_template_payload" TO "service_role";



GRANT SELECT,INSERT,UPDATE ON TABLE "private"."sync_log" TO "service_role";



GRANT SELECT ON TABLE "private"."template_activities" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_activities" TO "service_role";



GRANT SELECT ON TABLE "private"."template_amendments" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_amendments" TO "service_role";



GRANT SELECT ON TABLE "private"."template_budget_categories" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_budget_categories" TO "service_role";



GRANT SELECT ON TABLE "private"."template_forecast_calculation_configs" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_forecast_calculation_configs" TO "service_role";



GRANT SELECT ON TABLE "private"."template_memberships" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_memberships" TO "service_role";



GRANT SELECT ON TABLE "private"."template_organizations" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_organizations" TO "service_role";



GRANT SELECT ON TABLE "private"."template_reimbursement_types" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_reimbursement_types" TO "service_role";



GRANT SELECT ON TABLE "private"."template_scenarios" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_scenarios" TO "service_role";



GRANT SELECT ON TABLE "private"."template_soa_mappings" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_soa_mappings" TO "service_role";



GRANT SELECT ON TABLE "private"."template_studies" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_studies" TO "service_role";



GRANT SELECT ON TABLE "private"."template_study_arms" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_study_arms" TO "service_role";



GRANT SELECT ON TABLE "private"."template_study_epochs" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_study_epochs" TO "service_role";



GRANT SELECT ON TABLE "private"."template_study_visits" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_study_visits" TO "service_role";



GRANT SELECT ON TABLE "private"."template_users" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "private"."template_users" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."agent_memory_log" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."agent_memory_log" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."agent_memory_log" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_blobs" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_blobs" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_blobs" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_migrations" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_migrations" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_migrations" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_writes" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_writes" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoint_writes" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoints" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoints" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."checkpoints" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_activity" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_activity" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_activity" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_activity_cost" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_activity_cost" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_activity_cost" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_amendment" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_amendment" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_amendment" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_budget_category" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_budget_category" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_budget_category" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_budget_scenario" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_budget_scenario" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_budget_scenario" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_date" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_date" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_date" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_forecast_calculation_config" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_forecast_calculation_config" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_forecast_calculation_config" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_organization" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_organization" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_organization" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_reimbursement_type" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_reimbursement_type" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_reimbursement_type" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_site" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_site" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_site" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_arm" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_arm" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_arm" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_epochs" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_epochs" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_epochs" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_visits" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_visits" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_study_visits" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_user" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_user" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."dim_user" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."fact_enrollment" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."fact_enrollment" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."fact_enrollment" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."fact_forecast_detail" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."fact_forecast_detail" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."fact_forecast_detail" TO "service_role";



GRANT ALL ON TABLE "public"."fact_user_token_usage" TO "authenticated";
GRANT ALL ON TABLE "public"."fact_user_token_usage" TO "anon";
GRANT ALL ON TABLE "public"."fact_user_token_usage" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_scenario_configuration" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_scenario_configuration" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_scenario_configuration" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_shared_scenario" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_shared_scenario" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_shared_scenario" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_partners" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_partners" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_partners" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_staff" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_staff" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_staff" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_visit_activity" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_visit_activity" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."map_study_visit_activity" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."schema_version" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."schema_version" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."schema_version" TO "service_role";



GRANT SELECT,INSERT ON TABLE "public"."subscriptions" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."subscriptions" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."synced_clerk_invitations" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."synced_clerk_invitations" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."synced_clerk_invitations" TO "service_role";



GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."user_organization_membership" TO "anon";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."user_organization_membership" TO "authenticated";
GRANT SELECT,INSERT,REFERENCES,DELETE,TRIGGER,TRUNCATE,UPDATE ON TABLE "public"."user_organization_membership" TO "service_role";



RESET ALL;
'''






new schema and functions specifically for this mcp server:
'''
create schema if not exists "private_ctgov";

create sequence "private_ctgov"."materialized_payload_id_seq";

create table "private_ctgov"."job_queue" (
    "job_id" uuid not null default gen_random_uuid(),
    "organization_bk" text not null,
    "job_type" text not null,
    "status" text not null default 'pending'::text,
    "payload" jsonb,
    "result" jsonb,
    "created_at" timestamp with time zone not null default now(),
    "started_at" timestamp with time zone,
    "completed_at" timestamp with time zone
);


create table "private_ctgov"."materialized_payload" (
    "id" bigint not null default nextval('private_ctgov.materialized_payload_id_seq'::regclass),
    "organization_bk" text not null,
    "stage_number" integer not null,
    "template_table_name" text not null,
    "payload" jsonb not null,
    "checksum" text,
    "metadata" jsonb,
    "last_refreshed_at" timestamp with time zone not null default now()
);


create table "private_ctgov"."organizations" (
    "organization_bk" text not null,
    "sponsor_bk" text not null,
    "sponsor_name_original" text not null,
    "sponsor_name_norm" text not null,
    "is_fictitious" boolean not null default true,
    "source" text not null default 'ctgov'::text,
    "created_at" timestamp with time zone not null default now(),
    "updated_at" timestamp with time zone not null default now(),
    "created_by" text,
    "updated_by" text
);


create table "private_ctgov"."scenario_configurations" (
    "organization_bk" text not null,
    "scenario_configuration_bk" text not null,
    "parent_study_bk" text not null,
    "parent_scenario_bk" text not null,
    "created_at" timestamp with time zone not null default now(),
    "updated_at" timestamp with time zone not null default now(),
    "start_date" text,
    "end_date" text,
    "target_enrollment" text,
    "target_sites" integer,
    "study_status" text
);


create table "private_ctgov"."scenarios" (
    "organization_bk" text not null,
    "scenario_bk" text not null,
    "scenario_name" text not null,
    "scenario_type" text not null default 'Forecast'::text,
    "description" text,
    "created_at" timestamp with time zone not null default now(),
    "updated_at" timestamp with time zone not null default now()
);


create table "private_ctgov"."studies" (
    "organization_bk" text not null,
    "study_bk" text not null,
    "protocol_number" text,
    "study_title" text not null,
    "study_short_name" text,
    "therapeutic_area" text,
    "phase" text,
    "include_in_json" boolean not null default true,
    "created_at" timestamp with time zone not null default now(),
    "updated_at" timestamp with time zone not null default now()
);


alter sequence "private_ctgov"."materialized_payload_id_seq" owned by "private_ctgov"."materialized_payload"."id";

CREATE INDEX ix_job_queue_org_status ON private_ctgov.job_queue USING btree (organization_bk, status);

CREATE INDEX ix_pctg_mat_payload_org ON private_ctgov.materialized_payload USING btree (organization_bk);

CREATE UNIQUE INDEX ix_pctg_mat_payload_org_stage_table ON private_ctgov.materialized_payload USING btree (organization_bk, stage_number, template_table_name);

CREATE INDEX ix_pctg_mat_payload_stage ON private_ctgov.materialized_payload USING btree (stage_number);

CREATE INDEX ix_pctg_orgs_norm ON private_ctgov.organizations USING btree (sponsor_name_norm);

CREATE INDEX ix_pctg_orgs_scope ON private_ctgov.organizations USING btree (organization_bk);

CREATE INDEX ix_pctg_studies_org ON private_ctgov.studies USING btree (organization_bk);

CREATE INDEX ix_pctg_studies_org_bk ON private_ctgov.studies USING btree (organization_bk, study_bk);

CREATE UNIQUE INDEX job_queue_pkey ON private_ctgov.job_queue USING btree (job_id);

CREATE UNIQUE INDEX materialized_payload_pkey ON private_ctgov.materialized_payload USING btree (id);

CREATE UNIQUE INDEX studies_organization_bk_study_bk_key ON private_ctgov.studies USING btree (organization_bk, study_bk);

CREATE UNIQUE INDEX uq_pctg_orgs_scope_bk ON private_ctgov.organizations USING btree (organization_bk, sponsor_bk);

CREATE UNIQUE INDEX uq_pctg_orgs_scope_norm_fict ON private_ctgov.organizations USING btree (organization_bk, sponsor_name_norm, is_fictitious);

CREATE UNIQUE INDEX uq_pctg_scen_configs_scope_bk ON private_ctgov.scenario_configurations USING btree (organization_bk, scenario_configuration_bk);

CREATE UNIQUE INDEX uq_pctg_scen_configs_scope_parents ON private_ctgov.scenario_configurations USING btree (organization_bk, parent_study_bk, parent_scenario_bk);

CREATE UNIQUE INDEX uq_pctg_scenarios_scope_bk ON private_ctgov.scenarios USING btree (organization_bk, scenario_bk);

alter table "private_ctgov"."job_queue" add constraint "job_queue_pkey" PRIMARY KEY using index "job_queue_pkey";

alter table "private_ctgov"."materialized_payload" add constraint "materialized_payload_pkey" PRIMARY KEY using index "materialized_payload_pkey";

alter table "private_ctgov"."organizations" add constraint "uq_pctg_orgs_scope_bk" UNIQUE using index "uq_pctg_orgs_scope_bk";

alter table "private_ctgov"."organizations" add constraint "uq_pctg_orgs_scope_norm_fict" UNIQUE using index "uq_pctg_orgs_scope_norm_fict";

alter table "private_ctgov"."scenario_configurations" add constraint "uq_pctg_scen_configs_scope_bk" UNIQUE using index "uq_pctg_scen_configs_scope_bk";

alter table "private_ctgov"."scenario_configurations" add constraint "uq_pctg_scen_configs_scope_parents" UNIQUE using index "uq_pctg_scen_configs_scope_parents";

alter table "private_ctgov"."scenarios" add constraint "uq_pctg_scenarios_scope_bk" UNIQUE using index "uq_pctg_scenarios_scope_bk";

alter table "private_ctgov"."studies" add constraint "studies_organization_bk_study_bk_key" UNIQUE using index "studies_organization_bk_study_bk_key";

set check_function_bodies = off;

CREATE OR REPLACE FUNCTION private_ctgov.normalize_sponsor(p_name text)
 RETURNS text
 LANGUAGE sql
AS $function$
  SELECT
    trim(
      regexp_replace(
        regexp_replace(lower(coalesce(p_name,'')), '\s+', ' ', 'g'),
        '[^a-z0-9 ]', '', 'g'
      )
    )
$function$
;

CREATE OR REPLACE FUNCTION private_ctgov.slugify_with_hash(p_text text, p_max_len integer DEFAULT 64)
 RETURNS text
 LANGUAGE plpgsql
AS $function$
DECLARE
  base TEXT;
  hash8 TEXT;
  max_len INT := GREATEST(16, COALESCE(p_max_len, 64));
BEGIN
  base := regexp_replace(lower(coalesce(p_text,'')), '[^a-z0-9]+', '-', 'g');
  base := regexp_replace(base, '-{2,}', '-', 'g');
  base := trim(both '-' FROM base);
  hash8 := substr(md5(coalesce(p_text,'')), 1, 8);
  IF base IS NULL OR base = '' THEN
    RETURN 'org-' || hash8;
  END IF;
  IF length(base) > max_len - 9 THEN
    base := substr(base, 1, max_len - 9);
    base := trim(trailing '-' FROM base);
  END IF;
  RETURN base || '-' || hash8;
END
$function$
;

CREATE OR REPLACE FUNCTION private_ctgov.trg_touch_updated_at()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END;
$function$
;

grant select on table "private_ctgov"."materialized_payload" to "service_role";

grant select on table "private_ctgov"."studies" to "service_role";

CREATE TRIGGER trg_touch_updated_at_pctg_orgs BEFORE UPDATE ON private_ctgov.organizations FOR EACH ROW EXECUTE FUNCTION private_ctgov.trg_touch_updated_at();

CREATE TRIGGER trg_touch_updated_at_pctg_scen_configs BEFORE UPDATE ON private_ctgov.scenario_configurations FOR EACH ROW EXECUTE FUNCTION private_ctgov.trg_touch_updated_at();

CREATE TRIGGER trg_touch_updated_at_pctg_scenarios BEFORE UPDATE ON private_ctgov.scenarios FOR EACH ROW EXECUTE FUNCTION private_ctgov.trg_touch_updated_at();

CREATE TRIGGER trg_touch_updated_at_pctg_studies BEFORE UPDATE ON private_ctgov.studies FOR EACH ROW EXECUTE FUNCTION private_ctgov.trg_touch_updated_at();


set check_function_bodies = off;

CREATE OR REPLACE FUNCTION public.adopt_ctgov_organizations(p_organization_bk text)
 RETURNS TABLE(status text, message text, adopted_count integer)
 LANGUAGE plpgsql
 SECURITY DEFINER
AS $function$
DECLARE
  v_parent_org_sk BIGINT;
  v_adopted_count INT := 0;
BEGIN
  v_parent_org_sk := public.resolve_organization_sk_from_bk(p_organization_bk);
  IF v_parent_org_sk IS NULL THEN
    RAISE EXCEPTION 'Parent organization not found for BK: %', p_organization_bk;
  END IF;

  -- Directly update the CTG- records to NULL, which will fire our trigger.
  -- This is simpler and more reliable than calling the complex update worker.
  WITH updated_rows AS (
    UPDATE public.dim_organization
    SET organization_bk = NULL
    WHERE parent_organization_sk = v_parent_org_sk
      AND organization_bk LIKE 'CTG-%'
    RETURNING 1
  )
  SELECT count(*) INTO v_adopted_count FROM updated_rows;

  RETURN QUERY SELECT 'success'::TEXT, 'Organizations adopted successfully.'::TEXT, v_adopted_count::INT;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.adopt_ctgov_public_data(p_organization_bk text, p_tables text[] DEFAULT NULL::text[], p_dry_run boolean DEFAULT false)
 RETURNS jsonb
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'pg_temp'
AS $function$
DECLARE
  v_started_at timestamptz := clock_timestamp();
  v_finished_at timestamptz;
  v_org_bk TEXT := btrim(COALESCE(p_organization_bk, ''));
  v_org_sk BIGINT; -- Will hold the resolved SK
  v_dry BOOLEAN := COALESCE(p_dry_run, FALSE);
  v_tables TEXT[] := NULLIF(p_tables, ARRAY[]::TEXT[]);
  v_report JSONB := '{}'::jsonb;
  v_tot_adopted INT := 0;
  v_tot_remaining INT := 0;
  v_tot_minted INT := 0;
  v_msgs TEXT[] := ARRAY[]::TEXT[];
  v_errors JSONB := '[]'::jsonb;

  rec RECORD;
  v_table TEXT;
  v_bk_column_name TEXT;
  v_adopted INT;
  v_remaining INT;
  v_minted INT;
  v_sample_bks TEXT[];
BEGIN
  PERFORM set_config('statement_timeout', '900000', true);

  IF v_org_bk IS NULL OR v_org_bk = '' THEN
    RAISE EXCEPTION 'p_organization_bk is required';
  END IF;

  -- [CORRECTION V4] Resolve the SK once at the beginning
  v_org_sk := public.resolve_organization_sk_from_bk(p_organization_bk);
  v_msgs := array_append(v_msgs, format('Discovering candidate tables for org_sk=%s', v_org_sk));
  
  -- [CORRECTION V4] The discovery query is now much more precise.
  -- It looks for tables with 'organization_sk' and another column ending in '_bk'.
  FOR rec IN
    SELECT
      t.table_schema,
      t.table_name,
      bk_cols.column_name AS bk_column
    FROM information_schema.tables t
    JOIN information_schema.columns org_cols ON org_cols.table_name = t.table_name AND org_cols.table_schema = t.table_schema
    JOIN information_schema.columns bk_cols ON bk_cols.table_name = t.table_name AND bk_cols.table_schema = t.table_schema
    WHERE t.table_schema = 'public'
      AND t.table_type = 'BASE TABLE'
      AND org_cols.column_name = 'organization_sk' -- Look for the correct tenancy column
      AND bk_cols.column_name LIKE '%_bk'
      AND bk_cols.column_name != 'organization_bk' -- Exclude the org table itself
  LOOP
    IF v_tables IS NOT NULL AND NOT (rec.table_name = ANY (v_tables)) THEN
      CONTINUE;
    END IF;

    v_table := quote_ident(rec.table_schema) || '.' || quote_ident(rec.table_name);
    v_bk_column_name := quote_ident(rec.bk_column);

    v_msgs := array_append(v_msgs, format('Processing table %s using BK column %s', v_table, v_bk_column_name));

    -- [CORRECTION V4] Use organization_sk for all queries
    EXECUTE format('SELECT count(*) FROM %s WHERE organization_sk = $1 AND %s LIKE ''CTG-%%''', v_table, v_bk_column_name)
      INTO v_remaining
      USING v_org_sk;
      
    IF v_remaining = 0 THEN
      v_report := v_report || jsonb_build_object(rec.table_name, jsonb_build_object(
        'adopted_ctg_count', 0, 'remaining_ctg_count_after', 0, 'minted_ctf_count', 0, 'sample_ctf_bks', ARRAY[]::text[]
      ));
      CONTINUE;
    END IF;

    IF v_dry THEN
      v_report := v_report || jsonb_build_object(rec.table_name, jsonb_build_object(
        'adopted_ctg_count', v_remaining, 'remaining_ctg_count_after', v_remaining, 'minted_ctf_count', 0, 'sample_ctf_bks', ARRAY[]::text[]
      ));
      v_tot_adopted := v_tot_adopted + v_remaining;
      v_tot_remaining := v_tot_remaining + v_remaining;
      CONTINUE;
    END IF;

    EXECUTE format($fmt$
      UPDATE %s SET %s = NULL WHERE organization_sk = $1 AND %s LIKE 'CTG-%%'
    $fmt$, v_table, v_bk_column_name, v_bk_column_name)
    USING v_org_sk;
    GET DIAGNOSTICS v_adopted = ROW_COUNT;

   EXECUTE format('SELECT count(*) FROM %s WHERE organization_sk = $1 AND %s LIKE ''CTG-%%''', v_table, v_bk_column_name)
     INTO v_remaining
     USING v_org_sk;

    EXECUTE format('SELECT array(SELECT %s FROM %s WHERE organization_sk = $1 AND %s LIKE ''CTF-%%'' LIMIT 5)', v_bk_column_name, v_table, v_bk_column_name)
      INTO v_sample_bks
      USING v_org_sk;
    v_minted := COALESCE(v_adopted, 0);

    v_report := v_report || jsonb_build_object(rec.table_name, jsonb_build_object(
      'adopted_ctg_count', COALESCE(v_adopted,0), 'remaining_ctg_count_after', COALESCE(v_remaining,0),
      'minted_ctf_count', COALESCE(v_minted,0), 'sample_ctf_bks', COALESCE(v_sample_bks, ARRAY[]::text[])
    ));
    v_tot_adopted := v_tot_adopted + COALESCE(v_adopted,0);
    v_tot_remaining := v_tot_remaining + COALESCE(v_remaining,0);
    v_tot_minted := v_tot_minted + COALESCE(v_minted,0);
  END LOOP;

  v_finished_at := clock_timestamp();
  RETURN jsonb_build_object(
    'organization_bk', v_org_bk, 'dry_run', v_dry, 'started_at', v_started_at, 'finished_at', v_finished_at,
    'per_table', v_report, 'totals', jsonb_build_object(
      'adopted_ctg_count', v_tot_adopted, 'remaining_ctg_count_after', v_tot_remaining, 'minted_ctf_count', v_tot_minted
    ), 'messages', v_msgs, 'errors', v_errors
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.clear_ctgov_private_payloads(p_organization_bk text, p_stages integer[] DEFAULT NULL::integer[], p_tables text[] DEFAULT NULL::text[], p_dry_run boolean DEFAULT false)
 RETURNS jsonb
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'pg_temp'
AS $function$
DECLARE
  v_started_at timestamptz := clock_timestamp();
  v_finished_at timestamptz;
  v_org_bk TEXT := btrim(COALESCE(p_organization_bk,''));
  v_dry BOOLEAN := COALESCE(p_dry_run, FALSE);
  v_stages INT[] := NULLIF(p_stages, ARRAY[]::INT[]);
  v_tables TEXT[] := NULLIF(p_tables, ARRAY[]::TEXT[]);
  v_report JSONB := '{}'::jsonb;
  v_tot_deleted INT := 0;
  v_msgs TEXT[] := ARRAY[]::TEXT[];
  v_errors JSONB := '[]'::jsonb;
  v_deleted INT;
  v_remaining INT;
BEGIN
  PERFORM set_config('statement_timeout', '900000', true);
  IF v_org_bk IS NULL OR v_org_bk = '' THEN
    RAISE EXCEPTION 'p_organization_bk is required';
  END IF;

  IF v_tables IS NULL OR 'materialized_payload' = ANY(v_tables) THEN
    IF v_dry THEN
      EXECUTE $q$ SELECT count(*) FROM private_ctgov.materialized_payload
                  WHERE organization_bk = $1
                    AND ($2::int[] IS NULL OR stage_number = ANY($2)) $q$
      INTO v_deleted
      USING v_org_bk, v_stages;
    ELSE
      EXECUTE $q$ DELETE FROM private_ctgov.materialized_payload
                  WHERE organization_bk = $1
                    AND ($2::int[] IS NULL OR stage_number = ANY($2)) $q$
      USING v_org_bk, v_stages;
      GET DIAGNOSTICS v_deleted = ROW_COUNT;
    END IF;
    v_tot_deleted := v_tot_deleted + COALESCE(v_deleted,0);
    EXECUTE $q$ SELECT count(*) FROM private_ctgov.materialized_payload WHERE organization_bk = $1 $q$
    INTO v_remaining USING v_org_bk;
    v_report := v_report || jsonb_build_object('materialized_payload', jsonb_build_object(
      'deleted_count', COALESCE(v_deleted,0), 'remaining_count_after', COALESCE(v_remaining,0)
    ));
  END IF;

  IF v_tables IS NULL OR 'studies' = ANY(v_tables) THEN
    IF v_dry THEN
      EXECUTE $q$ SELECT count(*) FROM private_ctgov.studies WHERE organization_bk = $1 $q$
      INTO v_deleted USING v_org_bk;
    ELSE
      EXECUTE $q$ DELETE FROM private_ctgov.studies WHERE organization_bk = $1 $q$
      USING v_org_bk;
      GET DIAGNOSTICS v_deleted = ROW_COUNT;
    END IF;
    v_tot_deleted := v_tot_deleted + COALESCE(v_deleted,0);
    EXECUTE $q$ SELECT count(*) FROM private_ctgov.studies WHERE organization_bk = $1 $q$
    INTO v_remaining USING v_org_bk;
    v_report := v_report || jsonb_build_object('studies', jsonb_build_object(
      'deleted_count', COALESCE(v_deleted,0), 'remaining_count_after', COALESCE(v_remaining,0)
    ));
  END IF;

  v_finished_at := clock_timestamp();
  RETURN jsonb_build_object(
    'organization_bk', v_org_bk, 'dry_run', v_dry, 'started_at', v_started_at, 'finished_at', v_finished_at,
    'per_table', v_report, 'totals', jsonb_build_object('deleted_count', v_tot_deleted),
    'messages', v_msgs, 'errors', v_errors
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.clear_ctgov_public_ctg_data(p_organization_bk text, p_tables text[] DEFAULT NULL::text[], p_dry_run boolean DEFAULT false)
 RETURNS jsonb
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'pg_temp'
AS $function$
DECLARE
  v_started_at timestamptz := clock_timestamp();
  v_finished_at timestamptz;
  v_org_bk TEXT := btrim(COALESCE(p_organization_bk,''));
  v_dry BOOLEAN := COALESCE(p_dry_run, FALSE);
  v_tables TEXT[] := NULLIF(p_tables, ARRAY[]::TEXT[]);
  v_report JSONB := '{}'::jsonb;
  v_tot_deleted INT := 0;
  v_tot_remaining INT := 0;
  v_msgs TEXT[] := ARRAY[]::TEXT[];
  v_errors JSONB := '[]'::jsonb;
  rec RECORD;
  v_table TEXT;
  v_deleted INT;
  v_remaining INT;
BEGIN
  PERFORM set_config('statement_timeout', '900000', true);
  IF v_org_bk IS NULL OR v_org_bk = '' THEN
    RAISE EXCEPTION 'p_organization_bk is required';
  END IF;

  v_msgs := array_append(v_msgs, 'Discovering candidate public tables with organization_bk and study_bk');
  FOR rec IN
    SELECT c.table_schema, c.table_name
    FROM information_schema.columns c
    JOIN information_schema.columns d
      ON d.table_schema = c.table_schema AND d.table_name = c.table_name AND d.column_name = 'study_bk'
    WHERE c.table_schema = 'public'
      AND c.column_name = 'organization_bk'
  LOOP
    IF v_tables IS NOT NULL AND NOT (rec.table_name = ANY (v_tables)) THEN
      CONTINUE;
    END IF;
    v_table := quote_ident(rec.table_schema) || '.' || quote_ident(rec.table_name);

    EXECUTE format('SELECT count(*) FROM %s WHERE organization_bk = $1 AND study_bk LIKE ''CTG-%%''', v_table)
      INTO v_deleted
      USING v_org_bk;
    IF v_deleted = 0 THEN
      v_report := v_report || jsonb_build_object(rec.table_name, jsonb_build_object(
        'deleted_ctg_count', 0, 'remaining_ctg_count_after', 0
      ));
      CONTINUE;
    END IF;

    IF NOT v_dry THEN
      EXECUTE format('DELETE FROM %s WHERE organization_bk = $1 AND study_bk LIKE ''CTG-%%''', v_table)
        USING v_org_bk;
      GET DIAGNOSTICS v_deleted = ROW_COUNT;
    END IF;

    EXECUTE format('SELECT count(*) FROM %s WHERE organization_bk = $1 AND study_bk LIKE ''CTG-%%''', v_table)
      INTO v_remaining
      USING v_org_bk;

    v_report := v_report || jsonb_build_object(rec.table_name, jsonb_build_object(
      'deleted_ctg_count', COALESCE(v_deleted,0), 'remaining_ctg_count_after', COALESCE(v_remaining,0)
    ));
    v_tot_deleted := v_tot_deleted + COALESCE(v_deleted,0);
    v_tot_remaining := v_tot_remaining + COALESCE(v_remaining,0);
  END LOOP;

  v_finished_at := clock_timestamp();
  RETURN jsonb_build_object(
    'organization_bk', v_org_bk, 'dry_run', v_dry, 'started_at', v_started_at, 'finished_at', v_finished_at,
    'per_table', v_report, 'totals', jsonb_build_object(
      'deleted_ctg_count', v_tot_deleted, 'remaining_ctg_count_after', v_tot_remaining
    ), 'messages', v_msgs, 'errors', v_errors
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_create_job(p_organization_bk text, p_job_type text, p_payload jsonb)
 RETURNS uuid
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov'
AS $function$
DECLARE
  v_job_id UUID;
BEGIN
  INSERT INTO private_ctgov.job_queue (organization_bk, job_type, payload)
  VALUES (p_organization_bk, p_job_type, p_payload)
  RETURNING job_id INTO v_job_id;
  RETURN v_job_id;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_delete_studies(p_organization_bk text, p_bks text[])
 RETURNS TABLE(action_report jsonb)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
DECLARE v_del INT := 0;
BEGIN
  DELETE FROM private_ctgov.studies
  WHERE organization_bk = p_organization_bk
    AND study_bk = ANY(p_bks);
  GET DIAGNOSTICS v_del = ROW_COUNT;

  RETURN QUERY SELECT jsonb_build_object(
    'status','success','table','studies','deleted', v_del
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_get_job_status(p_job_id uuid)
 RETURNS jsonb
 LANGUAGE sql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov'
AS $function$
  SELECT to_jsonb(t) FROM private_ctgov.job_queue t WHERE t.job_id = p_job_id;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_get_organization(p_organization_bk text, p_sponsor_bk text)
 RETURNS jsonb
 LANGUAGE sql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
  SELECT to_jsonb(o)
  FROM private_ctgov.organizations o
  WHERE o.organization_bk = p_organization_bk
    AND o.sponsor_bk = p_sponsor_bk
  LIMIT 1
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_get_studies_payload(p_organization_bk text, p_stage_number integer DEFAULT 2)
 RETURNS TABLE(organization_bk text, stage_number integer, template_table_name text, payload jsonb, checksum text, last_refreshed_at timestamp with time zone, metadata jsonb)
 LANGUAGE sql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
  SELECT
    mp.organization_bk,
    mp.stage_number,
    mp.template_table_name,
    mp.payload,
    mp.checksum,
    mp.last_refreshed_at,
    mp.metadata
  FROM private_ctgov.materialized_payload mp
  WHERE mp.organization_bk = p_organization_bk
    AND mp.stage_number = p_stage_number
    AND mp.template_table_name = 'studies'
  LIMIT 1;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_list_organizations(p_organization_bk text, p_is_fictitious boolean DEFAULT NULL::boolean)
 RETURNS SETOF jsonb
 LANGUAGE sql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
  SELECT to_jsonb(o)
  FROM private_ctgov.organizations o
  WHERE o.organization_bk = p_organization_bk
    AND (p_is_fictitious IS NULL OR o.is_fictitious = p_is_fictitious)
  ORDER BY o.created_at DESC
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_list_studies(p_organization_bk text)
 RETURNS SETOF jsonb
 LANGUAGE sql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
  SELECT to_jsonb(t)
  FROM private_ctgov.studies t
  WHERE t.organization_bk = p_organization_bk
  ORDER BY t.study_bk;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_refresh_organizations_payload(p_organization_bk text, p_stage_number integer DEFAULT 1, p_metadata jsonb DEFAULT NULL::jsonb)
 RETURNS TABLE(action_report jsonb)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
DECLARE
  v_payload JSONB := '[]'::jsonb;
  v_checksum TEXT;
  v_rowcount INT := 0;
BEGIN
  -- Aggregate all staged organizations for the tenant into a single JSON array
  SELECT COALESCE(jsonb_agg(to_jsonb(o.*)), '[]'::jsonb)
  INTO v_payload
  FROM private_ctgov.organizations o
  WHERE o.organization_bk = p_organization_bk;

  v_rowcount := COALESCE(jsonb_array_length(v_payload), 0);
  v_checksum := md5(v_payload::text);

  -- Upsert this payload into the materialized cache
  INSERT INTO private_ctgov.materialized_payload (
    organization_bk, stage_number, template_table_name, payload, checksum, metadata, last_refreshed_at
  ) VALUES (
    p_organization_bk, p_stage_number, 'organizations', v_payload, v_checksum, p_metadata, now()
  )
  ON CONFLICT (organization_bk, stage_number, template_table_name)
  DO UPDATE SET
    payload = EXCLUDED.payload,
    checksum = EXCLUDED.checksum,
    metadata = EXCLUDED.metadata,
    last_refreshed_at = now();

  RETURN QUERY SELECT jsonb_build_object(
    'status','success',
    'message','Refreshed organizations payload',
    'organization_bk', p_organization_bk,
    'stage_number', p_stage_number,
    'template_table_name','organizations',
    'row_count', v_rowcount,
    'checksum', v_checksum
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_refresh_scenario_configurations_payload(p_organization_bk text, p_stage_number integer DEFAULT 2, p_metadata jsonb DEFAULT NULL::jsonb)
 RETURNS TABLE(action_report jsonb)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov'
AS $function$
DECLARE
  v_payload JSONB; v_checksum TEXT; v_rowcount INT;
BEGIN
  SELECT COALESCE(jsonb_agg(to_jsonb(sc.*)), '[]'::jsonb) INTO v_payload
  FROM private_ctgov.scenario_configurations sc WHERE sc.organization_bk = p_organization_bk;
  v_rowcount := COALESCE(jsonb_array_length(v_payload), 0);
  v_checksum := md5(v_payload::text);
  INSERT INTO private_ctgov.materialized_payload (organization_bk, stage_number, template_table_name, payload, checksum, metadata)
  VALUES (p_organization_bk, p_stage_number, 'scenario_configurations', v_payload, v_checksum, p_metadata)
  ON CONFLICT (organization_bk, stage_number, template_table_name) DO UPDATE SET payload = EXCLUDED.payload, checksum = EXCLUDED.checksum, metadata = EXCLUDED.metadata, last_refreshed_at = now();
  RETURN QUERY SELECT jsonb_build_object('status','success', 'message','Refreshed scenario_configurations payload', 'row_count', v_rowcount);
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_refresh_scenarios_payload(p_organization_bk text, p_stage_number integer DEFAULT 1, p_metadata jsonb DEFAULT NULL::jsonb)
 RETURNS TABLE(action_report jsonb)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov'
AS $function$
DECLARE
  v_payload JSONB; v_checksum TEXT; v_rowcount INT;
BEGIN
  SELECT COALESCE(jsonb_agg(to_jsonb(s.*)), '[]'::jsonb) INTO v_payload
  FROM private_ctgov.scenarios s WHERE s.organization_bk = p_organization_bk;
  v_rowcount := COALESCE(jsonb_array_length(v_payload), 0);
  v_checksum := md5(v_payload::text);
  INSERT INTO private_ctgov.materialized_payload (organization_bk, stage_number, template_table_name, payload, checksum, metadata)
  VALUES (p_organization_bk, p_stage_number, 'scenarios', v_payload, v_checksum, p_metadata)
  ON CONFLICT (organization_bk, stage_number, template_table_name) DO UPDATE SET payload = EXCLUDED.payload, checksum = EXCLUDED.checksum, metadata = EXCLUDED.metadata, last_refreshed_at = now();
  RETURN QUERY SELECT jsonb_build_object('status','success', 'message','Refreshed scenarios payload', 'row_count', v_rowcount);
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_refresh_studies_payload(p_organization_bk text, p_stage_number integer DEFAULT 2, p_metadata jsonb DEFAULT NULL::jsonb)
 RETURNS TABLE(action_report jsonb)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
DECLARE
  v_payload JSONB := '[]'::jsonb;
  v_checksum TEXT;
  v_rowcount INT := 0;
BEGIN
  SELECT COALESCE(jsonb_agg(jsonb_build_object(
    'study_bk', s.study_bk,
    'protocol_number', s.protocol_number,
    'study_title', s.study_title,
    'study_short_name', s.study_short_name,
    'therapeutic_area', s.therapeutic_area,
    'phase', s.phase
  )), '[]'::jsonb)
  INTO v_payload
  FROM private_ctgov.studies s
  WHERE s.organization_bk = p_organization_bk
    AND s.include_in_json;

  v_rowcount := COALESCE(jsonb_array_length(v_payload), 0);
  v_checksum := md5(v_payload::text);

  INSERT INTO private_ctgov.materialized_payload (
    organization_bk, stage_number, template_table_name, payload, checksum, metadata, last_refreshed_at
  ) VALUES (
    p_organization_bk, p_stage_number, 'studies', v_payload, v_checksum, p_metadata, now()
  )
  ON CONFLICT (organization_bk, stage_number, template_table_name)
  DO UPDATE SET
    payload = EXCLUDED.payload,
    checksum = EXCLUDED.checksum,
    metadata = EXCLUDED.metadata,
    last_refreshed_at = now();

  RETURN QUERY SELECT jsonb_build_object(
    'status','success',
    'message','Refreshed studies payload',
    'organization_bk', p_organization_bk,
    'stage_number', p_stage_number,
    'template_table_name','studies',
    'row_count', v_rowcount,
    'checksum', v_checksum
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_update_job_status(p_job_id uuid, p_status text, p_result jsonb DEFAULT NULL::jsonb)
 RETURNS void
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov'
AS $function$
BEGIN
  UPDATE private_ctgov.job_queue
  SET
    status = p_status,
    result = COALESCE(p_result, result),
    started_at = CASE WHEN p_status = 'running' THEN now() ELSE started_at END,
    completed_at = CASE WHEN p_status IN ('completed', 'failed') THEN now() ELSE NULL END
  WHERE job_id = p_job_id;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_upsert_organization(p_organization_bk text, p_sponsor_name text, p_is_fictitious boolean DEFAULT true, p_source text DEFAULT 'ctgov'::text)
 RETURNS TABLE(sponsor_bk text, sponsor_name_original text, sponsor_name_norm text, is_fictitious boolean, source text, created_at timestamp with time zone, updated_at timestamp with time zone)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
DECLARE
  v_norm TEXT;
  v_bk TEXT;
BEGIN
  IF p_organization_bk IS NULL OR btrim(p_organization_bk) = '' THEN
    RAISE EXCEPTION 'p_organization_bk is required';
  END IF;
  IF p_sponsor_name IS NULL OR btrim(p_sponsor_name) = '' THEN
    RAISE EXCEPTION 'p_sponsor_name is required';
  END IF;

  v_norm := private_ctgov.normalize_sponsor(p_sponsor_name);
  v_bk := private_ctgov.slugify_with_hash(v_norm, 64);

  RETURN QUERY
  INSERT INTO private_ctgov.organizations(
    organization_bk, sponsor_bk, sponsor_name_original, sponsor_name_norm, is_fictitious, source
  )
  VALUES (p_organization_bk, v_bk, p_sponsor_name, v_norm, COALESCE(p_is_fictitious, TRUE), COALESCE(p_source,'ctgov'))
  ON CONFLICT ON CONSTRAINT uq_pctg_orgs_scope_norm_fict
  DO UPDATE SET updated_at = now()
  RETURNING
    private_ctgov.organizations.sponsor_bk,
    private_ctgov.organizations.sponsor_name_original,
    private_ctgov.organizations.sponsor_name_norm,
    private_ctgov.organizations.is_fictitious,
    private_ctgov.organizations.source,
    private_ctgov.organizations.created_at,
    private_ctgov.organizations.updated_at;
END
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_upsert_scenario(p_organization_bk text, p_scenario_bk text, p_scenario_name text, p_scenario_type text, p_description text)
 RETURNS jsonb
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov'
AS $function$
DECLARE
  v_scenario_record RECORD;
BEGIN
  INSERT INTO private_ctgov.scenarios (
    organization_bk, scenario_bk, scenario_name, scenario_type, description
  ) VALUES (
    p_organization_bk, p_scenario_bk, p_scenario_name, p_scenario_type, p_description
  )
  ON CONFLICT (organization_bk, scenario_bk) DO UPDATE SET
    scenario_name = EXCLUDED.scenario_name,
    description = EXCLUDED.description,
    updated_at = now()
  RETURNING * INTO v_scenario_record;
  
  RETURN to_jsonb(v_scenario_record);
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_upsert_scenario_configurations(p_organization_bk text, p_records jsonb)
 RETURNS jsonb
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov'
AS $function$
DECLARE
  v_record JSONB;
  v_inserted_count INT := 0;
  v_updated_count INT := 0;
BEGIN
  FOR v_record IN SELECT * FROM jsonb_array_elements(p_records)
  LOOP
    INSERT INTO private_ctgov.scenario_configurations (
      organization_bk,
      scenario_configuration_bk,
      parent_study_bk,
      parent_scenario_bk,
      start_date,
      end_date,
      target_enrollment,
      target_sites,
      study_status
    ) VALUES (
      p_organization_bk,
      v_record->>'scenario_configuration_bk',
      v_record->>'parent_study_bk',
      v_record->>'parent_scenario_bk',
      v_record->>'start_date',
      v_record->>'end_date',
      v_record->>'target_enrollment',
      (v_record->>'target_sites')::INT,
      v_record->>'study_status'
    )
    ON CONFLICT (organization_bk, parent_study_bk, parent_scenario_bk) DO UPDATE SET
      start_date = EXCLUDED.start_date,
      end_date = EXCLUDED.end_date,
      target_enrollment = EXCLUDED.target_enrollment,
      target_sites = EXCLUDED.target_sites,
      study_status = EXCLUDED.study_status,
      updated_at = now();

    -- GET DIAGNOSTICS is tricky with ON CONFLICT, this is a reliable way to count
    IF FOUND THEN
      v_updated_count := v_updated_count + 1;
    ELSE
      v_inserted_count := v_inserted_count + 1;
    END IF;
  END LOOP;
  
  RETURN jsonb_build_object(
    'status', 'success',
    'inserted_count', v_inserted_count,
    'updated_count', v_updated_count
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.ctgov_upsert_studies(p_organization_bk text, p_records jsonb)
 RETURNS TABLE(action_report jsonb)
 LANGUAGE plpgsql
 SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'extensions'
AS $function$
DECLARE v_ins INT := 0; v_upd INT := 0;
BEGIN
  WITH src AS (
    SELECT
      (x->>'study_bk')::TEXT AS study_bk,
      (x->>'protocol_number')::TEXT AS protocol_number,
      (x->>'study_title')::TEXT AS study_title,
      (x->>'study_short_name')::TEXT AS study_short_name,
      (x->>'therapeutic_area')::TEXT AS therapeutic_area,
      (x->>'phase')::TEXT AS phase,
      COALESCE((x->>'include_in_json')::BOOLEAN, TRUE) AS include_in_json
    FROM jsonb_array_elements(p_records) x
  ), ins AS (
    INSERT INTO private_ctgov.studies (
      organization_bk, study_bk, protocol_number, study_title, study_short_name,
      therapeutic_area, phase, include_in_json
    )
    SELECT p_organization_bk, s.study_bk, s.protocol_number, s.study_title, s.study_short_name,
           s.therapeutic_area, s.phase, s.include_in_json
    FROM src s
    ON CONFLICT (organization_bk, study_bk) DO NOTHING
    RETURNING 1
  )
  SELECT count(*) INTO v_ins FROM ins;

  WITH src AS (
    SELECT
      (x->>'study_bk')::TEXT AS study_bk,
      (x->>'protocol_number')::TEXT AS protocol_number,
      (x->>'study_title')::TEXT AS study_title,
      (x->>'study_short_name')::TEXT AS study_short_name,
      (x->>'therapeutic_area')::TEXT AS therapeutic_area,
      (x->>'phase')::TEXT AS phase,
      COALESCE((x->>'include_in_json')::BOOLEAN, TRUE) AS include_in_json
    FROM jsonb_array_elements(p_records) x
  )
  UPDATE private_ctgov.studies t
  SET
    protocol_number = s.protocol_number,
    study_title     = s.study_title,
    study_short_name= s.study_short_name,
    therapeutic_area= s.therapeutic_area,
    phase           = s.phase,
    include_in_json = s.include_in_json,
    updated_at      = now()
  FROM src s
  WHERE t.organization_bk = p_organization_bk
    AND t.study_bk = s.study_bk;
  GET DIAGNOSTICS v_upd = ROW_COUNT;

  RETURN QUERY SELECT jsonb_build_object(
    'status','success','table','studies',
    'inserted_count', v_ins, 'updated_count', v_upd
  );
END;
$function$
;

CREATE OR REPLACE FUNCTION public.publish_ctgov_organizations(p_organization_bk text)
 RETURNS TABLE(status text, message text, processed_count integer)
 LANGUAGE plpgsql
 SECURITY DEFINER
AS $function$
DECLARE
  v_parent_org_sk BIGINT;
  v_payload JSONB;
  v_org_record JSONB; -- Loop variable will be a JSONB object
  v_total_processed INT := 0;
BEGIN
  -- 1. Resolve the parent (tenant) organization's SK
  v_parent_org_sk := public.resolve_organization_sk_from_bk(p_organization_bk);
  IF v_parent_org_sk IS NULL THEN
    RAISE EXCEPTION 'Parent organization not found for BK: %', p_organization_bk;
  END IF;

  -- 2. Get the materialized payload for organizations
  SELECT payload INTO v_payload
  FROM private_ctgov.materialized_payload
  WHERE organization_bk = p_organization_bk
    AND template_table_name = 'organizations'
    AND stage_number = 1; -- Reading from stage 1 where we cached it

  IF v_payload IS NULL OR jsonb_array_length(v_payload) = 0 THEN
    RETURN QUERY SELECT 'success'::TEXT, 'No organizations payload found to publish.'::TEXT, 0::INT;
    RETURN;
  END IF;

  -- 3. Loop through organizations in the payload and insert/update public.dim_organization
  FOR v_org_record IN SELECT * FROM jsonb_array_elements(v_payload)
  LOOP
    INSERT INTO public.dim_organization (
      organization_bk,
      organization_name,
      organization_type,
      parent_organization_sk,
      is_clerk_managed
    )
    VALUES (
      'CTG-ORG-' || (v_org_record->>'sponsor_bk'), -- Use a CTG prefix
      v_org_record->>'sponsor_name_original',
      'Sponsor'::public.organization_type_enum,
      v_parent_org_sk,
      FALSE
    )
    ON CONFLICT (parent_organization_sk, organization_name) DO UPDATE SET
      organization_type = EXCLUDED.organization_type,
      updated_at = now();

    v_total_processed := v_total_processed + 1;
  END LOOP;

  RETURN QUERY SELECT 'success'::TEXT, 'Organizations published successfully.'::TEXT, v_total_processed::INT;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.publish_ctgov_scenario_configurations(p_organization_bk text)
 RETURNS TABLE(status text, message text, processed_count integer)
 LANGUAGE plpgsql
 SECURITY DEFINER
AS $function$
DECLARE
  v_org_sk BIGINT; v_payload JSONB; v_record JSONB; v_total_processed INT := 0; v_study_sk BIGINT; v_scenario_sk BIGINT;
  v_study_status_enum public.study_status_enum;
  v_start_date_text TEXT;
  v_end_date_text TEXT;
  v_raw_status TEXT;
BEGIN
  v_org_sk := public.resolve_organization_sk_from_bk(p_organization_bk);
  SELECT payload INTO v_payload FROM private_ctgov.materialized_payload WHERE organization_bk = p_organization_bk AND template_table_name = 'scenario_configurations';
  IF v_payload IS NULL THEN RETURN QUERY SELECT 'success'::TEXT, 'No scenario_configurations payload found'::TEXT, 0::INT; RETURN; END IF;
  
  FOR v_record IN SELECT * FROM jsonb_array_elements(v_payload) LOOP
    SELECT s.study_sk INTO v_study_sk FROM public.dim_study s 
    WHERE s.organization_sk = v_org_sk AND s.study_bk = v_record->>'parent_study_bk';

    SELECT sc.scenario_sk INTO v_scenario_sk FROM public.dim_budget_scenario sc WHERE sc.organization_sk = v_org_sk AND sc.scenario_bk = 'CTG-' || (v_record->>'parent_scenario_bk');
    
    IF v_study_sk IS NOT NULL AND v_scenario_sk IS NOT NULL THEN
      v_raw_status := v_record->>'study_status';

      -- [CORRECTION] Intelligent, multi-step mapping logic
      v_study_status_enum := CASE
        -- 1. Direct, case-insensitive matches to the enum
        WHEN lower(v_raw_status) = 'completed' THEN 'Completed'::public.study_status_enum
        WHEN lower(v_raw_status) = 'recruiting' THEN 'Enrolling'::public.study_status_enum
        WHEN lower(v_raw_status) = 'enrolling by invitation' THEN 'Enrolling'::public.study_status_enum
        WHEN lower(v_raw_status) = 'terminated' THEN 'Terminated'::public.study_status_enum
        WHEN lower(v_raw_status) = 'suspended' THEN 'Suspended'::public.study_status_enum
        -- 2. Regex for variations of "Active"
        WHEN v_raw_status ~* 'active' THEN 'Active'::public.study_status_enum
        -- 3. Fallback for all other cases ("Not yet recruiting", "Withdrawn", etc.)
        ELSE 'Planning'::public.study_status_enum
      END;

      v_start_date_text := v_record->>'start_date';
      IF v_start_date_text ~ '^\d{4}-\d{2}$' THEN
        v_start_date_text := v_start_date_text || '-01';
      END IF;

      v_end_date_text := v_record->>'end_date';
      IF v_end_date_text ~ '^\d{4}-\d{2}$' THEN
        v_end_date_text := v_end_date_text || '-01';
      END IF;

      INSERT INTO public.map_scenario_configuration (
        organization_sk, scenario_configuration_bk, study_sk, scenario_sk,
        start_date, end_date, target_enrollment, target_sites, study_status
      )
      VALUES (
        v_org_sk, 
        'CTG-' || (v_record->>'scenario_configuration_bk'), 
        v_study_sk, 
        v_scenario_sk,
        (SELECT CASE WHEN v_start_date_text ~ '^\d{4}-\d{2}-\d{2}$' THEN v_start_date_text::date ELSE NULL END),
        (SELECT CASE WHEN v_end_date_text ~ '^\d{4}-\d{2}-\d{2}$' THEN v_end_date_text::date ELSE NULL END),
        (SELECT CASE WHEN v_record->>'target_enrollment' ~ '^\d+$' THEN (v_record->>'target_enrollment')::integer ELSE NULL END),
        (v_record->>'target_sites')::integer,
        v_study_status_enum
      )
      ON CONFLICT (organization_sk, scenario_configuration_bk) DO UPDATE SET
        start_date = EXCLUDED.start_date,
        end_date = EXCLUDED.end_date,
        target_enrollment = EXCLUDED.target_enrollment,
        target_sites = EXCLUDED.target_sites,
        study_status = EXCLUDED.study_status,
        updated_at = now();
      
      v_total_processed := v_total_processed + 1;
    END IF;
  END LOOP;
  
  RETURN QUERY SELECT 'success'::TEXT, 'Scenario configurations published'::TEXT, v_total_processed::INT;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.publish_ctgov_scenarios(p_organization_bk text)
 RETURNS TABLE(status text, message text, processed_count integer)
 LANGUAGE plpgsql
 SECURITY DEFINER
AS $function$
DECLARE
  v_org_sk BIGINT; v_payload JSONB; v_record JSONB; v_total_processed INT := 0;
BEGIN
  v_org_sk := public.resolve_organization_sk_from_bk(p_organization_bk);
  SELECT payload INTO v_payload FROM private_ctgov.materialized_payload WHERE organization_bk = p_organization_bk AND template_table_name = 'scenarios';
  IF v_payload IS NULL THEN RETURN QUERY SELECT 'success'::TEXT, 'No scenarios payload found'::TEXT, 0::INT; RETURN; END IF;
  FOR v_record IN SELECT * FROM jsonb_array_elements(v_payload) LOOP
    INSERT INTO public.dim_budget_scenario (organization_sk, scenario_bk, scenario_name, scenario_type, description)
    VALUES (v_org_sk, 'CTG-' || (v_record->>'scenario_bk'), v_record->>'scenario_name', (v_record->>'scenario_type')::public.budget_scenario_type_enum, v_record->>'description')
    ON CONFLICT (organization_sk, scenario_bk) DO UPDATE SET scenario_name = EXCLUDED.scenario_name, description = EXCLUDED.description, updated_at = now();
    v_total_processed := v_total_processed + 1;
  END LOOP;
  RETURN QUERY SELECT 'success'::TEXT, 'Scenarios published'::TEXT, v_total_processed::INT;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.publish_ctgov_studies(p_organization_bk text)
 RETURNS TABLE(status text, message text, processed_count integer)
 LANGUAGE plpgsql
 SECURITY DEFINER
AS $function$
DECLARE
  v_org_sk BIGINT;
  v_payload JSONB;
  v_study_record JSONB;
  v_total_processed INT := 0;
  v_phase_string TEXT;
  v_phase_enum public.study_phase_enum;
BEGIN
  -- 1. Resolve organization_sk from the BK
  v_org_sk := public.resolve_organization_sk_from_bk(p_organization_bk);
  IF v_org_sk IS NULL THEN
    RAISE EXCEPTION 'Organization not found for BK: %', p_organization_bk;
  END IF;

  -- 2. Get the materialized payload for studies
  SELECT payload INTO v_payload
  FROM private_ctgov.materialized_payload
  WHERE organization_bk = p_organization_bk
    AND template_table_name = 'studies'
    AND stage_number = 2;

  IF v_payload IS NULL THEN
    -- [CORRECTION] Return a row matching the new TABLE signature
    RETURN QUERY SELECT 'success'::TEXT, 'No studies payload found to publish.'::TEXT, 0::INT;
    RETURN;
  END IF;

  -- 3. Loop through studies and insert into public.dim_study
  FOR v_study_record IN SELECT * FROM jsonb_array_elements(v_payload)
  LOOP
    v_phase_string := v_study_record->>'phase';

    v_phase_enum := CASE v_phase_string
      WHEN 'Phase 1' THEN 'Phase 1'::public.study_phase_enum
      WHEN 'Phase 2' THEN 'Phase 2'::public.study_phase_enum
      WHEN 'Phase 3' THEN 'Phase 3'::public.study_phase_enum
      WHEN 'Phase 4' THEN 'Phase 4'::public.study_phase_enum
      ELSE 'Other'::public.study_phase_enum
    END;

    INSERT INTO public.dim_study (
      organization_sk, study_bk, study_short_name, study_title,
      phase, therapeutic_area, protocol_number
    )
    VALUES (
      v_org_sk, v_study_record->>'study_bk', v_study_record->>'study_short_name',
      v_study_record->>'study_title', v_phase_enum, v_study_record->>'therapeutic_area',
      v_study_record->>'protocol_number'
    )
    ON CONFLICT (organization_sk, study_bk) DO UPDATE SET
      study_title = EXCLUDED.study_title,
      phase = EXCLUDED.phase,
      therapeutic_area = EXCLUDED.therapeutic_area,
      protocol_number = EXCLUDED.protocol_number,
      updated_at = now();

    v_total_processed := v_total_processed + 1;
  END LOOP;

  -- [CORRECTION] Return a row matching the new TABLE signature
  RETURN QUERY SELECT 'success'::TEXT, 'Studies published successfully.'::TEXT, v_total_processed::INT;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.resolve_organization_sk_from_bk(p_organization_bk text)
 RETURNS bigint
 LANGUAGE plpgsql
 STABLE SECURITY DEFINER
 SET search_path TO 'public', 'private_ctgov', 'pg_temp'
AS $function$
DECLARE
  v_org_sk BIGINT;
BEGIN
  IF p_organization_bk IS NULL OR btrim(p_organization_bk) = '' THEN
    RAISE EXCEPTION 'p_organization_bk is required';
  END IF;
  -- Prefer the tenant/root organization (no parent)
  SELECT o.organization_sk
  INTO v_org_sk
  FROM public.dim_organization o
  WHERE o.organization_bk = p_organization_bk
    AND (o.parent_organization_sk IS NULL OR o.parent_organization_sk = 0)
    AND o.is_deleted = FALSE
  ORDER BY o.organization_sk
  LIMIT 1;
  IF v_org_sk IS NULL THEN
    -- Fallback: any matching row (take first), still useful for isolation checks
    SELECT o.organization_sk
    INTO v_org_sk
    FROM public.dim_organization o
    WHERE o.organization_bk = p_organization_bk
      AND o.is_deleted = FALSE
    ORDER BY (o.parent_organization_sk IS NULL) DESC, o.organization_sk
    LIMIT 1;
  END IF;
  IF v_org_sk IS NULL THEN
    RAISE EXCEPTION 'Organization not found for organization_bk=%', p_organization_bk;
  END IF;
  RETURN v_org_sk;
END;
$function$
;

CREATE OR REPLACE FUNCTION public.trg_fn_generate_ctf_bk_on_adopt()
 RETURNS trigger
 LANGUAGE plpgsql
AS $function$
BEGIN
  IF TG_TABLE_NAME = 'dim_study' THEN
    NEW.study_bk := 'CTF-STUDY-' || extensions.uuid_generate_v4()::text;
  ELSIF TG_TABLE_NAME = 'dim_organization' THEN
    NEW.organization_bk := 'CTF-ORG-' || extensions.uuid_generate_v4()::text;
  ELSIF TG_TABLE_NAME = 'dim_budget_scenario' THEN -- <<< ADD THIS BLOCK
    NEW.scenario_bk := 'CTF-SCEN-' || extensions.uuid_generate_v4()::text;
  ELSIF TG_TABLE_NAME = 'map_scenario_configuration' THEN -- <<< ADD THIS BLOCK
    NEW.scenario_configuration_bk := 'CTF-CFG-' || extensions.uuid_generate_v4()::text;
  END IF;
  RETURN NEW;
END;
$function$
;

CREATE TRIGGER trg_adopt_scenario BEFORE UPDATE ON public.dim_budget_scenario FOR EACH ROW WHEN (((new.scenario_bk IS NULL) AND (old.scenario_bk IS NOT NULL))) EXECUTE FUNCTION trg_fn_generate_ctf_bk_on_adopt();

CREATE TRIGGER trg_adopt_organization BEFORE UPDATE ON public.dim_organization FOR EACH ROW WHEN (((new.organization_bk IS NULL) AND (old.organization_bk IS NOT NULL))) EXECUTE FUNCTION trg_fn_generate_ctf_bk_on_adopt();

CREATE TRIGGER trg_adopt_study BEFORE UPDATE ON public.dim_study FOR EACH ROW WHEN (((new.study_bk IS NULL) AND (old.study_bk IS NOT NULL))) EXECUTE FUNCTION trg_fn_generate_ctf_bk_on_adopt();

CREATE TRIGGER trg_adopt_scenario_configuration BEFORE UPDATE ON public.map_scenario_configuration FOR EACH ROW WHEN (((new.scenario_configuration_bk IS NULL) AND (old.scenario_configuration_bk IS NOT NULL))) EXECUTE FUNCTION trg_fn_generate_ctf_bk_on_adopt();
'''


python server:
'''
# C:\Users\jorge\OneDrive\Documents\GitHub\ctgov-seeder-mcp-python\ctgov_seeder_mcp\server.py

import os
import re
import json
import time
import hashlib
import logging
import functools
import inspect
from typing import Any, Optional, Union, Dict, List, Tuple, Callable, Awaitable, TypeVar, ParamSpec, cast
from collections import Counter
import asyncio
from concurrent.futures import ThreadPoolExecutor

import httpx
import pandas as pd
from dotenv import load_dotenv
from mcp.server.fastmcp import FastMCP
from enum import Enum
from pydantic import BaseModel, Field
try:
    from pytrials.client import ClinicalTrials
except Exception:  # pragma: no cover
    ClinicalTrials = None

from ctgov_seeder_mcp.audit import with_audit

def _mk_model_config(**kwargs: Any) -> Any:
    return kwargs

# ------------------------------------------------------------------------------
# Environment and Logging
# ------------------------------------------------------------------------------

load_dotenv()

LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO").upper()
logging.basicConfig(level=LOG_LEVEL, format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
logger = logging.getLogger("ctgov_seeder_mcp")

SUPABASE_URL = os.getenv("SUPABASE_URL", "").rstrip("/")
SUPABASE_SERVICE_ROLE_KEY = os.getenv("SUPABASE_SERVICE_ROLE_KEY", "")
REQUEST_TIMEOUT_SECONDS = float(os.getenv("REQUEST_TIMEOUT_SECONDS", "30"))

if not SUPABASE_URL or not SUPABASE_SERVICE_ROLE_KEY:
    logger.warning("SUPABASE_URL or SUPABASE_SERVICE_ROLE_KEY not set. RPC calls will fail until configured.")

# Global defaults
# Create a dedicated thread pool for our background jobs
_JOB_EXECUTOR = ThreadPoolExecutor(max_workers=4)

# ------------------------------------------------------------------------------
# MCP Server
# ------------------------------------------------------------------------------

mcp = FastMCP("CTGov Seeder MCP (Python)")

# ------------------------------------------------------------------------------
# Central Resource Store
# ------------------------------------------------------------------------------

_RESOURCES = {
    "ctgov://cheatsheet/columns": {
        "name": "CT.gov Column Cheatsheet",
        "description": "A quick reference for common CT.gov data columns.",
        "content": (
            "# CT.gov Column Cheatsheet\n\n"
            "Common columns returned from `ctgov_find_studies_*` tools and available for querying:\n\n"
            "- `NCT Number`: The unique clinical trial identifier.\n"
            "- `Study Title`: The official title of the study.\n"
            "- `Study Status`: The current recruitment status (e.g., Recruiting, Completed).\n"
            "- `Phases`: The clinical trial phase (e.g., Phase 1, Phase 2).\n"
            "- `Sponsor`: The primary organization responsible for the trial.\n"
            "- `Start Date`: The date the study began.\n"
            "- `Primary Completion Date`: The primary completion date of the study.\n"
            "- `Enrollment`: The target or actual enrollment number.\n"
        ),
        "mimeType": "text/markdown"
    },
    "seeding://guide/workflow": {
        "name": "Seeding Workflow Guide",
        "description": "A step-by-step guide for seeding data, including handling large sponsors.",
        "content": (
            "# Recommended Seeding Workflow\n\n"
            "## Standard Workflow (for small sponsors)\n\n"
            "1.  **Discover**: Use `ctgov_find_sponsors` to find the exact sponsor name and study count.\n"
            "2.  **Confirm**: Ask the user to confirm the sponsor to proceed with.\n"
            "3.  **Stage**: Use `stage_fictitious_organization`, `ctgov_find_studies_by_sponsor`, and `ctgov_stage_studies_by_nct` to stage the data.\n"
            "4.  **Publish**: Use the `publish_*` tools to move data to the public schema.\n\n"
            "## Large-Scale Workflow (for sponsors with >1000 studies)\n\n"
            "When a sponsor has more than 1000 studies, the standard discovery tools are insufficient. Use this asynchronous workflow:\n\n"
            "1.  **Start Discovery Job**: Call the `start_discovery_job(search_expression)` tool. This starts a background job to find ALL studies, bypassing the 1000-item API limit. The tool returns a `job_id` immediately.\n"
            "2.  **Monitor Job**: Periodically call `get_job_status(job_id)` to check the job's progress. The status will be 'running' or 'completed'.\n"
            "3.  **Retrieve Results**: Once the job is 'completed', the `result` field in the status will contain the full list of NCT IDs.\n"
            "4.  **Confirm and Stage**: Show the user the total count of studies found. Ask for confirmation and how many they wish to stage (e.g., 'all 13,000' or 'the first 100').\n"
            "5.  **Start Staging Job**: Use the `start_staging_studies_job` tool with the list of NCT IDs retrieved from the discovery job.\n"
        ),
        "mimeType": "text/markdown"
    },
    "seeding://guide/bk_naming": {
        "name": "Business Key (BK) Naming Guide",
        "description": "Summary of the CTG-* and CTF-* business key naming strategy.",
        "content": (
            "# Business Key (BK) Naming Guide\n\n"
            "- **`CTG-` Prefix**: Represents temporary, staged data seeded from ClinicalTrials.gov. These records are owned by the system and can be cleared.\n"
            "- **`CTF-` Prefix**: Represents permanent, adopted data. When a user 'adopts' a `CTG-` record, its key is changed to a `CTF-` key, signifying user ownership.\n"
            "- **Idempotency**: Staging and publishing operations are idempotent. Re-running them will update existing `CTG-` records but will not create duplicates.\n"
        ),
        "mimeType": "text/markdown"
    }
}

# ------------------------------------------------------------------------------
# Supabase RPC Client
# ------------------------------------------------------------------------------

class SupabaseRPC:
    def __init__(self, url: str, service_role_key: str, timeout_seconds: float = 30.0):
        self.base_url = f"{url}/rest/v1"
        self.key = service_role_key
        self.timeout = httpx.Timeout(timeout_seconds)

    async def call(self, function: str, params: dict) -> Any:
        headers = {
            "apikey": self.key, "Authorization": f"Bearer {self.key}", "Content-Type": "application/json",
        }
        url = f"{self.base_url}/rpc/{function}"
        logger.debug(f"SupabaseRPC.call start function={function}")
        async with httpx.AsyncClient(timeout=self.timeout) as client:
            resp = await client.post(url, headers=headers, json=params)
            try:
                resp.raise_for_status()
            except httpx.HTTPStatusError as e:
                detail = None
                try: detail = resp.json()
                except Exception: detail = resp.text
                _audit_event(
                    action="rpc_error", status="error", function=function,
                    http_status=resp.status_code, detail=_redact_value(detail),
                )
                logger.error(f"RPC {function} failed: {e} | Detail: {detail}")
                raise
            try: out = resp.json()
            except Exception: out = resp.text
            logger.debug(f"SupabaseRPC.call success function={function}")
            return out

_supabase: Optional[SupabaseRPC] = None

def get_supabase() -> SupabaseRPC:
    global _supabase
    if _supabase is None:
        if not SUPABASE_URL or not SUPABASE_SERVICE_ROLE_KEY:
            raise RuntimeError("SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set to use this server.")
        _supabase = SupabaseRPC(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, REQUEST_TIMEOUT_SECONDS)
    return _supabase

# ------------------------------------------------------------------------------
# Internal Helper Functions
# ------------------------------------------------------------------------------

_ORG_BK_PATTERN = re.compile(r"^[A-Za-z0-9][A-Za-z0-9:_\-\.\+]{1,127}$")

def _validate_org_bk(org_bk: str) -> None:
    if not isinstance(org_bk, str) or not org_bk:
        raise ValueError("organization_bk must be a non-empty string.")
    if len(org_bk) > 128 or _ORG_BK_PATTERN.match(org_bk) is None:
        raise ValueError("organization_bk has invalid format. Allowed: [A-Za-z0-9:_-.+], length <= 128.")

def _redact_value(value: Any) -> Any:
    if value is None: return None
    if isinstance(value, str):
        if len(value) <= 128: return value
        return f"{value[:64]}…{value[-16:]} (len={len(value)})"
    if isinstance(value, (int, float, bool)): return value
    if isinstance(value, dict):
        try: size, keys = len(json.dumps(value, ensure_ascii=False)), list(value.keys())[:12]
        except Exception: size, keys = None, list(value.keys())[:12]
        return {"type": "object", "keys": keys, "approx_size": size}
    if isinstance(value, list):
        try: size = len(json.dumps(value, ensure_ascii=False))
        except Exception: size = None
        sample_type = type(value[0]).__name__ if value else None
        return {"type": "array", "length": len(value), "approx_size": size, "sample_type": sample_type}
    return _redact_value(str(value))

P = ParamSpec("P")
R = TypeVar("R")

def log_tool(func: Callable[P, Awaitable[R]]) -> Callable[P, Awaitable[R]]:
    @functools.wraps(func)
    async def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        name = getattr(func, "__name__", "unknown_tool")
        logger.info(f"tool_call start name={name}")
        try:
            return await func(*args, **kwargs)
        finally:
            logger.info(f"tool_call end name={name}")
    return wrapper

def _ok_result(data: Dict[str, Any]) -> Dict[str, Any]:
    return {"status": "ok", "data": data}

def _err_result(message: str, detail: Any = None, error_type: Optional[str] = None, extra: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
    out: Dict[str, Any] = {"status": "error", "message": message}
    if error_type: out["error_type"] = error_type
    if detail is not None: out["detail"] = _redact_value(detail)
    if isinstance(extra, dict): out.update(extra)
    return out

def _summarize_json(obj: Any) -> Dict[str, Any]:
    try:
        if isinstance(obj, dict): return {"type": "dict", "keys": list(obj.keys())[:12]}
        if isinstance(obj, list): return {"type": "list", "length": len(obj)}
        return {"type": "scalar", "repr": str(obj)[:256]}
    except Exception: return {"type": "unknown"}

def _coerce_org_sk_from_res(res: Any) -> int:
    if isinstance(res, int): return int(res)
    if isinstance(res, dict):
        for key in ("organization_sk", "org_sk"):
            if key in res and isinstance(res[key], int): return int(res[key])
        for v in res.values():
            if isinstance(v, int): return int(v)
    if isinstance(res, list) and res:
        first = res[0]
        if isinstance(first, int): return int(first)
        if isinstance(first, dict):
            for key in ("organization_sk", "org_sk"):
                if key in first and isinstance(first[key], int): return int(first[key])
            for v in first.values():
                if isinstance(v, int): return int(v)
    raise ValueError("Could not coerce organization_sk from RPC result.")

def _coerce_org_row(row_like: Any) -> Optional[Dict[str, Any]]:
    if row_like is None: return None
    if isinstance(row_like, dict): return row_like
    if isinstance(row_like, list):
        for it in row_like:
            if isinstance(it, dict): return it
        return None
    return None

def _audit_event(**kwargs: Any) -> None:
    try:
        logger.debug(f"audit_event: {json.dumps(kwargs, ensure_ascii=False)[:2048]}")
    except Exception:
        logger.debug(f"audit_event: {kwargs}")

def _slug(text: Optional[str]) -> Optional[str]:
    if not text: return None
    try:
        s = str(text).strip().lower()
        s = re.sub(r"[^a-z0-9]+", "-", s)
        s = re.sub(r"-{2,}", "-", s).strip("-")
        return s or None
    except Exception: return None

def _normalize_study_record_from_row(row: pd.Series) -> Dict[str, Any]:
    nct_id = str((row.get("NCT Number") or "")).strip().upper()
    sponsor_name = row.get("Sponsor") or "unknown"
    return {
        "study_bk": f"CTG-STUDY-{_slug(sponsor_name)}-{nct_id}",
        "study_short_name": nct_id,
        "study_title": row.get("Study Title"),
        "phase": str(row.get("Phases") or "").replace("PHASE", "Phase ").replace("|", "/").strip(),
    }

def _map_ctgov_status_to_enum(ctgov_status: Optional[str]) -> str:
    if not isinstance(ctgov_status, str):
        return "Planning"
    status = ctgov_status.lower()
    mapping = {
        "completed": "Completed", "recruiting": "Enrolling",
        "enrolling by invitation": "Enrolling", "active, not recruiting": "Active",
        "terminated": "Terminated", "suspended": "Suspended",
    }
    if status in mapping: return mapping[status]
    return "Planning"

async def _run_discovery_job(job_id: str, search_expression: str):
    """
    Background worker that uses the CT.gov V2 API directly to find all studies,
    handling modern token-based pagination.
    """
    logger.info(f"Starting V2 discovery job {job_id} for expression: '{search_expression}'")
    sp = get_supabase()
    await sp.call("ctgov_update_job_status", {"p_job_id": job_id, "p_status": "running"})

    all_nct_ids = []
    next_page_token = None
    base_url = "https://clinicaltrials.gov/api/v2/studies"
    
    try:
        async with httpx.AsyncClient(timeout=60.0) as client:
            while True:
                # [CORRECTION] Use the correct V2 API parameters: query.spons, pageSize, and pageToken
                params = {
                    "query.spons": search_expression,
                    "fields": "NCTId",
                    "pageSize": 1000
                }
                if next_page_token:
                    params["pageToken"] = next_page_token

                logger.info(f"Job {job_id}: Fetching page with token: {next_page_token}")
                response = await client.get(base_url, params=params)
                response.raise_for_status()
                data = response.json()

                studies = data.get('studies', [])
                for study in studies:
                    # The NCT ID is nested in the V2 response structure
                    all_nct_ids.append(study['protocolSection']['identificationModule']['nctId'])

                next_page_token = data.get('nextPageToken')
                if not next_page_token:
                    # This was the last page
                    break

        result_payload = {
            "search_expression": search_expression,
            "total_studies_found": len(all_nct_ids),
            "nct_ids": all_nct_ids
        }
        await sp.call("ctgov_update_job_status", {
            "p_job_id": job_id, "p_status": "completed", "p_result": result_payload
        })
        logger.info(f"V2 Discovery job {job_id} completed. Found {len(all_nct_ids)} studies.")

    except Exception as e:
        import traceback
        logger.error(f"V2 Discovery job {job_id} failed: {e}\n{traceback.format_exc()}")
        await sp.call("ctgov_update_job_status", {
            "p_job_id": job_id, "p_status": "failed", "p_result": {"error": str(e)}
        })

# ------------------------------------------------------------------------------
# MCP Resource Tools
# ------------------------------------------------------------------------------

@mcp.tool()
@log_tool
@with_audit(validate_org=False)
async def list_resources() -> dict:
    """
    Lists all available documentation resources provided by this server.
    Use the `get_resource` tool with a URI from this list to read the content.
    """
    resource_list = [
        {"uri": uri, "name": data["name"], "description": data["description"]}
        for uri, data in _RESOURCES.items()
    ]
    return _ok_result({"resources": resource_list})

@mcp.tool()
@log_tool
@with_audit(validate_org=False)
async def get_resource(uri: str) -> dict:
    """
    Retrieves the content of a server-defined documentation resource by its URI.
    Call `list_resources` to discover available URIs.
    """
    resource_data = _RESOURCES.get(uri)
    if resource_data:
        return _ok_result({
            "uri": uri, "name": resource_data["name"],
            "content": resource_data["content"], "mimeType": resource_data.get("mimeType", "text/plain")
        })
    else:
        return _err_result("resource_not_found", f"Resource with URI '{uri}' not found.")

# ------------------------------------------------------------------------------
# Verification and Observability Tools
# ------------------------------------------------------------------------------

DEFAULT_ISOLATION_TABLES: Dict[str, str] = {
    "dim_organization": "organization_bk", "dim_study": "study_bk",
    "dim_budget_scenario": "scenario_bk", "map_scenario_configuration": "scenario_configuration_bk",
}

async def _postgrest_count_flexible(table: str, bk_col: str, tenancy_col: str, org_sk: int, other_orgs: bool = False) -> int:
    if not SUPABASE_URL or not SUPABASE_SERVICE_ROLE_KEY:
        raise RuntimeError("SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set")
    base = f"{SUPABASE_URL}/rest/v1/{table}"
    org_filter = f"neq.{org_sk}" if other_orgs else f"eq.{org_sk}"
    params = {
        "select": "*", "limit": "1", tenancy_col: org_filter,
        "is_deleted": "eq.false", bk_col: "like.CTG-%",
    }
    headers = {
        "apikey": SUPABASE_SERVICE_ROLE_KEY, "Authorization": f"Bearer {SUPABASE_SERVICE_ROLE_KEY}", "Prefer": "count=exact",
    }
    async with httpx.AsyncClient(timeout=httpx.Timeout(REQUEST_TIMEOUT_SECONDS)) as client:
        resp = await client.get(base, params=params, headers=headers)
        resp.raise_for_status()
        cr = resp.headers.get("Content-Range") or resp.headers.get("content-range")
        if cr and "/" in cr:
            try: return int(cr.split("/")[-1])
            except Exception: pass
        return 1 if resp.json() else 0

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def verify_org_isolation(organization_bk: str, tables: Optional[List[str]] = None) -> dict:
    """
    Validates multi-tenant isolation for CTG-% data by counting records for the specified
    organization versus all other organizations. `other_orgs_count` should be 0.
    """
    org_bk = organization_bk
    sp = get_supabase()
    res = await sp.call("resolve_organization_sk_from_bk", {"p_organization_bk": org_bk})
    org_sk = _coerce_org_sk_from_res(res)
    selected = tables or list(DEFAULT_ISOLATION_TABLES.keys())
    counts: Dict[str, Dict[str, int]] = {}
    for t in selected:
        bk_col = DEFAULT_ISOLATION_TABLES.get(t)
        if not bk_col:
            counts[t] = {"org_count": -1, "other_orgs_count": -1}
            continue
        tenancy_col = "parent_organization_sk" if t == "dim_organization" else "organization_sk"
        try:
            org_count = await _postgrest_count_flexible(t, bk_col, tenancy_col, org_sk, other_orgs=False)
        except Exception as e:
            org_count = -1
            logger.warning(f"verify_org_isolation: failed org_count for {t}: {e}")
        try:
            other_count = await _postgrest_count_flexible(t, bk_col, tenancy_col, org_sk, other_orgs=True)
        except Exception as e:
            other_count = -1
            logger.warning(f"verify_org_isolation: failed other_orgs_count for {t}: {e}")
        counts[t] = {"org_count": org_count, "other_orgs_count": other_count}
    return _ok_result({"organization_sk": org_sk, "tables_checked": selected, "counts": counts})

# ------------------------------------------------------------------------------
# CT.gov Discovery Tools
# ------------------------------------------------------------------------------

_CT_CLIENT: Any = None

def _get_ct_client() -> Any:
    global _CT_CLIENT
    if ClinicalTrials is None:
        raise RuntimeError("pytrials is not available; install 'pytrials' to use CT.gov tools.")
    if _CT_CLIENT is not None: return _CT_CLIENT
    try:
        _CT_CLIENT = ClinicalTrials()
        return _CT_CLIENT
    except Exception as e:
        logger.error(f"Failed to initialize ClinicalTrials client: {e}")
        raise

@mcp.tool()
@log_tool
@with_audit(validate_org=False)
async def ctgov_find_sponsors(search_expression: str) -> dict:
    """
    Finds unique sponsor names matching a search expression and provides their study counts.
    This is a fast operation that does not fetch full study details.
    """
    client = _get_ct_client()
    try:
        study_fields_result = client.get_study_fields(
            search_expr=search_expression, fields=["Sponsor"], max_studies=1000, fmt="csv"
        )
        if not study_fields_result or len(study_fields_result) < 2:
            return _ok_result({"sponsors": [], "message": "No sponsors found."})
        
        header = study_fields_result[0]
        sponsor_names = [row[header.index("Sponsor")] for row in study_fields_result[1:]]
        relevant_sponsor_names = [name for name in sponsor_names if search_expression.lower() in name.lower()]
        sponsor_counts = Counter(relevant_sponsor_names)
        sponsors_summary = [{"Sponsor": name, "N Studies": count} for name, count in sponsor_counts.items()]
        sponsors_summary.sort(key=lambda x: x["N Studies"], reverse=True)
        return _ok_result({"sponsors": sponsors_summary})
    except Exception as e:
        return _err_result("find_sponsors_failed", str(e))

@mcp.tool()
@log_tool
@with_audit(validate_org=False)
async def ctgov_find_studies_by_sponsor(sponsor_name: str, max_results: Optional[int] = 100) -> dict:
    """
    Finds studies for a specific sponsor via an exact match on the sponsor name.
    Returns a summary list of matching studies.
    """
    client = _get_ct_client()
    expr = f'AREA[LeadSponsorName]"{sponsor_name}"'
    fetched = client.get_full_studies(search_expr=expr, max_studies=int(max_results or 100))
    df = pd.DataFrame.from_records(fetched[1:], columns=fetched[0])
    out: List[Dict[str, Any]] = []
    if not df.empty:
        for _, r in df.iterrows():
            nct_id = r.get("NCT Number", "")
            out.append({
                "nct_id": nct_id, "title": r.get("Study Title") or r.get("Official Title") or "",
                "status": (r.get("Study Status") or "").title(), "phase": str(r.get("Phases") or "").replace("PHASE", "Phase ").replace("|", "/").strip(),
                "source_url": r.get("Study URL", f"https://clinicaltrials.gov/study/{nct_id}") if nct_id else None,
            })
    return _ok_result({"sponsor_name": sponsor_name, "count": len(out), "results": out})

# ------------------------------------------------------------------------------
# Staging Tools
# ------------------------------------------------------------------------------

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def stage_fictitious_organization(organization_bk: str, sponsor_name: str) -> dict:
    """
    Creates a single fictitious organization record in the private_ctgov schema
    without staging any associated studies.
    """
    sp = get_supabase()
    org_res = await sp.call("ctgov_upsert_organization", {
        "p_organization_bk": organization_bk, "p_sponsor_name": sponsor_name,
        "p_is_fictitious": True, "p_source": "ctgov-manual",
    })
    org_row = _coerce_org_row(org_res)
    if not isinstance(org_row, dict):
        return _err_result("organization_upsert_failed", detail=org_res)
    return _ok_result({"message": "Fictitious organization staged successfully.", "organization": org_row})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def stage_ctgov_scenario(organization_bk: str) -> dict:
    """
    Stages a default, system-generated scenario for ClinicalTrials.gov data.
    This operation is idempotent.
    """
    sp = get_supabase()
    result = await sp.call("ctgov_upsert_scenario", {
        "p_organization_bk": organization_bk, "p_scenario_bk": "ctg-baseline-scenario-v1",
        "p_scenario_name": "CT.gov Baseline", "p_scenario_type": "Forecast",
        "p_description": "A baseline scenario automatically generated from public ClinicalTrials.gov data."
    })
    return _ok_result({"message": "Default CT.gov scenario staged successfully.", "scenario": result})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def start_staging_studies_job(organization_bk: str, nct_ids: List[str]) -> dict:
    """
    Starts a background job to fetch and stage a large number of studies.
    This tool returns immediately with a job_id. Use the `get_job_status`
    tool to check the progress and get the result.
    """
    sp = get_supabase()
    
    # Create a job record in the database
    job_payload = {"nct_ids": nct_ids}
    job_id = await sp.call("ctgov_create_job", {
        "p_organization_bk": organization_bk,
        "p_job_type": "stage_studies",
        "p_payload": job_payload
    })

    if not job_id:
        return _err_result("job_creation_failed", "Could not create a job in the database.")

    # Start the background task
    task = asyncio.create_task(_run_staging_job(job_id, organization_bk, nct_ids))
    _BACKGROUND_TASKS[job_id] = task

    return _ok_result({
        "message": f"Started background job to stage {len(nct_ids)} studies.",
        "job_id": job_id
    })

@mcp.tool()
@log_tool
@with_audit(validate_org=False)
async def get_job_status(job_id: str) -> dict:
    """
    Retrieves the status and result of a background job.
    """
    sp = get_supabase()
    status = await sp.call("ctgov_get_job_status", {"p_job_id": job_id})
    
    if not status:
        return _err_result("not_found", f"Job with ID {job_id} not found.")
        
    return _ok_result(status)

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def link_staged_studies_to_scenario(organization_bk: str, scenario_bk: str) -> dict:
    """
    Links all studies in private_ctgov.studies to the specified scenario_bk, enriching
    the link with detailed data fetched from ClinicalTrials.gov.
    """
    try:
        sp = get_supabase()
        client = _get_ct_client()
        staged_studies = await sp.call("ctgov_list_studies", {"p_organization_bk": organization_bk})
        if not staged_studies:
            return _ok_result({"message": "No staged studies found to link.", "linked_count": 0})
        nct_ids = [study.get('study_short_name') for study in staged_studies if study.get('study_short_name')]
        if not nct_ids:
            return _ok_result({"message": "No NCT IDs found in staged studies.", "linked_count": 0})
        
        all_studies_data = []
        for nct_id in nct_ids:
            study_data = client.get_full_studies(search_expr=nct_id, max_studies=1)
            if len(study_data) > 1: all_studies_data.append(study_data[1])
        if not all_studies_data:
            return _err_result("link_studies_failed", "Failed to fetch any study details from ClinicalTrials.gov.")
        
        header = client.get_full_studies(search_expr=nct_ids[0], max_studies=1)[0]
        full_studies_df = pd.DataFrame(all_studies_data, columns=header)
        
        configs_to_create = []
        for _, row in full_studies_df.iterrows():
            study_bk = next((s['study_bk'] for s in staged_studies if s['study_short_name'] == row.get('NCT Number')), None)
            if not study_bk: continue
            locations = row.get('Locations')
            site_count = len(locations.split('|')) if isinstance(locations, str) and locations else 0
            configs_to_create.append({
                "scenario_configuration_bk": f"CFG-{study_bk}-{scenario_bk}", "parent_study_bk": study_bk,
                "parent_scenario_bk": scenario_bk, "start_date": row.get('Start Date'),
                "end_date": row.get('Primary Completion Date'),
                "target_enrollment": str(row.get('Enrollment')) if pd.notna(row.get('Enrollment')) else None,
                "target_sites": site_count, "study_status": row.get('Study Status')
            })
        
        result = await sp.call("ctgov_upsert_scenario_configurations", {"p_organization_bk": organization_bk, "p_records": configs_to_create})
        return _ok_result({
            "message": f"Successfully linked and enriched {len(configs_to_create)} studies for scenario '{scenario_bk}'.",
            "upsert_result": result
        })
    except Exception as e:
        return _err_result("link_studies_failed", str(e))

# ------------------------------------------------------------------------------
# Materialization (Refresh) Tools
# ------------------------------------------------------------------------------

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def refresh_studies_payload(organization_bk: str) -> dict:
    """Materializes a JSON payload for staged studies."""
    sp = get_supabase()
    result = await sp.call("ctgov_refresh_studies_payload", {"p_organization_bk": organization_bk})
    return _ok_result({"refresh_result": result})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def refresh_organizations_payload(organization_bk: str) -> dict:
    """Materializes a JSON payload for staged organizations."""
    sp = get_supabase()
    result = await sp.call("ctgov_refresh_organizations_payload", {"p_organization_bk": organization_bk})
    return _ok_result({"refresh_result": result})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def refresh_scenarios_payload(organization_bk: str) -> dict:
    """Materializes a JSON payload for staged scenarios."""
    sp = get_supabase()
    result = await sp.call("ctgov_refresh_scenarios_payload", {"p_organization_bk": organization_bk})
    return _ok_result({"refresh_result": result})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def refresh_scenario_configurations_payload(organization_bk: str) -> dict:
    """Materializes a JSON payload for staged scenario configurations."""
    sp = get_supabase()
    result = await sp.call("ctgov_refresh_scenario_configurations_payload", {"p_organization_bk": organization_bk})
    return _ok_result({"refresh_result": result})

# ------------------------------------------------------------------------------
# Publishing Tools
# ------------------------------------------------------------------------------

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def publish_studies(organization_bk: str) -> dict:
    """Publishes staged studies from the materialized payload to the public schema."""
    sp = get_supabase()
    result = await sp.call("publish_ctgov_studies", {"p_organization_bk": organization_bk})
    return _ok_result({"publish_result": result})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def publish_organizations(organization_bk: str) -> dict:
    """Publishes staged organizations from the materialized payload to the public schema."""
    sp = get_supabase()
    result = await sp.call("publish_ctgov_organizations", {"p_organization_bk": organization_bk})
    return _ok_result({"publish_result": result})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def publish_scenarios(organization_bk: str) -> dict:
    """Publishes staged scenarios from the materialized payload to the public schema."""
    sp = get_supabase()
    result = await sp.call("publish_ctgov_scenarios", {"p_organization_bk": organization_bk})
    return _ok_result({"publish_result": result})

@mcp.tool()
@log_tool
@with_audit(validate_org=True)
async def publish_scenario_configurations(organization_bk: str) -> dict:
    """Publishes staged scenario configurations from the materialized payload to the public schema."""
    sp = get_supabase()
    result = await sp.call("publish_ctgov_scenario_configurations", {"p_organization_bk": organization_bk})
    return _ok_result({"publish_result": result})

# ------------------------------------------------------------------------------
# Adoption Tools (Internal, not exposed via @mcp.tool)
# ------------------------------------------------------------------------------

@log_tool
@with_audit(validate_org=True)
async def adopt_organization(organization_bk: str) -> dict:
    """Adopts published CTG-prefixed organizations into permanent CTF- records."""
    sp = get_supabase()
    res = await sp.call("adopt_ctgov_organizations", {"p_organization_bk": organization_bk})
    return _ok_result({"adopt_result_summary": _summarize_json(res)})

@log_tool
@with_audit(validate_org=True)
async def adopt_studies(organization_bk: str) -> dict:
    """Adopts published CTG-prefixed studies into permanent CTF- records."""
    sp = get_supabase()
    res = await sp.call("adopt_ctgov_public_data", {"p_organization_bk": organization_bk, "p_tables": ["dim_study"]})
    return _ok_result({"adopt_result_summary": _summarize_json(res)})

# Add near the top with other imports
import asyncio

# This dictionary will hold our running background tasks
_BACKGROUND_TASKS = {}

def _blocking_staging_work(job_id: str, organization_bk: str, nct_ids: List[str]):
    """
    This function contains the actual blocking I/O work.
    It will be run in a separate thread.
    """
    logger.info(f"Thread for job {job_id} started.")
    client = _get_ct_client()
    records: List[Dict[str, Any]] = []
    missing: List[str] = []
    
    try:
        header = client.get_full_studies(search_expr=nct_ids[0], max_studies=1)[0]
    except Exception as e:
        logger.error(f"Job {job_id}: Could not retrieve API header: {e}")
        # Cannot proceed without the header
        return {"processed_count": 0, "missing_count": len(nct_ids), "missing_nct_ids": nct_ids, "error": "Header fetch failed"}

    for nct in nct_ids:
        try:
            fetched = client.get_full_studies(search_expr=str(nct).upper(), max_studies=1)
            if len(fetched) > 1:
                row_series = pd.Series(fetched[1], index=header)
                records.append(_normalize_study_record_from_row(row_series))
            else:
                missing.append(nct)
        except Exception as e:
            logger.error(f"Job {job_id}: Error processing study {nct}: {e}")
            missing.append(nct)
    
    return {"records": records, "missing": missing}


async def _run_staging_job(job_id: str, organization_bk: str, nct_ids: List[str]):
    """
    This is the async wrapper that manages the background job.
    """
    logger.info(f"Starting background job {job_id} for {len(nct_ids)} studies.")
    sp = get_supabase()
    
    await sp.call("ctgov_update_job_status", {"p_job_id": job_id, "p_status": "running"})

    try:
        # Run the blocking work in a separate thread
        result = await asyncio.to_thread(_blocking_staging_work, job_id, organization_bk, nct_ids)
        
        records = result.get("records", [])
        missing = result.get("missing", [])

        if records:
            await sp.call("ctgov_upsert_studies", {"p_organization_bk": organization_bk, "p_records": records})

        result_payload = {"processed_count": len(records), "missing_count": len(missing), "missing_nct_ids": missing}
        await sp.call("ctgov_update_job_status", {
            "p_job_id": job_id,
            "p_status": "completed",
            "p_result": result_payload
        })
        logger.info(f"Background job {job_id} completed.")

    except Exception as e:
        import traceback
        logger.error(f"Job {job_id} failed catastrophically: {e}\n{traceback.format_exc()}")
        await sp.call("ctgov_update_job_status", {
            "p_job_id": job_id,
            "p_status": "failed",
            "p_result": {"error": str(e)}
        })


@mcp.tool()
@log_tool
@with_audit(validate_org=False)
async def start_discovery_job(organization_bk: str, search_expression: str) -> dict:
    """
    Starts a background job to discover ALL studies for a given sponsor search expression,
    using the V2 API to bypass the 1000-item limit by handling pagination.
    Returns immediately with a job_id. Use `get_job_status` to check progress.
    """
    sp = get_supabase()
    
    job_id = await sp.call("ctgov_create_job", {
        "p_organization_bk": organization_bk,
        "p_job_type": "discover_all_studies_v2",
        "p_payload": {"search_expression": search_expression}
    })

    if not job_id:
        return _err_result("job_creation_failed", "Could not create a discovery job in the database.")

    loop = asyncio.get_running_loop()
    loop.run_in_executor(
        _JOB_EXECUTOR,
        # [CORRECTION] Call the new V2 worker
        lambda: asyncio.run(_run_v2_discovery_job(job_id, search_expression))
    )

    return _ok_result({
        "message": f"Started V2 background discovery job for '{search_expression}'. Use get_job_status to check progress.",
        "job_id": job_id
    })

async def _run_v2_discovery_job(job_id: str, search_expression: str):
    """
    Background worker that uses the CT.gov V2 API directly to find all studies,
    handling modern token-based pagination.
    """
    logger.info(f"Starting V2 discovery job {job_id} for expression: '{search_expression}'")
    sp = get_supabase()
    await sp.call("ctgov_update_job_status", {"p_job_id": job_id, "p_status": "running"})

    all_nct_ids = []
    next_page_token = None
    base_url = "https://clinicaltrials.gov/api/v2/studies"
    
    try:
        async with httpx.AsyncClient(timeout=60.0) as client:
            while True:
                params = {
                    "query.spons": search_expression,
                    "fields": "NCTId",
                    "pageSize": 1000
                }
                if next_page_token:
                    params["pageToken"] = next_page_token

                logger.info(f"Job {job_id}: Fetching page with token {next_page_token}")
                response = await client.get(base_url, params=params)
                response.raise_for_status()
                data = response.json()

                studies = data.get('studies', [])
                for study in studies:
                    all_nct_ids.append(study['protocolSection']['identificationModule']['nctId'])

                next_page_token = data.get('nextPageToken')
                if not next_page_token:
                    break # Exit loop if there are no more pages

        result_payload = {
            "search_expression": search_expression,
            "total_studies_found": len(all_nct_ids),
            "nct_ids": all_nct_ids
        }
        await sp.call("ctgov_update_job_status", {
            "p_job_id": job_id, "p_status": "completed", "p_result": result_payload
        })
        logger.info(f"V2 Discovery job {job_id} completed. Found {len(all_nct_ids)} studies.")

    except Exception as e:
        import traceback
        logger.error(f"V2 Discovery job {job_id} failed: {e}\n{traceback.format_exc()}")
        await sp.call("ctgov_update_job_status", {
            "p_job_id": job_id, "p_status": "failed", "p_result": {"error": str(e)}
        })
'''


the mcp server system instructions as they stand now:

