# test

Here is the exact flow as drawn in v3. The numbers match the diagram labels, and the returns are the dashed arrows, which have no number.

**A. Request and opportunity check**
1. **Begin Order Creation (Opportunity ID):** User → Streamlit UI
2. **POST /log + POST /process:** Streamlit UI → Backend API
3. **Write session log (user, datetime, opp id):** Backend API → GCS. GCS returns "session logged".
4. **Check opportunity exists:** Backend API → GCS. GCS returns "exists / not found".

**B. `alt` block: opportunity found?**
- **Found:** continue to process (step 7).
- **Not found:**
  - Backend API returns "opportunity not found, offer Sync now" to the UI.
  - 5. **Sync now (user choice):** UI → Backend API
  - 6. **Sync opportunity data:** Backend API → GCS. GCS returns "sync result".
  - **`break` if sync failed:** Backend API returns "failure: opportunity ID doesn't exist" to the UI and the flow stops.
  - **If the sync succeeded:** continue to process (step 7).

**C. Process**
7. **Invoke workflow:** Backend API → Quote To Order Agent
8. **Read VBOP data + validation.json:** Q2O → GCS. GCS returns "VBOP data + validation rules".
9. **Validate VBOP vs validation.json:** Q2O, internal step
10. **Interpret documents (LLM):** Q2O → RAI
11. **pre_checks(text):** RAI, internal step
    - RAI forwards the prompt to GCP Gemini, and Gemini returns the model response.
12. **post_checks(response):** RAI, internal step
    - RAI returns the "interpreted result" to Q2O.

**D. `par` block: these two branches run at the same time**
- **Branch 1, BSP validation:**
  - 13. **Validate BSP checks (BigQuery tools):** Q2O → BigQuery. BigQuery returns the "BSP validation result".
- **Branch 2, product codes:**
  - 14. **Resolve product codes:** Q2O → Product Code Navigator
  - 15. **Deterministic lookup:** Product Code Navigator → BigQuery. BigQuery returns "matched codes", with no model call.
  - The Navigator returns "resolved codes" to Q2O.

**E. Order, save and output**
16. **Format order rows:** Q2O, internal step
17. **Draft email (LLM):** Q2O → RAI. RAI forwards the prompt to Gemini, Gemini returns the model response, and RAI returns "sales order + email draft" to Q2O.
    - Q2O returns "result (order, rows, email)" to Backend API.
18. **Save sales order + rows:** Backend API → BigQuery. BigQuery returns "saved".
    - Backend API sends the "final response" to the UI.
    - The UI shows the final output to the user.
