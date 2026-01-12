# ToolHive RFCs

This repository contains Requests for Comments (RFCs) for the ToolHive ecosystem. RFCs are design documents that describe significant changes, new features, or architectural decisions across any ToolHive project.

## What is an RFC?

An RFC (Request for Comments) is a design document that proposes a significant change to the ToolHive ecosystem. RFCs provide a consistent and controlled path for new features and changes to enter the project, ensuring that all stakeholders have an opportunity to provide feedback.

## When to Write an RFC

You should write an RFC for:

- New features that affect multiple components or repositories
- Significant architectural changes
- Changes that affect the public API or user-facing behavior
- Security-sensitive changes
- Cross-cutting concerns that span multiple ToolHive projects
- Breaking changes or deprecations

You probably **don't** need an RFC for:

- Bug fixes
- Documentation improvements
- Minor refactoring
- Performance improvements that don't change behavior
- Changes isolated to a single component with no external impact

## ToolHive Ecosystem

This RFC repository serves the entire ToolHive ecosystem, including but not limited to:

| Repository | Description |
|------------|-------------|
| [toolhive](https://github.com/stacklok/toolhive) | Core ToolHive runtime and CLI |
| [toolhive-studio](https://github.com/stacklok/toolhive-studio) | Desktop application for managing MCP servers |
| [toolhive-registry](https://github.com/stacklok/toolhive-registry) | ToolHive's registry of MCP servers |
| [toolhive-registry-server](https://github.com/stacklok/toolhive-registry-server) | API server implementing the MCP Registry API |
| [toolhive-cloud-ui](https://github.com/stacklok/toolhive-cloud-ui) | Cloud UI for MCP servers |

## RFC Process

### 1. Pre-RFC Discussion (Optional)

Before writing a full RFC, consider opening a thread on [Discord](https://discord.gg/stacklok) to gather initial feedback on your idea. This can help refine the proposal before investing time in a full RFC.

### 2. Create the RFC

1. Fork this repository
2. Copy `rfcs/0000-template.md` to `rfcs/XXXX-descriptive-name.md`
   - Use the next available Pull Request number for your RFC (check existing RFCs)
   - Use a short, descriptive name with hyphens
3. Fill in the RFC template
4. Submit a Pull Request

### 3. RFC Review

- The RFC will be reviewed by maintainers and community members
- Feedback will be provided via PR comments
- The author should address feedback and update the RFC as needed
- Discussion should focus on technical merit and alignment with project goals

### 4. RFC Decision

RFCs can be:
- **Accepted**: The RFC is approved and can be implemented
- **Rejected**: The RFC is not approved (with explanation)
- **Postponed**: The RFC is deferred for future consideration
- **Withdrawn**: The author withdraws the RFC

### 5. Implementation

Once accepted, the RFC can be implemented. The RFC should be updated with:
- Links to implementation PRs
- Any deviations from the original design
- Final status (implemented, partially implemented, superseded)

## RFC Numbering

RFCs are numbered based on the PR numbers, so they are incremental, but not necessarily sequential (0001, 0002, 0004, etc.). When creating a new RFC, check the existing RFCs and use the next available number. A CI task will ensure you're using the right number.

For RFCs that originate from issues in specific repositories, you may reference the issue number in the RFC (e.g., "This RFC addresses toolhive#1234").

## Directory Structure

```
toolhive-rfcs/
├── README.md                    # This file
├── CONTRIBUTING.md              # Contribution guidelines
├── rfcs/
│   ├── 0000-template.md         # RFC template
│   └── ...
└── assets/                      # Images and diagrams for RFCs
    └── XXXX/                    # Assets for RFC XXXX
```

## License

This repository is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed guidelines on writing and submitting RFCs.
