# Test Coverage Analysis

## Current State: Zero Test Coverage

This project currently has **no automated tests** of any kind — no unit tests, integration tests, end-to-end tests, or configuration validation. There is also no test framework, test runner, or test configuration in the project.

The CI/CD pipeline (`.github/workflows/docker-image.yml`) builds and pushes Docker images **without running any validation or tests**.

---

## Areas to Improve

### 1. HTML Validation Tests

**Priority: High**

The site's HTML files (`index.html`, `error/index.html`) are never validated. Broken markup can cause rendering issues across browsers.

**Proposed tests:**
- Validate HTML against W3C standards using a tool like `html-validate` or `htmlhint`
- Check for broken internal links (e.g., `#intro`, `#work`, `#about`, `#contact`)
- Verify all required meta tags are present (`charset`, `viewport`)
- Ensure all images have `alt` attributes

**Tooling:** Add `htmlhint` or `html-validate` as an npm dev dependency with a config file.

---

### 2. Dockerfile and Container Tests

**Priority: High**

The `Dockerfile` is deployed to production but never validated. A broken Dockerfile breaks the entire deployment pipeline.

**Proposed tests:**
- **Lint the Dockerfile** using `hadolint` to catch bad practices (e.g., using `latest` tag — which the current Dockerfile does)
- **Build smoke test:** Build the image and verify it starts successfully
- **Content test:** Verify that files are correctly copied to `/usr/local/apache2/htdocs/`
- **HTTP response test:** Start the container and verify it returns HTTP 200 on `/`
- **Container structure tests** using Google's `container-structure-test` tool to validate file presence and permissions

**Tooling:** `hadolint` for linting, `container-structure-test` for structural validation, `curl` or `wget` for HTTP smoke tests.

---

### 3. Kubernetes Manifest Validation

**Priority: High**

The Kubernetes manifests in `deployment/` are never validated. Invalid manifests will fail at deploy time.

**Proposed tests:**
- **Schema validation** using `kubeval` or `kubeconform` to check YAML against K8s API schemas
- **Port consistency:** Verify that `containerPort` in deployment.yaml (8080) is correct for Apache httpd (default is 80, not 8080 — this is likely a bug)
- **Label consistency:** Ensure `selector.matchLabels` matches `template.metadata.labels`
- **Service/Deployment alignment:** Verify the Service selector matches the Deployment labels and ports are consistent

**Note:** There is a likely existing bug — the Deployment specifies `containerPort: 8080` and the Service targets port `8080`, but the httpd Docker image serves on port **80** by default.

---

### 4. CI Pipeline Tests (GitHub Actions Workflow Validation)

**Priority: Medium**

The workflow file itself is never validated and could break on syntax errors.

**Proposed tests:**
- Lint the workflow YAML with `actionlint`
- Validate the workflow structure is correct
- Ensure required secrets are documented

---

### 5. JavaScript Unit Tests

**Priority: Medium**

The project includes ~1,000 lines of custom JavaScript across two files (`main.js` and `util.js`) that are completely untested.

**Proposed tests for `assets/js/util.js`:**
- Test the `navList()` jQuery plugin generates correct nav markup from lists
- Test the `panel()` plugin handles show/hide/toggle actions
- Test the `placeholder()` polyfill behavior
- Test the `$.prioritize()` function reorders elements correctly

**Proposed tests for `assets/js/main.js`:**
- Test hash-based routing logic (article show/hide on URL hash changes)
- Test that clicking nav links shows the correct article
- Test article lock/unlock state management
- Test window resize handler behavior

**Tooling:** Jest with jsdom, or Vitest for a modern alternative.

---

### 6. End-to-End / Integration Tests

**Priority: Medium**

No tests verify the site works as a whole in a real browser.

**Proposed tests:**
- Navigation between sections (Intro, Work, About, Contact) works
- Contact form fields are present and interactable
- Page loads without JavaScript console errors
- Responsive layout renders correctly at different viewport sizes
- 404 error page is served correctly

**Tooling:** Playwright or Cypress.

---

### 7. CSS/SASS Compilation Tests

**Priority: Low**

The project uses SASS (`.scss` files) that compile to CSS, but the build process is not validated.

**Proposed tests:**
- Verify SASS compiles without errors
- Lint stylesheets using `stylelint`

---

### 8. Accessibility Tests

**Priority: Low**

No accessibility testing is performed.

**Proposed tests:**
- Run `axe-core` or `pa11y` against the rendered page
- Verify proper heading hierarchy
- Ensure form inputs have associated labels (the Contact form has labels, but this should be validated)
- Check color contrast ratios

---

## Recommended Implementation Order

| Phase | Area | Effort | Impact |
|-------|------|--------|--------|
| 1 | Dockerfile linting + container smoke test | Low | High |
| 2 | Kubernetes manifest validation | Low | High |
| 3 | HTML validation | Low | Medium |
| 4 | CI workflow validation | Low | Medium |
| 5 | JavaScript unit tests | Medium | Medium |
| 6 | End-to-end tests | High | High |
| 7 | CSS/SASS linting | Low | Low |
| 8 | Accessibility tests | Medium | Medium |

## CI Pipeline Integration

All tests above should be added as steps in `.github/workflows/docker-image.yml` **before** the Docker build/push step, so that broken code is caught before it reaches the container registry. A suggested pipeline order:

1. Checkout code
2. Lint HTML, Dockerfile, K8s manifests, and workflow files
3. Run JavaScript unit tests
4. Build Docker image
5. Run container smoke tests
6. Run end-to-end tests against the container
7. Push image to registry (only if all tests pass)

## Potential Bugs Found During Analysis

1. **Port mismatch:** `deployment/deployment.yaml` specifies `containerPort: 8080` and `deployment/service.yaml` targets port `8080`, but the `httpd:latest` Docker image serves on port **80** by default. Unless a custom httpd config is provided (it is not), this will cause the service to be unreachable.

2. **Dockerfile uses `latest` tag:** `FROM httpd:latest` is non-deterministic and can cause unexpected breakages. Should pin to a specific version.

3. **Select option values are all "1":** In `index.html` line 306-309, all `<option>` elements in the Category dropdown have `value="1"`, making them indistinguishable on form submission.
