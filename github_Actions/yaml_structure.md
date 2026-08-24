Workflow
│
├── name
│
├── on
│   ├── push
│   ├── pull_request
│   ├── workflow_dispatch
│   ├── schedule
│   ├── workflow_call
│   └── workflow_run
│
├── permissions
│
├── env
│
├── concurrency
│
└── jobs
    │
    ├── lint
    │
    ├── test
    │
    ├── build
    │   ├── matrix
    │   ├── cache
    │   └── artifacts
    │
    ├── security
    │
    ├── integration-test
    │   └── services
    │
    ├── deploy-development
    │
    ├── deploy-staging
    │   └── environment
    │
    ├── deploy-production
    │   ├── environment
    │   ├── approval
    │   └── secrets
    │
    └── notify
