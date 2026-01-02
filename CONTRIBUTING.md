# Contributing to Kuack

Thank you for your interest in contributing to Kuack! This document provides information about the development process and tools available to simplify development.

## Development Tools

To simplify the development process, we maintain two separate projects that work alongside the main kuack repository:

### Devspace

The [Devspace](https://github.com/kuack-io/devspace) project provides a development environment configuration for kuack using DevSpace on top of Minikube. It enables local development with hot reload capabilities, making it easy to iterate on changes without complex manual setup.

Key features:
- Local Kubernetes cluster setup with Minikube
- Hot reload for development
- Automatic port forwarding
- Integration with the main kuack codebase

For more information, see the [Devspace repository](https://github.com/kuack-io/devspace).

### E2E Test Suite

The [E2E](https://github.com/kuack-io/e2e) project contains the end-to-end test suite for Kuack applications. It uses TypeScript, Playwright, Cucumber, and Kubernetes integration to provide comprehensive testing capabilities.

Key features:
- Browser automation with Playwright
- Kubernetes API integration
- BDD with Gherkin/Cucumber
- Beautiful Allure reports
- Designed for extreme parallelization
- Runs as Kubernetes Jobs with in-cluster authentication

For more information, see the [E2E repository](https://github.com/kuack-io/e2e).

## Testing

Test coverage is currently low, but will be growing aggressively in the nearest future, as it's our top priority now.
