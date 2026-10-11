# LinkedIn Auto Job Applier Next

An updated version of the original LinkedIn Auto Job Applier, modified to support LinkedIn's newer 2026 Jobs interface and improve compatibility with the current Easy Apply workflow.

This project is based on the original [Auto_job_applier_linkedIn](https://github.com/GodsScion/Auto_job_applier_linkedIn) project by Sai Vignesh Golla.

The main focus of this version is adapting the existing Selenium automation to LinkedIn's redesigned Jobs UI, which changed the DOM structure used for job cards, search filters, job details, and Easy Apply applications.

---

## My Changes

## 1. LinkedIn 2026 Jobs UI Compatibility

The original implementation relied heavily on LinkedIn's older Jobs DOM structure.

For example, job listings were previously detected using:

```python
//li[@data-occludable-job-id]
```

After LinkedIn's Jobs interface update, these elements were no longer present, causing the bot to wait for job listings until Selenium raised a `TimeoutException`.

The updated implementation supports the newer LinkedIn job card structure:

```html
<div
    role="button"
    componentkey="job-card-component-ref-4477265846">
```

Job cards can now be detected using:

```python
//*[@role="button" and starts-with(@componentkey, "job-card-component-ref-")]
```

This allows the bot to discover jobs in LinkedIn's current dynamically rendered search results.

---

## 2. New Job ID Extraction

The previous version obtained the LinkedIn Job ID from:

```python
job.get_dom_attribute("data-occludable-job-id")
```

The new LinkedIn interface no longer provides that attribute.

Job IDs are now embedded in the job card's `componentkey`.

For example:

```text
job-card-component-ref-4477265846
```

The updated code extracts:

```text
4477265846
```

using the new component key.

This Job ID continues to be used to construct canonical LinkedIn job URLs:

```text
https://www.linkedin.com/jobs/view/4477265846
```

---

## 3. Updated Job Card Detection

The job search loop was modified to support LinkedIn's new dynamically generated job cards.

Instead of querying:

```python
driver.find_elements(
    By.XPATH,
    "//li[@data-occludable-job-id]"
)
```

the updated version searches for the new job card components.

The job list is also re-queried during processing instead of keeping a single static list of Selenium elements.

This is important because LinkedIn virtualizes its search results and may replace DOM elements while the user or bot scrolls through the list.

Re-querying reduces problems with stale Selenium elements.

---

## 4. Updated Job Title Parsing

The previous implementation expected job titles to exist inside LinkedIn's older `artdeco-entity-lockup` structure.

The 2026 interface changed this structure.

The updated parser now supports the title elements used by the newer job cards.

For example, LinkedIn may render a card similar to:

```html
<span aria-hidden="true">
    Software Engineer
</span>
```

The parser now extracts the visible job title from the new card while retaining legacy selectors as fallbacks when possible.

---

## 5. Updated Company Parsing

One of the compatibility problems with the new interface was that the bot could detect a job title and Job ID but return:

```text
Software Engineer | Unknown
```

because the old company-name selector no longer existed.

The company parsing logic was updated to inspect the new job card structure instead of depending entirely on the old LinkedIn class names.

This allows job information to be represented correctly as:

```text
Software Engineer | Company Name
```

instead of:

```text
Software Engineer | Unknown
```

---

## 6. Updated Job Location Parsing

The job location parser was also adapted to LinkedIn's new job cards.

The updated implementation can retrieve location information such as:

```text
Phoenix, AZ (On-site)
Mesa, AZ
Santa Fe, NM (On-site)
```

from the new card layout.

Where possible, the parser also determines the work style from the location text:

```text
On-site
Remote
Hybrid
```

Legacy parsing remains available as a fallback.

---

## 7. Search Filter Compatibility Fix

LinkedIn changed the structure of its job-search filters.

The original bot expected elements such as:

```text
All filters
Past week
Entry level
Easy Apply
Show results
```

to be available through specific text and XPath selectors.

The newer LinkedIn interface caused errors such as:

```text
Click Failed! Didn't find 'Past week'
Click Failed! Didn't find 'Entry level'
Click Failed! Didn't find 'Easy Apply'
Setting the preferences failed!
```

To prevent the entire search process from failing when no filters are configured, I added logic that detects whether any search filters are actually enabled.

When all optional filters are disabled, the bot now skips the legacy:

```text
All filters
    ↓
Configure filters
    ↓
Show results
```

workflow completely.

Instead, it uses LinkedIn's already-loaded search results directly.

Example log:

```text
No search filters configured. Using LinkedIn search results directly.
```

This prevents outdated filter selectors from blocking the entire job-search process.

---

## 8. LinkedIn Search Location Compatibility

The previous search-location logic expected a location input with:

```python
aria-label="City, state, or zip code"
```

The newer LinkedIn interface no longer consistently exposes this element.

This previously resulted in:

```text
Setting search location as: "United States"
Search Location input was not given!
Setting the preferences failed!
```

The updated workflow allows the location field to remain empty so LinkedIn's existing/default location can be used without forcing the outdated location selector.

---

## 9. Updated Job Details Handling

LinkedIn's redesigned Jobs interface also changed the structure of the right-side job details panel.

The updated version adds support for newer component-based job details while preserving the original selectors as fallbacks.

This allows the bot to continue reading the selected job after clicking a new-style job card.

---

## 10. Updated Job Description Extraction

The job-description logic was updated for the newer LinkedIn Jobs interface.

The modified version can retrieve the current job description and continue using the original project's processing logic for:

- Required experience detection
- Blacklisted keywords
- Visa sponsorship phrases
- Master's degree requirements
- AI skill extraction
- Job relevance checks

This restored job-description processing after the old LinkedIn description selector stopped matching the new page.

---

## 11. Safer About Company Handling

The original blacklist workflow expected LinkedIn's old:

```python
jobs-company__box
```

element.

When that element could not be found in the new UI, jobs could incorrectly fail during the company blacklist check.

The updated implementation changes this behavior.

If the new LinkedIn UI does not expose the old About Company element, the bot no longer automatically treats that as a reason to skip the job.

The blacklist check is performed when company information is available, while missing legacy DOM elements are handled more gracefully.

This prevents valid jobs from being rejected simply because LinkedIn changed its HTML structure.

---

## 12. Updated Easy Apply Detection

LinkedIn's Easy Apply interface has also changed.

The updated version adds support for newer Easy Apply buttons, including selectors based on current accessibility attributes such as:

```text
aria-label="Easy Apply to this job"
```

The existing Easy Apply strategies are retained as fallbacks.

This creates multiple detection strategies instead of relying on a single LinkedIn DOM structure.

---

## 13. LinkedIn 2026 Dialog Support

A major change in the newer LinkedIn interface is the use of native-style dialog elements for application flows.

The current interface may create an application modal similar to:

```html
<dialog
    data-testid="dialog"
    open>
```

The original bot expected the older:

```text
jobs-easy-apply-modal
```

structure.

The updated implementation adds support for LinkedIn's newer dialog-based application UI.

This allows the bot to recognize that an Easy Apply application has actually opened.

---

## 14. Easy Apply Dialog Cleanup

During testing, a failed Easy Apply attempt could leave the LinkedIn application dialog open.

When the bot attempted to process the next job, Selenium would fail with:

```text
ElementClickInterceptedException
```

because the open dialog was covering the next job card.

The updated version adds cleanup behavior for application dialogs.

If an Easy Apply attempt fails or cannot continue, the bot attempts to close or dismiss the remaining dialog before moving to the next job.

This prevents one failed application from blocking the rest of the search run.

---

## 15. Improved Job Processing Stability

The job-processing workflow was modified to handle LinkedIn's dynamically rendered interface more safely.

The updated workflow is approximately:

```text
Search for jobs
      ↓
Detect new LinkedIn job cards
      ↓
Extract component key
      ↓
Extract Job ID
      ↓
Read title/company/location
      ↓
Open job
      ↓
Wait for job details
      ↓
Read description
      ↓
Check job requirements
      ↓
Detect Easy Apply
      ↓
Open application dialog
      ↓
Process application
      ↓
Clean up dialog
      ↓
Continue to next job
```

This replaces several assumptions that were specific to LinkedIn's previous Jobs UI.

---

## 16. Backward-Compatible Selectors

Where practical, I kept the original LinkedIn selectors as fallback strategies.

The goal was not to completely replace the original implementation with code that only works on the newest interface.

Instead, the updated logic generally follows:

```text
Try LinkedIn 2026 selector
        ↓
If unavailable
        ↓
Try legacy LinkedIn selector
```

This provides better compatibility across different LinkedIn layouts and accounts.

---

## 17. LinkedIn DOM Debugging Tools

To diagnose future LinkedIn UI changes, I added debugging support that can save the current page DOM when expected job elements cannot be found.

The bot can generate:

```text
linkedin_jobs_debug.html
linkedin_jobs_debug.png
```

The HTML file contains the actual LinkedIn DOM received by Selenium.

This makes it possible to inspect changes such as:

```text
Old:
data-occludable-job-id

New:
componentkey="job-card-component-ref-..."
```

instead of guessing which selector changed.

The screenshot provides additional visual context for the DOM capture.

---

## 18. Python 3.12 Compatibility

The project was configured and tested using:

```text
Python 3.12
```

A compatibility issue was discovered in `undetected_chromedriver`, which still imported:

```python
from distutils.version import LooseVersion
```

`distutils` is no longer available in Python 3.12 in the same way expected by that dependency.

The local compatibility fix replaces it with:

```python
from packaging.version import Version as LooseVersion
```

This allows `undetected_chromedriver` to import successfully in the Python 3.12 environment used for development.

---

## 19. ChromeDriver Compatibility

The project was tested with automatic ChromeDriver management enabled.

The startup workflow can detect and prepare a ChromeDriver compatible with the installed Chrome version.

The modified project was tested on:

```text
macOS
Python 3.12
Google Chrome
Selenium
undetected_chromedriver
```

---

## 20. Control Panel Testing

The updated bot was tested through the project's Flask control panel rather than only by executing the automation script directly.

The control panel can be started with:

```bash
python app.py
```

and accessed through the local address printed by Flask, for example:

```text
http://127.0.0.1:5000
```

This workflow was useful during development because configuration changes and automation logs could be tested repeatedly without modifying configuration files manually.

---

## 21. Privacy and Git Improvements

The repository configuration was updated to prevent local and personal files from being committed accidentally.

The `.gitignore` now excludes files such as:

```text
user_config.json
.env
.venv/
resumes/
logs/
*.pid
.bot_run*
linkedin_jobs_debug*.html
linkedin_jobs_debug*.png
*.backup
runAiBot_*_backup.py
.DS_Store
__pycache__/
*.pyc
```

This is particularly important because the local configuration may contain:

- LinkedIn login information
- API keys
- Personal contact information
- Resume files
- Application history
- Debugging data

These files should remain local and are not part of the public repository.

---

## 22. Tested Job Processing

During development, the updated implementation successfully progressed beyond the original job-card failures and processed LinkedIn's newer search results.

A test run was able to:

```text
Total runs:                   2
Jobs Easy Applied:           15
External job links collected: 0
Total applied or collected:  15
```

The run also processed skipped and failed jobs instead of terminating immediately when the new LinkedIn DOM was encountered.

This confirmed that the updated job-card detection, Job ID extraction, job-description processing, and Easy Apply workflow can operate against the newer LinkedIn interface.

---

## Main Technical Changes

| Component | Original Behavior | Updated Behavior |
|---|---|---|
| Job cards | `data-occludable-job-id` | `job-card-component-ref-*` support |
| Job ID | DOM attribute | Extracted from `componentkey` |
| Job title | Legacy LinkedIn classes | New card parsing + fallback |
| Company | Legacy lockup classes | New card parsing + fallback |
| Location | Legacy card structure | New card parsing + fallback |
| Search filters | Always use legacy filter flow | Skip when no filters are configured |
| Job details | Legacy class selectors | New component support + fallback |
| Description | Old Jobs DOM | New Jobs UI support |
| Company blacklist | Requires old About Company box | Graceful fallback when unavailable |
| Easy Apply | Legacy button/modal | New button + `<dialog>` support |
| Failed modal | Could block next job | Dialog cleanup added |
| Job list | Old `<li>` elements | New component-based cards |
| Debugging | Logs only | DOM + screenshot capture |
| Python | Older dependency assumptions | Python 3.12 compatibility work |
| Privacy | Basic ignore rules | Personal/debug/runtime files excluded |

---

## Current Limitations

LinkedIn frequently changes its frontend and does not provide a stable public DOM API for this type of browser automation.

Some areas may still require future compatibility updates, particularly:

- Pagination
- Infinite/virtualized scrolling
- Newly introduced Easy Apply question types
- Date-posted extraction
- Company information
- New application dialogs
- Search filter selectors

The implementation therefore keeps fallback selectors where possible and includes debugging tools to make future UI changes easier to diagnose.

---

## Original Project

This repository is based on:

[GodsScion/Auto_job_applier_linkedIn](https://github.com/GodsScion/Auto_job_applier_linkedIn)

Original author: **Sai Vignesh Golla**

My work in this repository focuses on compatibility updates, debugging, and adaptations for LinkedIn's newer Jobs and Easy Apply interfaces.

The original project and this modified version are distributed under the terms described in the repository's `LICENSE` and `NOTICE` files.
