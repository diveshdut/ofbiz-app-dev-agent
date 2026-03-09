# OFBiz Agent Skills Plugin

## Overall Goal
The `ofbiz-skills` plugin is a repository of standardized knowledge, best practices, and implementation patterns for Apache OFBiz. Its primary purpose is to empower AI agents (like Antigravity) to perform complex OFBiz development tasks—such as creating entities, defining services, building screens, and managing business logic—with high accuracy and adherence to framework-specific guardrails.

By providing these "skills" in a structured format, we ensure that agents follow the correct design patterns (e.g., favoring View Entities over manual iteration, using Worker classes correctly, adhering to security standards) without manual intervention for every step.

## Agent Setup & Activation

For an AI agent (like Antigravity) to effectively use these skills, they need to be accessible to the agent within your project workspace.

### 1. Framework Location (Plugin)
Keep the source of these skills in your OFBiz plugins directory so they remain version-controlled alongside your project:
`~/ofbiz-framework/plugins/ofbiz-skills/`

### 2. Workspace Agent Directory (Activation)
Rather than installing these globally in the agent's home directory, you should activate them locally for your OFBiz project. Gemini/Antigravity looks for skills in the `.agents/skills/` directory at the root of your workspace.

### 🛠️ Recommended Setup: Symlinking
To maintain the skills in your GitHub-tracked plugin while enabling them for the local agent, create a symbolic link from your OFBiz root:

```bash
# Navigate to your OFBiz framework root
cd ~/ofbiz-framework

# Create the agents directory if it doesn't exist
mkdir -p .agents

# Symlink the entire ofbiz-skills plugin to the workspace skills directory
ln -s plugins/ofbiz-skills .agents/skills
```

## How to Use with an Agent
Once symlinked, the agent will automatically discover these skills when working in your OFBiz project directory.

### Integration Patterns
- **Contextual Loading**: The agent natively supports the `.agents/skills/` folder and will index these files as part of its available skill set.
- **Triggered Reading**: Instruct the agent to "trigger" a skill read when it detects a specific file type extension (e.g., `.rest.xml` triggers `manage-api-integration`).

## Directory Structure
These skills should be placed within the `plugins` directory of your OFBiz framework:
`/path/to/ofbiz/plugins/ofbiz-skills/`

Each directory contains:
- `SKILL.md`: The core knowledge file containing the goal, procedures, and guardrails for that specific skill.

## Available Skills
For a detailed list of all skills with short descriptions, see [SKILLS_SUMMARY.md](~/ofbiz-skills/SKILLS_SUMMARY.md).

The toolkit currently covers:
- **Core Abstractions**: `manage-entities`, `manage-services`, `manage-data`.
- **UI & Interaction**: `manage-screens`, `manage-forms`, `manage-menus`, `manage-templates`, `manage-ajax`.
- **Logic & Flows**: `manage-groovy`, `manage-java`, `manage-java-patterns`, `manage-eca`, `manage-service-groups`.
- **Integrations**: `manage-api-integration`, `manage-email-services`.
- **Advanced Management**: `manage-security-advanced`, `manage-localization-advanced`, `manage-webapps`, `manage-cache-and-performance`.
- **Strategies**: `manage-strategies` (Xml vs Java/Groovy, Dos and Donts).

## Deployment and Updates
This plugin is linked to the GitHub repository: `https://github.com/arunpati/ofbiz-agent-skills.git`. To update the skills available to your agent, pull the latest changes from the repository.
