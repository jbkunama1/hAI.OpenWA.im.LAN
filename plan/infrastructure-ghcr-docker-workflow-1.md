---
goal: Add ghcr.io Docker image build & push workflow and update documentation
version: 1.0
date_created: 2026-09-04
last_updated: 2026-09-04
owner: jbkunama1
status: 'Planned'
tags: ['infrastructure', 'docker', 'github-actions', 'ghcr', 'ci-cd']
---

# Introduction

![Status: Planned](https://img.shields.io/badge/status-Planned-blue)

This plan adds a GitHub Actions workflow to automatically build and push Docker images for the OpenWA API and Dashboard to ghcr.io (GitHub Container Registry). It also updates the README documentation to reference the pre-built ghcr.io images, allowing users to deploy without local Docker builds.

## 1. Requirements & Constraints

- **REQ-001**: Build and push two Docker images to ghcr.io: `openwa-api` and `openwa-dashboard`
- **REQ-002**: Trigger on push to main branch and on version tags (v*)
- **REQ-003**: Use multi-platform builds (linux/amd64, linux/arm64) for broad compatibility
- **REQ-004**: Tag images with: `latest`, semantic version from tag, and git SHA
- **REQ-005**: Use GitHub Actions built-in GITHUB_TOKEN for ghcr.io authentication
- **REQ-006**: Update README to document ghcr.io image usage as alternative to local builds
- **SEC-001**: No secrets in workflow - use GITHUB_TOKEN with packages:write permission
- **CON-001**: Repository must have "Packages" permission enabled in Settings > Actions > General
- **CON-002**: Images must be public or user must have access to private ghcr.io packages
- **GUD-001**: Follow Docker best practices (layer caching, multi-stage builds)
- **PAT-001**: Use docker/build-push-action@v6 with cache-from/to for build speed

## 2. Implementation Steps

### Implementation Phase 1 - Create ghcr.io Docker Build Workflow

- GOAL-001: Create `.github/workflows/docker-build-push.yml` for building and pushing both images to ghcr.io

| Task     | Description           | Completed | Date       |
| -------- | --------------------- | --------- | ---------- |
| TASK-001 | Create docker-build-push.yml workflow file |           |            |
| TASK-002 | Configure workflow triggers (push to main, tags v*) |           |            |
| TASK-003 | Add permissions for packages:write and contents:read |           |            |
| TASK-004 | Set up Docker Buildx for multi-platform builds |           |            |
| TASK-005 | Build openwa-api image from upstream OpenWA repo |           |            |
| TASK-006 | Build openwa-dashboard image from local dashboard folder |           |            |
| TASK-007 | Push images with multiple tags (latest, semver, sha) |           |            |
| TASK-008 | Test workflow by pushing a test tag |           |            |

### Implementation Phase 2 - Update README Documentation

- GOAL-002: Update README.md and README_EN.md to document ghcr.io image usage

| Task     | Description           | Completed | Date       |
| -------- | --------------------- | --------- | ---------- |
| TASK-009 | Add ghcr.io image references to README.md (German) |           |            |
| TASK-010 | Add ghcr.io image references to README_EN.md (English) |           |            |
| TASK-011 | Document image tags available (latest, vX.Y.Z, sha) |           |            |
| TASK-012 | Add example Portainer stack using ghcr.io images |           |            |
| TASK-013 | Update "Voraussetzungen" / "Prerequisites" section |           |            |

### Implementation Phase 3 - Verify and Improve Existing Workflows

- GOAL-003: Review and enhance trufflehog workflow if needed

| Task     | Description           | Completed | Date       |
| -------- | --------------------- | --------- | ---------- |
| TASK-014 | Review trufflehog.yml for completeness |           |            |
| TASK-015 | Add PR scan trigger to trufflehog workflow |           |            |
| TASK-016 | Verify space-shooter.yml is acceptable as-is |           |            |

## 3. Alternatives

- **ALT-001**: Use Docker Hub instead of ghcr.io - Rejected: ghcr.io is native to GitHub, no extra account needed, integrated permissions
- **ALT-002**: Build images only on release tags - Rejected: Users need `latest` for continuous deployment testing
- **ALT-003**: Single combined workflow for all CI - Rejected: Separation of concerns (security scan vs build) is cleaner

## 4. Dependencies

- **DEP-001**: GitHub Actions enabled on repository
- **DEP-002**: Repository Settings > Actions > General > Workflow permissions: "Read and write permissions" + "Allow GitHub Actions to create and approve pull requests"
- **DEP-003**: Packages permission enabled for GITHUB_TOKEN
- **DEP-004**: Upstream OpenWA repository (https://github.com/rmyndharis/OpenWA) accessible for API builds

## 5. Files

- **FILE-001**: `.github/workflows/docker-build-push.yml` (NEW) - Main Docker build and push workflow
- **FILE-002**: `README.md` (MODIFY) - Add ghcr.io usage documentation (German)
- **FILE-003**: `README_EN.md` (MODIFY) - Add ghcr.io usage documentation (English)
- **FILE-004**: `.github/workflows/trufflehog.yml` (MODIFY) - Enhance with PR trigger

## 6. Testing

- **TEST-001**: Push to main branch triggers workflow and publishes images to ghcr.io
- **TEST-002**: Create and push version tag (e.g., v1.0.0) creates versioned image tags
- **TEST-003**: Verify multi-platform images (amd64, arm64) are available
- **TEST-004**: Test Portainer stack deployment using ghcr.io images
- **TEST-005**: Verify trufflehog runs on PRs and scheduled scans

## 7. Risks & Assumptions

- **RISK-001**: Upstream OpenWA repo changes Dockerfile structure breaking API build
- **RISK-002**: ghcr.io rate limits for anonymous pulls (mitigated by authenticated pulls)
- **RISK-003**: Dashboard build failures due to upstream dependency conflicts (vite@8 issue)
- **ASSUMPTION-001**: User has admin access to repository to enable package permissions
- **ASSUMPTION-002**: Node 20 + --legacy-peer-deps resolves dashboard build issues

## 8. Related Specifications / Further Reading

- [GitHub Container Registry documentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [docker/build-push-action](https://github.com/docker/build-push-action)
- [OpenWA upstream repository](https://github.com/rmyndharis/OpenWA)
- [TruffleHog GitHub Action](https://github.com/trufflesecurity/trufflehog)