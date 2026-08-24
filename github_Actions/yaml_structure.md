Generic structure at a glance
.github/
└── workflows/
    ├── ci.yml
    ├── cd.yml
    ├── release.yml
    └── reusable-deploy.yml


And conceptually:

Workflow
│
├── name
├── on
│   ├── push
│   ├── pull_request
│   ├── workflow_dispatch
│   ├── schedule
│   ├── workflow_call
│   └── workflow_run
│
├── permissions
├── env
├── concurrency
│
└── jobs
    │
    ├── lint
    ├── test
    ├── build
    │   ├── matrix
    │   ├── cache
    │   └── artifacts
    │
    ├── security
    ├── integration-test
    │   └── services
    │
    ├── deploy-dev
    ├── deploy-staging
    │   └── environment
    │
    ├── deploy-production
    │   ├── environment
    │   ├── approval
    │   └── secrets
    │
    └── notify


This is intentionally a reference/template, rather than something you would use unchanged in production. You would normally split CI, deployment, release, and reusable workflows into separate YAML files once the repository grows.
