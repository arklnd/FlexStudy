# Git Activity Heatmap — All Modules

> **Period**: 2023-03-05 to 2026-03-05 (last 3 years)
>
> **Total commits analysed**: 401 (388 meaningful + 6 merge/squash)
>
> **Heat scale** (relative to highest-activity module):
> `HOT` = 75-100% | `WARM` = 50-74% | `MILD` = 25-49% | `COOL` = 1-24% | `NONE` = 0

---

## Table of Contents

1. [Module-Level Heatmap](#1-module-level-heatmap)
2. [Category Distribution](#2-category-distribution)
3. [Per-Module Detail and Top Files](#3-per-module-detail-and-top-files)
4. [Global Top-50 Hottest Files](#4-global-top-50-hottest-files)

---

## 1. Module-Level Heatmap

Sorted by total commit-touches. A *touch* = the module contained at least one changed file in that commit.

| Heat | Module | Total | Feature | Bugfix | Refactor | Chore | Test | Docs | Other |
|:----:|--------|------:|--------:|-------:|---------:|------:|-----:|-----:|------:|
| HOT  | **onbase-apps-host** | **156** | 85 | 41 | 0 | 0 | 1 | 0 | 29 |
| HOT  | **root/config** | **129** | 71 | 30 | 0 | 0 | 0 | 0 | 28 |
| WARM | **onbase-web-server** | **103** | 52 | 32 | 0 | 0 | 0 | 0 | 19 |
| MILD | **client-app** | **76** | 33 | 35 | 0 | 0 | 0 | 0 | 8 |
| MILD | **shared-components** | **62** | 34 | 20 | 0 | 0 | 0 | 0 | 8 |
| MILD | **wv-components** | **60** | 23 | 31 | 0 | 0 | 0 | 0 | 6 |
| MILD | **workflow-approval-mgmt** | **53** | 27 | 5 | 0 | 0 | 0 | 0 | 21 |
| MILD | **wf-approval-mgmt-lib** | **43** | 25 | 2 | 0 | 0 | 0 | 0 | 16 |
| MILD | **tools** | **40** | 21 | 10 | 0 | 0 | 0 | 0 | 9 |
| COOL | **dev-app** | **35** | 23 | 5 | 0 | 0 | 0 | 0 | 7 |
| COOL | **onbase-apps-shell** | **17** | 15 | 1 | 0 | 0 | 0 | 0 | 1 |
| COOL | **flex-router** | **10** | 8 | 1 | 0 | 0 | 0 | 0 | 1 |
| COOL | **onbase-web-server-core** | **10** | 8 | 0 | 0 | 0 | 0 | 0 | 2 |
| COOL | **flex-runtime** | **9** | 7 | 1 | 0 | 0 | 0 | 0 | 1 |
| COOL | **flex-shared** | **7** | 6 | 0 | 0 | 0 | 0 | 0 | 1 |
| COOL | **flex-forms** | **7** | 6 | 0 | 0 | 0 | 0 | 0 | 1 |
| COOL | **flex-operations** | **6** | 5 | 0 | 0 | 0 | 0 | 0 | 1 |
| COOL | **flex-types** | **6** | 5 | 0 | 0 | 0 | 0 | 0 | 1 |
| COOL | **flex-config** | **5** | 4 | 0 | 0 | 0 | 0 | 0 | 1 |
| COOL | **flex-host** | **5** | 4 | 0 | 0 | 0 | 0 | 0 | 1 |
| COOL | **e2e-shared** | **4** | 3 | 0 | 0 | 0 | 0 | 0 | 1 |

---

## 2. Category Distribution

Commit categories are derived from conventional commit prefixes (`feat:`, `fix:`, etc.) and
branch-name patterns embedded in PR merge messages (`feature/`, `bugfix/`, `hotfix/`).

| Category | Commits | % of non-merge |
|----------|--------:|---------------:|
| feature | 171 | 44.1% |
| bugfix | 123 | 31.7% |
| other | 93 | 24% |
| test | 1 | 0.3% |
| *merge/squash* | 6 | — |
| **TOTAL** | 401 | 100% |

### Note on 'other' category

Many commits use free-form subject lines (e.g., `version update in package.json`,
`Pull request #NNN: <branch-name>`) that do not match conventional commit prefixes.
These are bucketed as `other`. The intent can largely be inferred from the
JIRA ticket number in the branch name embedded in the merge message.

---

## 3. Per-Module Detail and Top Files

### [HOT ] onbase-apps-host

| Metric | Count |
|--------|------:|
| Total commit touches | **156** |
| Feature | 85 |
| Bugfix | 41 |
| Refactor | 0 |
| Chore | 0 |
| Test | 1 |
| Docs | 0 |
| Other / version bumps | 29 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 77 | 35 | 22 | 20 | `package.json` |
| 2 | 32 | 22 | 5 | 5 | `src/app/features/sources/core/services/source-view-registry.service.ts` |
| 3 | 20 | 15 | 3 | 2 | `string-constants.json` |
| 4 | 16 | 12 | 3 | 1 | `src/app/features/source-view-presenter/components/hy-ng-source-view-presenter.component.ts` |
| 5 | 13 | 11 | 1 | 1 | `src/app/features/master-detail-w-sources/components/hy-ng-master-detail-w-sources/hy-ng-master-detail-w-sources.component.html` |
| 6 | 13 | 8 | 4 | 1 | `src/app/features/master-detail-w-sources/components/hy-ng-master-detail-w-sources/hy-ng-master-detail-w-sources.component.ts` |
| 7 | 13 | 8 | 4 | 1 | `src/app/style-components/theme/app.theme.scss` |
| 8 | 12 | 11 | 1 | 0 | `src/app/features/source-view-presenter/components/hy-ng-source-view-presenter.component.spec.ts` |
| 9 | 12 | 10 | 2 | 0 | `src/app/features/master-detail-w-sources/components/hy-ng-master-detail-w-sources/hy-ng-master-detail-w-sources.component.spec.ts` |
| 10 | 10 | 9 | 0 | 1 | `src/app/features/quick-access/components/hy-ng-quick-access-component/hy-ng-quick-access.component.ts` |
| 11 | 9 | 8 | 0 | 1 | `src/app/views/settings/settings.view.ts` |
| 12 | 9 | 8 | 0 | 1 | `src/app/features/quick-access/components/hy-ng-quick-access-component/hy-ng-quick-access.component.html` |
| 13 | 9 | 6 | 2 | 1 | `project.json` |
| 14 | 9 | 8 | 0 | 1 | `src/app/views/settings/settings.view.spec.ts` |
| 15 | 9 | 6 | 1 | 2 | `src/app/features/quick-access/components/hy-ng-quick-access-component/hy-ng-quick-access.component.scss` |

### [WARM] onbase-web-server

| Metric | Count |
|--------|------:|
| Total commit touches | **103** |
| Feature | 52 |
| Bugfix | 32 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 19 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 26 | 18 | 3 | 5 | `src/index.ts` |
| 2 | 23 | 15 | 5 | 3 | `src/lib/source-views/workflow/components/workflow-queue.sv.component.ts` |
| 3 | 19 | 11 | 7 | 1 | `src/lib/source-views/workview/components/workview-filter.sv.component.ts` |
| 4 | 18 | 10 | 7 | 1 | `src/lib/modules/content/common/work-item-in-queue-tasks-menu/work-item-in-queue-tasks-menu.component.ts` |
| 5 | 18 | 12 | 5 | 1 | `src/lib/source-views/doc-query/components/document-content-grid/document-content-grid.sv.component.ts` |
| 6 | 17 | 10 | 3 | 4 | `src/lib/source-views/workflow/components/workflow-queue.sv.component.html` |
| 7 | 17 | 9 | 6 | 2 | `src/lib/modules/content/searching/search-panel/search-panel.component.ts` |
| 8 | 17 | 10 | 6 | 1 | `src/lib/modules/content/common/work-item-tasks-menu/work-item-tasks-menu.component.ts` |
| 9 | 15 | 8 | 5 | 2 | `src/lib/modules/content/viewer/viewer.component.ts` |
| 10 | 15 | 9 | 3 | 3 | `src/lib/source-views/doc-query/components/document-content-grid/document-content-grid.sv.component.html` |
| 11 | 15 | 11 | 2 | 2 | `src/lib/modules/content/data-grid/components/content-data-grid.component.ts` |
| 12 | 14 | 9 | 2 | 3 | `src/lib/modules/content/searching/search-panel/search-panel.component.html` |
| 13 | 13 | 7 | 3 | 3 | `src/lib/source-views/workview/components/workview-filter.sv.component.html` |
| 14 | 13 | 6 | 4 | 3 | `src/lib/modules/content/related-items/source-view-related-items.component.ts` |
| 15 | 12 | 11 | 0 | 1 | `src/lib/shared/dialog-service/modal-dialog-container/modal-dialog-container.component.ts` |

### [MILD] wv-components

| Metric | Count |
|--------|------:|
| Total commit touches | **60** |
| Feature | 23 |
| Bugfix | 31 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 6 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 22 | 14 | 7 | 1 | `src/lib/hy-ng-workview-object-viewer/core/services/object-service-bridge/object-service-bridge.service.ts` |
| 2 | 15 | 9 | 5 | 1 | `src/lib/hy-ng-workview-object-viewer/core/services/workview-controller-object/workview-controller-object.service.ts` |
| 3 | 10 | 8 | 2 | 0 | `src/lib/hy-ng-workview-object-viewer/views/viewer-shell/viewer-shell.view.ts` |
| 4 | 9 | 7 | 1 | 1 | `src/lib/hy-ng-workview-object-viewer/views/viewer-shell/components/object-history-dialog/object-history.dialog.ts` |
| 5 | 8 | 6 | 1 | 1 | `src/lib/constraints/hy-ng-entry-constraint/hy-ng-entry-constraint.component.ts` |
| 6 | 8 | 6 | 2 | 0 | `src/lib/hy-ng-workview-object-viewer/core/api-services/view-writer-api/view-writer-api.service.ts` |
| 7 | 8 | 6 | 2 | 0 | `src/lib/hy-ng-workview-object-viewer/core/services/view-writer/view-writer.service.ts` |
| 8 | 8 | 7 | 1 | 0 | `src/lib/hy-ng-workview-object-viewer/core/services/viewer-shell/viewer-shell.service.ts` |
| 9 | 8 | 7 | 0 | 1 | `src/lib/hy-ng-workview-object-viewer/hy-ng-workview-object-viewer.component.ts` |
| 10 | 7 | 7 | 0 | 0 | `src/index.ts` |
| 11 | 7 | 5 | 1 | 1 | `src/lib/hy-ng-workview-object-viewer/views/viewer-shell/components/object-history-dialog/object-history.dialog.spec.ts` |
| 12 | 6 | 5 | 0 | 1 | `src/lib/constraints/hy-ng-entry-constraint/hy-ng-entry-constraint.component.spec.ts` |
| 13 | 6 | 5 | 1 | 0 | `src/lib/hy-ng-workview-object-viewer/core/services/view-writer/mock-view-writer.service.ts` |
| 14 | 6 | 5 | 0 | 1 | `src/lib/hy-ng-workview-object-viewer/views/viewer-shell/components/template-dialog/template.dialog.spec.ts` |
| 15 | 6 | 6 | 0 | 0 | `src/lib/hy-ng-workview-object-viewer/test-utils/mock-services.ts` |

### [MILD] wf-approval-mgmt-lib

| Metric | Count |
|--------|------:|
| Total commit touches | **43** |
| Feature | 25 |
| Bugfix | 2 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 16 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 16 | 11 | 0 | 5 | `src/lib/shared/data-access/services/approval-processes.service.ts` |
| 2 | 15 | 10 | 0 | 5 | `src/lib/shared/data-access/services/approval-processes.service.spec.ts` |
| 3 | 14 | 10 | 0 | 4 | `src/lib/shared/data-access/facade/approval-process-details-facade.service.ts` |
| 4 | 12 | 7 | 0 | 5 | `src/lib/shared/components/hy-ng-conditions/hy-ng-conditions.component.html` |
| 5 | 10 | 6 | 0 | 4 | `src/lib/shared/data-access/store/effects/approval-process-details.effect.ts` |
| 6 | 9 | 5 | 0 | 4 | `src/lib/shared/data-access/store/effects/approval-process-details.effect.spec.ts` |
| 7 | 9 | 6 | 0 | 3 | `src/lib/shared/components/hy-ng-conditions/hy-ng-conditions.component.spec.ts` |
| 8 | 9 | 6 | 0 | 3 | `src/lib/shared/data-access/store/actions/approval-process-details.action.ts` |
| 9 | 9 | 6 | 0 | 3 | `src/lib/shared/data-access/store/reducers/approval-process-details.reducer.ts` |
| 10 | 8 | 5 | 0 | 3 | `src/lib/shared/data-access/store/reducers/approval-process-details.reducer.spec.ts` |
| 11 | 8 | 6 | 0 | 2 | `src/lib/shared/data-access/facade/approval-process-details-facade.service.spec.ts` |
| 12 | 7 | 4 | 0 | 3 | `src/lib/shared/components/hy-ng-conditions/hy-ng-conditions.component.ts` |
| 13 | 7 | 4 | 0 | 3 | `src/lib/shared/helpers/mockJsonHelper.ts` |
| 14 | 6 | 3 | 0 | 3 | `src/lib/alerts/hy-ng-modal/hy-ng-modal.component.html` |
| 15 | 5 | 3 | 0 | 2 | `src/lib/alerts/hy-ng-modal/_hy-ng-modal.theme.scss` |

### [MILD] workflow-approval-mgmt

| Metric | Count |
|--------|------:|
| Total commit touches | **53** |
| Feature | 27 |
| Bugfix | 5 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 21 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 21 | 13 | 2 | 6 | `package.json` |
| 2 | 11 | 7 | 0 | 4 | `src/app/components/hy-ng-approval-process/hy-ng-rules/hy-ng-rule-details/hy-ng-rules-detail.component.html` |
| 3 | 11 | 4 | 2 | 5 | `src/app/components/hy-ng-approval-process/hy-ng-process/hy-ng-process.component.ts` |
| 4 | 10 | 2 | 0 | 8 | `src/app/components/hy-ng-approval-process/hy-ng-approval-process.component.html` |
| 5 | 10 | 6 | 0 | 4 | `src/app/components/hy-ng-approval-process/hy-ng-rules/hy-ng-rule-details/hy-ng-rules-detail.component.scss` |
| 6 | 10 | 6 | 0 | 4 | `src/app/components/hy-ng-approval-process/hy-ng-process/hy-ng-modify-pane/hy-ng-modify-pane.component.spec.ts` |
| 7 | 8 | 3 | 0 | 5 | `src/app/components/hy-ng-approval-process/hy-ng-rules/hy-ng-rule-details/hy-ng-rules-detail.component.ts` |
| 8 | 8 | 4 | 0 | 4 | `src/app/components/hy-ng-approval-process/hy-ng-rules/hy-ng-rule-details/hy-ng-rules-detail.component.spec.ts` |
| 9 | 8 | 4 | 0 | 4 | `src/app/components/hy-ng-approval-process/hy-ng-process/hy-ng-modify-pane/hy-ng-modify-pane.component.ts` |
| 10 | 7 | 3 | 0 | 4 | `src/app/components/hy-ng-dashboard/hy-ng-dashboard.component.html` |
| 11 | 7 | 2 | 0 | 5 | `src/app/components/hy-ng-approval-process/hy-ng-rules/hy-ng-rule-details/hy-ng-modify-rule/hy-ng-modify-rule.component.scss` |
| 12 | 7 | 4 | 0 | 3 | `src/app/components/hy-ng-approval-process/hy-ng-rules/hy-ng-rule-details/hy-ng-modify-rule/hy-ng-modify-rule.component.spec.ts` |
| 13 | 7 | 4 | 0 | 3 | `src/app/components/hy-ng-approval-process/hy-ng-process/hy-ng-process.component.spec.ts` |
| 14 | 7 | 2 | 0 | 5 | `src/app/components/hy-ng-approval-process/hy-ng-approval-process.component.scss` |
| 15 | 6 | 3 | 0 | 3 | `src/app/components/hy-ng-approval-process/hy-ng-rules/hy-ng-rules.component.scss` |

### [MILD] shared-components

| Metric | Count |
|--------|------:|
| Total commit touches | **62** |
| Feature | 34 |
| Bugfix | 20 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 8 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 21 | 12 | 4 | 5 | `src/lib/hy-ng-data-grid/hy-ng-data-grid.component.ts` |
| 2 | 16 | 7 | 5 | 4 | `src/lib/hy-ng-data-grid/hy-ng-data-grid.component.html` |
| 3 | 14 | 11 | 2 | 1 | `src/lib/hy-ng-data-grid/hy-ng-data-grid.component.spec.ts` |
| 4 | 11 | 9 | 0 | 2 | `src/lib/menu-with-buttons/menu-with-buttons.component.html` |
| 5 | 11 | 9 | 0 | 2 | `src/lib/shared-components.module.ts` |
| 6 | 9 | 7 | 1 | 1 | `src/lib/hy-ng-table-group-menu/hy-ng-table-group-menu.component.html` |
| 7 | 9 | 5 | 3 | 1 | `src/lib/hy-ng-table-group-menu/hy-ng-table-group-menu.component.ts` |
| 8 | 8 | 6 | 2 | 0 | `src/index.ts` |
| 9 | 8 | 6 | 1 | 1 | `src/lib/hy-ng-table-group-menu/hy-ng-table-group-menu.component.spec.ts` |
| 10 | 8 | 7 | 0 | 1 | `src/lib/menu-with-buttons/menu-with-buttons.component.ts` |
| 11 | 6 | 5 | 0 | 1 | `src/lib/shared-components.module.spec.ts` |
| 12 | 6 | 4 | 1 | 1 | `src/lib/hy-ng-splitter/hy-ng-splitter.component.ts` |
| 13 | 6 | 4 | 1 | 1 | `src/lib/hy-ng-search-bar/hy-ng-search-bar.component.ts` |
| 14 | 5 | 4 | 0 | 1 | `src/lib/hy-ng-search-bar/hy-ng-search-bar.component.html` |
| 15 | 5 | 3 | 2 | 0 | `src/lib/hy-ng-data-grid/hy-ng-data-grid.component.scss` |

### [MILD] client-app

| Metric | Count |
|--------|------:|
| Total commit touches | **76** |
| Feature | 33 |
| Bugfix | 35 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 8 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 52 | 16 | 29 | 7 | `package.json` |
| 2 | 15 | 10 | 3 | 2 | `string-constants.json` |
| 3 | 7 | 1 | 5 | 1 | `src/environments/config/screens/data-grid-screen.ts` |
| 4 | 7 | 4 | 2 | 1 | `project.json` |
| 5 | 6 | 5 | 1 | 0 | `src/app/views/viewer/viewer.view.spec.ts` |
| 6 | 6 | 5 | 0 | 1 | `src/app/style-components/theme/app.theme.scss` |
| 7 | 6 | 4 | 1 | 1 | `src/app/components/user-settings/user-settings.component.ts` |
| 8 | 6 | 5 | 1 | 0 | `src/app/views/viewer/viewer.view.ts` |
| 9 | 6 | 3 | 2 | 1 | `src/app/views/client/client.view.ts` |
| 10 | 5 | 4 | 0 | 1 | `src/app/components/create-object-button/create-object-button.component.spec.ts` |
| 11 | 5 | 3 | 1 | 1 | `src/app/components/user-settings/user-settings.component.spec.ts` |
| 12 | 5 | 1 | 4 | 0 | `src/app/core/services/user-settings/user-settings.service.ts` |
| 13 | 4 | 0 | 4 | 0 | `src/environments/environment.ts` |
| 14 | 4 | 0 | 4 | 0 | `src/environments/environment.test.ts` |
| 15 | 4 | 0 | 4 | 0 | `src/app/core/services/application/application.service.ts` |

### [MILD] tools

| Metric | Count |
|--------|------:|
| Total commit touches | **40** |
| Feature | 21 |
| Bugfix | 10 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 9 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 11 | 3 | 6 | 2 | `tools/vagrant/provisioners/configure-obapps-tests.rb` |
| 2 | 7 | 6 | 1 | 0 | `tools/styling/igx-variable-map.scss` |
| 3 | 6 | 3 | 1 | 2 | `tools/vagrant/obapps-app-vm.rb` |
| 4 | 4 | 2 | 1 | 1 | `tools/ban-fit-and-fdescribe.eslintrc` |
| 5 | 4 | 3 | 1 | 0 | `tools/vagrant/common/initialize.rb` |
| 6 | 4 | 2 | 1 | 1 | `tools/styling/_apply-default-themes.scss` |
| 7 | 4 | 1 | 1 | 2 | `tools/vagrant/provisioners/configure-workflow-approval-mgmt-tests.rb` |
| 8 | 3 | 1 | 0 | 2 | `tools/build/build-spec.yml` |
| 9 | 3 | 1 | 0 | 2 | `tools/vagrant/workflow-approval-mgmt-app-vm.rb` |
| 10 | 3 | 2 | 0 | 1 | `tools/styling/_hy-ui-var-safe.scss` |
| 11 | 3 | 1 | 0 | 2 | `tools/vagrant/wvclient-app-vm.rb` |
| 12 | 2 | 1 | 0 | 1 | `tools/vagrant/provisioners/configure-wvclient-tests.rb` |
| 13 | 2 | 2 | 0 | 0 | `tools/hooks/prepush.sh` |
| 14 | 2 | 2 | 0 | 0 | `tools/build/extract-strings-constants.js` |
| 15 | 2 | 2 | 0 | 0 | `tools/test-patch.ts` |

### [HOT ] root/config

| Metric | Count |
|--------|------:|
| Total commit touches | **129** |
| Feature | 71 |
| Bugfix | 30 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 28 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 33 | 22 | 6 | 5 | `package.json` |
| 2 | 33 | 20 | 5 | 8 | `Jenkinsfile` |
| 3 | 27 | 16 | 5 | 6 | `package-lock.json` |
| 4 | 17 | 15 | 1 | 1 | `src/e2e/index.app.ts` |
| 5 | 13 | 6 | 4 | 3 | `src/e2e/general-runtime/workflow/workflow-sources.cy.ts` |
| 6 | 12 | 5 | 0 | 7 | `src/page_objects/approvalProcessPage.ts` |
| 7 | 12 | 7 | 1 | 4 | `src/e2e/general-runtime/master-detail.cy.ts` |
| 8 | 11 | 8 | 2 | 1 | `.bitbucket/CODEOWNERS` |
| 9 | 11 | 6 | 2 | 3 | `src/e2e/general-runtime/workflow/common-functions.app.ts` |
| 10 | 11 | 6 | 2 | 3 | `src/e2e/general-runtime/quick-access-action.cy.ts` |
| 11 | 10 | 5 | 2 | 3 | `src/e2e/general-runtime/workflow/adhoc-task.cy.ts` |
| 12 | 9 | 3 | 0 | 6 | `src/e2e/approvalProcess/autoApprovalPaths.cy.ts` |
| 13 | 9 | 4 | 0 | 5 | `src/e2e/approvalProcess/approvalLevels.cy.ts` |
| 14 | 9 | 6 | 1 | 2 | `src/e2e/general-runtime/workflow/wv-objects-lifecycle.cy.ts` |
| 15 | 9 | 3 | 0 | 6 | `src/e2e/approvalProcess/approvalPaths.cy.ts` |

### [COOL] onbase-apps-shell

| Metric | Count |
|--------|------:|
| Total commit touches | **17** |
| Feature | 15 |
| Bugfix | 1 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 10 | 9 | 0 | 1 | `src/lib/components/shell-header/shell-header.component.html` |
| 2 | 9 | 7 | 1 | 1 | `src/lib/components/shell-header/shell-header.component.ts` |
| 3 | 6 | 5 | 0 | 1 | `src/lib/components/shell-header/shell-header.component.spec.ts` |
| 4 | 4 | 3 | 0 | 1 | `src/lib/components/shell-header/shell-header.component.scss` |
| 5 | 4 | 3 | 0 | 1 | `project.json` |
| 6 | 4 | 4 | 0 | 0 | `src/lib/components/shell/shell.component.html` |
| 7 | 4 | 4 | 0 | 0 | `src/lib/components/shell/shell.component.ts` |
| 8 | 3 | 2 | 0 | 1 | `src/lib/components/_onbase-apps-shell-theme.scss` |
| 9 | 3 | 3 | 0 | 0 | `src/lib/components/shell-view/shell-view.component.spec.ts` |
| 10 | 3 | 3 | 0 | 0 | `src/lib/onbase-apps-shell.module.ts` |
| 11 | 2 | 2 | 0 | 0 | `src/index.ts` |
| 12 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 13 | 2 | 2 | 0 | 0 | `src/lib/components/shell-view/shell-view.component.ts` |
| 14 | 2 | 2 | 0 | 0 | `src/lib/material.module.ts` |
| 15 | 1 | 1 | 0 | 0 | `eslint.config.mjs` |

### [COOL] dev-app

| Metric | Count |
|--------|------:|
| Total commit touches | **35** |
| Feature | 23 |
| Bugfix | 5 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 7 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 17 | 12 | 3 | 2 | `string-constants.json` |
| 2 | 11 | 5 | 1 | 5 | `package.json` |
| 3 | 7 | 4 | 0 | 3 | `src/app/views/file-upload-dialog-wrapper/file-upload-dialog-wrapper.component.spec.ts` |
| 4 | 6 | 4 | 0 | 2 | `src/app/views/file-upload/file-upload.component.ts` |
| 5 | 6 | 3 | 0 | 3 | `src/app/views/file-upload-dialog-wrapper/file-upload-dialog-wrapper.component.ts` |
| 6 | 5 | 3 | 1 | 1 | `project.json` |
| 7 | 5 | 4 | 0 | 1 | `src/app/views/doc-search/doc-search.component.ts` |
| 8 | 5 | 4 | 0 | 1 | `src/app/views/doc-query-results/doc-query-results.component.ts` |
| 9 | 5 | 3 | 0 | 2 | `src/app/views/folders/folders.component.ts` |
| 10 | 3 | 2 | 0 | 1 | `src/app/modules/material.module.ts` |
| 11 | 3 | 2 | 0 | 1 | `src/app/views/file-upload/file-upload.component.spec.ts` |
| 12 | 3 | 3 | 0 | 0 | `src/app/app.component.spec.ts` |
| 13 | 3 | 2 | 0 | 1 | `src/theme.scss` |
| 14 | 3 | 1 | 0 | 2 | `src/app/views/file-upload/file-upload.component.html` |
| 15 | 2 | 2 | 0 | 0 | `src/app/views/home/home.component.ts` |

### [COOL] flex-router

| Metric | Count |
|--------|------:|
| Total commit touches | **10** |
| Feature | 8 |
| Bugfix | 1 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 4 | 3 | 0 | 1 | `project.json` |
| 2 | 4 | 3 | 1 | 0 | `src/lib/components/flex-screen/flex-screen.component.ts` |
| 3 | 4 | 4 | 0 | 0 | `src/index.ts` |
| 4 | 3 | 3 | 0 | 0 | `src/lib/components/flex-router-outlet/flex-router-outlet.component.spec.ts` |
| 5 | 3 | 3 | 0 | 0 | `src/lib/directives/flex-router-link.directive.ts` |
| 6 | 3 | 3 | 0 | 0 | `src/lib/components/flex-router-outlet/flex-router-outlet.component.ts` |
| 7 | 3 | 3 | 0 | 0 | `src/lib/services/router/flex-router.service.spec.ts` |
| 8 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 9 | 2 | 2 | 0 | 0 | `src/lib/services/router/router-navigation-facilitator/flex-router-navigation-facilitator.service.spec.ts` |
| 10 | 2 | 2 | 0 | 0 | `src/lib/directives/flex-router-link.module.ts` |
| 11 | 2 | 2 | 0 | 0 | `src/lib/services/router/router-tree/flex-router-tree.service.spec.ts` |
| 12 | 2 | 2 | 0 | 0 | `src/lib/components/flex-screen/flex-screen.component.spec.ts` |
| 13 | 2 | 2 | 0 | 0 | `src/lib/services/router/router-tree/tree-node/screen-node.ts` |
| 14 | 2 | 1 | 1 | 0 | `src/lib/services/focus.service.spec.ts` |
| 15 | 2 | 2 | 0 | 0 | `src/lib/services/router/flex-router.service.ts` |

### [COOL] flex-runtime

| Metric | Count |
|--------|------:|
| Total commit touches | **9** |
| Feature | 7 |
| Bugfix | 1 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 6 | 5 | 0 | 1 | `src/lib/components/dialogs/alert-dialog/flex-alert.dialog.spec.ts` |
| 2 | 6 | 5 | 0 | 1 | `src/lib/components/dialogs/confirm-dialog/flex-confirm.dialog.spec.ts` |
| 3 | 6 | 4 | 1 | 1 | `src/lib/components/flex-app-viewer/flex-app-viewer.component.spec.ts` |
| 4 | 4 | 3 | 1 | 0 | `src/lib/flex.module.ts` |
| 5 | 4 | 4 | 0 | 0 | `src/lib/components/dialogs/confirm-dialog/flex-confirm.dialog.ts` |
| 6 | 4 | 3 | 0 | 1 | `src/lib/services/flex-dialog.service.spec.ts` |
| 7 | 4 | 3 | 0 | 1 | `project.json` |
| 8 | 4 | 4 | 0 | 0 | `src/lib/components/dialogs/alert-dialog/flex-alert.dialog.ts` |
| 9 | 2 | 1 | 1 | 0 | `src/lib/components/flex-app-viewer/flex-app-viewer.component.ts` |
| 10 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 11 | 2 | 2 | 0 | 0 | `src/lib/services/property-binding/flex-property-binding.service.spec.ts` |
| 12 | 1 | 1 | 0 | 0 | `tsconfig.spec.json` |
| 13 | 1 | 1 | 0 | 0 | `src/lib/services/runtime-config/flex-runtime-config.service.spec.ts` |
| 14 | 1 | 1 | 0 | 0 | `src/lib/guards/can-screen-deactivate.guard.spec.ts` |
| 15 | 1 | 1 | 0 | 0 | `src/lib/services/forms/form-to-screen-registry.service.spec.ts` |

### [COOL] flex-shared

| Metric | Count |
|--------|------:|
| Total commit touches | **7** |
| Feature | 6 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 4 | 3 | 0 | 1 | `project.json` |
| 2 | 2 | 1 | 0 | 1 | `src/lib/utils/task-queue.spec.ts` |
| 3 | 2 | 2 | 0 | 0 | `src/lib/test-utils/flex-shared-testing.module.ts` |
| 4 | 2 | 2 | 0 | 0 | `src/index.ts` |
| 5 | 2 | 2 | 0 | 0 | `src/lib/components/style-helper/style-helper.component.ts` |
| 6 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 7 | 1 | 1 | 0 | 0 | `eslint.config.mjs` |
| 8 | 1 | 1 | 0 | 0 | `src/lib/test-utils/transloco-testing.module.ts` |
| 9 | 1 | 1 | 0 | 0 | `src/lib/interfaces/operation-execution-args.ts` |
| 10 | 1 | 1 | 0 | 0 | `src/lib/utils/format-string.spec.ts` |
| 11 | 1 | 1 | 0 | 0 | `src/lib/services/http-service/flex-http.service.spec.ts` |
| 12 | 1 | 1 | 0 | 0 | `src/lib/flex-shared.module.ts` |
| 13 | 1 | 1 | 0 | 0 | `src/lib/interfaces/services/router-param-service/router-parameter-service.ts` |
| 14 | 1 | 1 | 0 | 0 | `src/lib/services/log-service/flex-log.service.ts` |
| 15 | 1 | 1 | 0 | 0 | `src/lib/components/style-helper.module.ts` |

### [COOL] onbase-web-server-core

| Metric | Count |
|--------|------:|
| Total commit touches | **10** |
| Feature | 8 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 2 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 5 | 4 | 0 | 1 | `src/lib/pipes/translate.pipe.ts` |
| 2 | 4 | 3 | 0 | 1 | `project.json` |
| 3 | 3 | 3 | 0 | 0 | `src/index.ts` |
| 4 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 5 | 2 | 2 | 0 | 0 | `src/lib/shared/models/web-client.window.ts` |
| 6 | 2 | 2 | 0 | 0 | `src/lib/shared/services/windows-manager/windows-manager.service.interface.ts` |
| 7 | 2 | 2 | 0 | 0 | `src/lib/shared/services/windows-manager/mock-windows-manager.service.ts` |
| 8 | 2 | 2 | 0 | 0 | `src/lib/shared/services/data-validation/data-validation.interface.ts` |
| 9 | 2 | 2 | 0 | 0 | `src/lib/shared/services/unload-manager/unload-manager.service.interface.ts` |
| 10 | 2 | 2 | 0 | 0 | `.eslintrc.json` |
| 11 | 2 | 2 | 0 | 0 | `src/lib/shared/services/page/page.service.interface.ts` |
| 12 | 1 | 1 | 0 | 0 | `karma.conf.js` |
| 13 | 1 | 1 | 0 | 0 | `src/lib/shared/interfaces/onbase-web-server-unloadable-component.interface.ts` |
| 14 | 1 | 1 | 0 | 0 | `src/lib/shared/services/session-checker/mock-session-checker.service.ts` |
| 15 | 1 | 1 | 0 | 0 | `src/lib/shared/services/unload-manager/mock-unload-manager.service.ts` |

### [COOL] flex-forms

| Metric | Count |
|--------|------:|
| Total commit touches | **7** |
| Feature | 6 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 4 | 3 | 0 | 1 | `src/lib/services/ng-forms.service.spec.ts` |
| 2 | 4 | 3 | 0 | 1 | `project.json` |
| 3 | 2 | 2 | 0 | 0 | `src/lib/services/validation-types/max.validation.ts` |
| 4 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 5 | 2 | 2 | 0 | 0 | `src/lib/services/validation-types/min-length.validation.ts` |
| 6 | 2 | 2 | 0 | 0 | `src/lib/services/validation-types/min.validation.ts` |
| 7 | 2 | 2 | 0 | 0 | `src/lib/services/test-data/mock-test-component.component.ts` |
| 8 | 2 | 2 | 0 | 0 | `src/lib/services/test-data/mock-flex-form.component.ts` |
| 9 | 2 | 2 | 0 | 0 | `src/lib/services/validation-types/max-length.validation.ts` |
| 10 | 2 | 2 | 0 | 0 | `src/lib/services/validation-types/occurs-before.validation.ts` |
| 11 | 2 | 2 | 0 | 0 | `src/lib/services/validation-types/occurs-after.validation.ts` |
| 12 | 1 | 1 | 0 | 0 | `src/lib/services/validation-types/required-no-whitespace.validation.ts` |
| 13 | 1 | 1 | 0 | 0 | `src/lib/validators/numeric.validator.ts` |
| 14 | 1 | 1 | 0 | 0 | `src/lib/services/test-data/wrapper-component.component.ts` |
| 15 | 1 | 1 | 0 | 0 | `src/lib/models/custom-expressions/format-string.expression.ts` |

### [COOL] flex-operations

| Metric | Count |
|--------|------:|
| Total commit touches | **6** |
| Feature | 5 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 5 | 4 | 0 | 1 | `src/lib/operations/display-component-dialog/flex-component-dialog/flex-component.dialog.spec.ts` |
| 2 | 5 | 4 | 0 | 1 | `src/lib/operations/display-component-dialog/flex-component-dialog/flex-component.dialog.ts` |
| 3 | 4 | 3 | 0 | 1 | `project.json` |
| 4 | 4 | 3 | 0 | 1 | `src/lib/operations/display-component-dialog/display-component-dialog.operation.spec.ts` |
| 5 | 4 | 3 | 0 | 1 | `src/lib/flex-operations.module.ts` |
| 6 | 2 | 2 | 0 | 0 | `src/lib/operations/compare/compare.operation.ts` |
| 7 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 8 | 2 | 1 | 0 | 1 | `src/lib/operations/display-component-dialog/flex-component-dialog/flex-component.dialog.scss` |
| 9 | 1 | 1 | 0 | 0 | `src/lib/state-store/operation-state-store.proxy.spec.ts` |
| 10 | 1 | 1 | 0 | 0 | `src/lib/operations/set-property-from-route-data-param/set-property-from-route-data-param.operation.spec.ts` |
| 11 | 1 | 1 | 0 | 0 | `src/lib/operations/bind-form-data/bind-form-data.operation.spec.ts` |
| 12 | 1 | 1 | 0 | 0 | `src/lib/operations/set-property/set-property.operation.ts` |
| 13 | 1 | 1 | 0 | 0 | `src/lib/operations/invoke-method/invoke-method.operation.ts` |
| 14 | 1 | 1 | 0 | 0 | `src/lib/services/operation-type-registry.service/operation-type-registry.service.ts` |
| 15 | 1 | 1 | 0 | 0 | `src/lib/operations/bind-form-data/bind-form-data.operation.ts` |

### [COOL] flex-config

| Metric | Count |
|--------|------:|
| Total commit touches | **5** |
| Feature | 4 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 4 | 3 | 0 | 1 | `project.json` |
| 2 | 2 | 2 | 0 | 0 | `src/lib/application-configuration.creator/flex-application.config-creator.ts` |
| 3 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 4 | 1 | 1 | 0 | 0 | `.eslintrc.json` |
| 5 | 1 | 1 | 0 | 0 | `src/index.ts` |
| 6 | 1 | 1 | 0 | 0 | `src/lib/application-configuration.creator/operation-list.config-creator.ts` |
| 7 | 1 | 1 | 0 | 0 | `eslint.config.mjs` |
| 8 | 1 | 1 | 0 | 0 | `src/lib/application-configuration/onbase-settings.ts` |
| 9 | 1 | 1 | 0 | 0 | `src/lib/application-configuration/operations/operation-config-data.ts` |
| 10 | 1 | 1 | 0 | 0 | `src/lib/application-configuration.creator/flex-screen.config-creator.ts` |
| 11 | 1 | 1 | 0 | 0 | `src/lib/application-configuration.creator/event-mapping.config-creator.ts` |
| 12 | 1 | 1 | 0 | 0 | `src/lib/application-configuration.creator/control.config-creator.ts` |
| 13 | 1 | 1 | 0 | 0 | `tsconfig.spec.json` |
| 14 | 1 | 1 | 0 | 0 | `src/lib/application-configuration/forms/form-field-validation-data.ts` |
| 15 | 1 | 1 | 0 | 0 | `src/lib/application-configuration/flex-runtime-configuration.model.ts` |

### [COOL] flex-types

| Metric | Count |
|--------|------:|
| Total commit touches | **6** |
| Feature | 5 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 4 | 3 | 0 | 1 | `project.json` |
| 2 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 3 | 1 | 1 | 0 | 0 | `src/index.ts` |
| 4 | 1 | 1 | 0 | 0 | `tsconfig.json` |
| 5 | 1 | 1 | 0 | 0 | `eslint.config.mjs` |
| 6 | 1 | 1 | 0 | 0 | `src/lib/omit-function-props.type.ts` |
| 7 | 1 | 1 | 0 | 0 | `.eslintrc.json` |
| 8 | 1 | 1 | 0 | 0 | `src/lib/union.type.ts` |
| 9 | 1 | 1 | 0 | 0 | `tsconfig.spec.json` |
| 10 | 1 | 1 | 0 | 0 | `src/lib/typed-simple-changes.type.ts` |

### [COOL] flex-host

| Metric | Count |
|--------|------:|
| Total commit touches | **5** |
| Feature | 4 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 4 | 3 | 0 | 1 | `project.json` |
| 2 | 2 | 2 | 0 | 0 | `src/lib/flex-host.component.ts` |
| 3 | 2 | 2 | 0 | 0 | `src/test.ts` |
| 4 | 1 | 1 | 0 | 0 | `.eslintrc.json` |
| 5 | 1 | 1 | 0 | 0 | `src/lib/flex-host.module.ts` |
| 6 | 1 | 1 | 0 | 0 | `src/index.ts` |
| 7 | 1 | 1 | 0 | 0 | `tsconfig.spec.json` |
| 8 | 1 | 1 | 0 | 0 | `src/lib/test-utils/flex-host-test.module.ts` |
| 9 | 1 | 1 | 0 | 0 | `src/lib/flex-host.component.spec.ts` |
| 10 | 1 | 1 | 0 | 0 | `eslint.config.mjs` |

### [COOL] e2e-shared

| Metric | Count |
|--------|------:|
| Total commit touches | **4** |
| Feature | 3 |
| Bugfix | 0 |
| Refactor | 0 |
| Chore | 0 |
| Test | 0 |
| Docs | 0 |
| Other / version bumps | 1 |

**Top touched files:**

| # | Touches | Feat | Fix | Other | File |
|--:|--------:|-----:|----:|------:|------|
| 1 | 4 | 3 | 0 | 1 | `project.json` |
| 2 | 2 | 2 | 0 | 0 | `src/lib/cypress/onbase-config/support/onbase-config-commands.ts` |
| 3 | 2 | 2 | 0 | 0 | `.eslintrc.json` |
| 4 | 2 | 2 | 0 | 0 | `src/lib/cypress/iframe/switch-to-iframe.ts` |
| 5 | 2 | 2 | 0 | 0 | `src/lib/cypress/onbase-config/plugins/onbase-config-plugin.ts` |
| 6 | 1 | 1 | 0 | 0 | `eslint.config.mjs` |
| 7 | 1 | 1 | 0 | 0 | `README.md` |
| 8 | 1 | 1 | 0 | 0 | `tsconfig.json` |
| 9 | 1 | 1 | 0 | 0 | `src/index.ts` |
| 10 | 1 | 1 | 0 | 0 | `tsconfig.lib.json` |

---

## 4. Global Top-50 Hottest Files

| Rank | Touches | Feat | Fix | Other | Module | File |
|-----:|--------:|-----:|----:|------:|--------|------|
| 1 | 77 | 35 | 22 | 20 | onbase-apps-host | `apps/onbase-apps-host/package.json` |
| 2 | 52 | 16 | 29 | 7 | client-app | `apps/client-app/package.json` |
| 3 | 33 | 22 | 6 | 5 | root/config | `package.json` |
| 4 | 33 | 20 | 5 | 8 | root/config | `Jenkinsfile` |
| 5 | 32 | 22 | 5 | 5 | onbase-apps-host | `apps/onbase-apps-host/src/app/features/sources/core/services/source-view-registry.service.ts` |
| 6 | 27 | 16 | 5 | 6 | root/config | `package-lock.json` |
| 7 | 26 | 18 | 3 | 5 | onbase-web-server | `libs/onbase-web-server/src/index.ts` |
| 8 | 23 | 15 | 5 | 3 | onbase-web-server | `libs/onbase-web-server/src/lib/source-views/workflow/components/workflow-queue.sv.component.ts` |
| 9 | 22 | 14 | 7 | 1 | wv-components | `libs/wv-components/src/lib/hy-ng-workview-object-viewer/core/services/object-service-bridge/object-service-bridge.service.ts` |
| 10 | 21 | 13 | 2 | 6 | workflow-approval-mgmt | `apps/workflow-approval-mgmt/package.json` |
| 11 | 21 | 12 | 4 | 5 | shared-components | `libs/shared-components/src/lib/hy-ng-data-grid/hy-ng-data-grid.component.ts` |
| 12 | 20 | 15 | 3 | 2 | onbase-apps-host | `apps/onbase-apps-host/string-constants.json` |
| 13 | 19 | 11 | 7 | 1 | onbase-web-server | `libs/onbase-web-server/src/lib/source-views/workview/components/workview-filter.sv.component.ts` |
| 14 | 18 | 10 | 7 | 1 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/common/work-item-in-queue-tasks-menu/work-item-in-queue-tasks-menu.component.ts` |
| 15 | 18 | 12 | 5 | 1 | onbase-web-server | `libs/onbase-web-server/src/lib/source-views/doc-query/components/document-content-grid/document-content-grid.sv.component.ts` |
| 16 | 17 | 15 | 1 | 1 | root/config | `apps/onbase-apps-host-e2e/src/e2e/index.app.ts` |
| 17 | 17 | 10 | 3 | 4 | onbase-web-server | `libs/onbase-web-server/src/lib/source-views/workflow/components/workflow-queue.sv.component.html` |
| 18 | 17 | 9 | 6 | 2 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/searching/search-panel/search-panel.component.ts` |
| 19 | 17 | 10 | 6 | 1 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/common/work-item-tasks-menu/work-item-tasks-menu.component.ts` |
| 20 | 17 | 12 | 3 | 2 | dev-app | `apps/dev-app/string-constants.json` |
| 21 | 16 | 11 | 0 | 5 | wf-approval-mgmt-lib | `libs/wf-approval-mgmt-lib/src/lib/shared/data-access/services/approval-processes.service.ts` |
| 22 | 16 | 12 | 3 | 1 | onbase-apps-host | `apps/onbase-apps-host/src/app/features/source-view-presenter/components/hy-ng-source-view-presenter.component.ts` |
| 23 | 16 | 7 | 5 | 4 | shared-components | `libs/shared-components/src/lib/hy-ng-data-grid/hy-ng-data-grid.component.html` |
| 24 | 15 | 10 | 0 | 5 | wf-approval-mgmt-lib | `libs/wf-approval-mgmt-lib/src/lib/shared/data-access/services/approval-processes.service.spec.ts` |
| 25 | 15 | 9 | 5 | 1 | wv-components | `libs/wv-components/src/lib/hy-ng-workview-object-viewer/core/services/workview-controller-object/workview-controller-object.service.ts` |
| 26 | 15 | 10 | 3 | 2 | client-app | `apps/client-app/string-constants.json` |
| 27 | 15 | 8 | 5 | 2 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/viewer/viewer.component.ts` |
| 28 | 15 | 9 | 3 | 3 | onbase-web-server | `libs/onbase-web-server/src/lib/source-views/doc-query/components/document-content-grid/document-content-grid.sv.component.html` |
| 29 | 15 | 11 | 2 | 2 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/data-grid/components/content-data-grid.component.ts` |
| 30 | 14 | 11 | 2 | 1 | shared-components | `libs/shared-components/src/lib/hy-ng-data-grid/hy-ng-data-grid.component.spec.ts` |
| 31 | 14 | 10 | 0 | 4 | wf-approval-mgmt-lib | `libs/wf-approval-mgmt-lib/src/lib/shared/data-access/facade/approval-process-details-facade.service.ts` |
| 32 | 14 | 9 | 2 | 3 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/searching/search-panel/search-panel.component.html` |
| 33 | 13 | 11 | 1 | 1 | onbase-apps-host | `apps/onbase-apps-host/src/app/features/master-detail-w-sources/components/hy-ng-master-detail-w-sources/hy-ng-master-detail-w-sources.component.html` |
| 34 | 13 | 8 | 4 | 1 | onbase-apps-host | `apps/onbase-apps-host/src/app/features/master-detail-w-sources/components/hy-ng-master-detail-w-sources/hy-ng-master-detail-w-sources.component.ts` |
| 35 | 13 | 8 | 4 | 1 | onbase-apps-host | `apps/onbase-apps-host/src/app/style-components/theme/app.theme.scss` |
| 36 | 13 | 6 | 4 | 3 | root/config | `apps/onbase-apps-host-e2e/src/e2e/general-runtime/workflow/workflow-sources.cy.ts` |
| 37 | 13 | 7 | 3 | 3 | onbase-web-server | `libs/onbase-web-server/src/lib/source-views/workview/components/workview-filter.sv.component.html` |
| 38 | 13 | 6 | 4 | 3 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/related-items/source-view-related-items.component.ts` |
| 39 | 12 | 5 | 0 | 7 | root/config | `apps/workflow-approval-mgmt-e2e/src/page_objects/approvalProcessPage.ts` |
| 40 | 12 | 11 | 0 | 1 | onbase-web-server | `libs/onbase-web-server/src/lib/shared/dialog-service/modal-dialog-container/modal-dialog-container.component.ts` |
| 41 | 12 | 11 | 1 | 0 | onbase-apps-host | `apps/onbase-apps-host/src/app/features/source-view-presenter/components/hy-ng-source-view-presenter.component.spec.ts` |
| 42 | 12 | 10 | 2 | 0 | onbase-apps-host | `apps/onbase-apps-host/src/app/features/master-detail-w-sources/components/hy-ng-master-detail-w-sources/hy-ng-master-detail-w-sources.component.spec.ts` |
| 43 | 12 | 7 | 1 | 4 | root/config | `apps/onbase-apps-host-e2e/src/e2e/general-runtime/master-detail.cy.ts` |
| 44 | 12 | 10 | 1 | 1 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/workflow/components/item-viewer.component.ts` |
| 45 | 12 | 7 | 0 | 5 | wf-approval-mgmt-lib | `libs/wf-approval-mgmt-lib/src/lib/shared/components/hy-ng-conditions/hy-ng-conditions.component.html` |
| 46 | 11 | 9 | 0 | 2 | shared-components | `libs/shared-components/src/lib/menu-with-buttons/menu-with-buttons.component.html` |
| 47 | 11 | 5 | 1 | 5 | dev-app | `apps/dev-app/package.json` |
| 48 | 11 | 9 | 0 | 2 | shared-components | `libs/shared-components/src/lib/shared-components.module.ts` |
| 49 | 11 | 8 | 2 | 1 | root/config | `.bitbucket/CODEOWNERS` |
| 50 | 11 | 8 | 1 | 2 | onbase-web-server | `libs/onbase-web-server/src/lib/modules/content/common/workflow-dialog/workflow-dialog.ts` |

---

*Generated by analysing `git log --since=3.years.ago` on 2026-03-05*
