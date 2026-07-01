# Permissions CLI

`calypr-cli permissions` is the operator-facing CLI for the Arborist routes
that are actually exposed through the public Gen3 `/authz` surface.

This is intentionally not a full Arborist admin client. Raw Arborist catalog
routes like user, role, and resource CRUD are not exposed through revproxy, so
`calypr-cli` does not try to support them.

Use `calypr-cli collaborators` when a user is asking for access they do not
already have. Use `calypr-cli permissions` when an admin needs to inspect or
change the ownership and direct-access state that Arborist exposes publicly.

## Basic Shape

All commands use the normal calypr-cli profile:

```bash
calypr-cli --profile calypr permissions <command>
```

Add `--json` when you need raw output for debugging or scripts:

```bash
calypr-cli --profile calypr permissions --json auth mapping
```

## Current Auth Mapping

Show the current profile user's Arborist mapping:

```bash
calypr-cli --profile calypr permissions auth mapping
```

This command reads the public `GET /authz/mapping` surface for the current
token. It does not support arbitrary username lookups.

## Organization Membership

Organization membership is a convenience wrapper around Arborist direct-access
grants on `/programs/<org>/projects`.

```bash
calypr-cli --profile calypr permissions org-membership add user@ohsu.edu Ellrott_Lab
calypr-cli --profile calypr permissions org-membership rm user@ohsu.edu Ellrott_Lab
```

The default role is `org-member`. It carries only
`arborist/create-descendant`, which lets the member create projects under the
existing organization without granting ownership or access on existing projects.

You can specify another role when needed:

```bash
calypr-cli --profile calypr permissions org-membership add user@ohsu.edu Ellrott_Lab --role org-member
```

Do not pass a resource path to `org-membership`. This is valid:

```bash
Ellrott_Lab
```

This is invalid:

```bash
/programs/Ellrott_Lab
```

## Ownership Power Tools

Create a new organization resource and make the caller its owner:

```bash
calypr-cli --profile calypr permissions ownership create-descendant \
  --parent /programs \
  --name Ellrott_Lab \
  --template gen3-program
```

Create a new project resource under an organization and make the caller its
owner:

```bash
calypr-cli --profile calypr permissions ownership create-descendant \
  --parent /programs/Ellrott_Lab/projects \
  --name git_drs_test \
  --template gen3-project
```

Add or remove owners:

```bash
calypr-cli --profile calypr permissions ownership add-owner \
  --resource /programs/Ellrott_Lab \
  --user user@ohsu.edu

calypr-cli --profile calypr permissions ownership rm-owner \
  --resource /programs/Ellrott_Lab \
  --user user@ohsu.edu
```

Read the normalized ownership and direct-access state for a resource:

```bash
calypr-cli --profile calypr permissions ownership get-resource \
  --resource /programs/Ellrott_Lab/projects/git_drs_test \
  --include-admins \
  --include-children
```

## Direct Access Power Tools

Grant or revoke direct non-owner access on an existing resource:

```bash
calypr-cli --profile calypr permissions access grant-user \
  --resource /programs/Ellrott_Lab/projects/git_drs_test \
  --user user@ohsu.edu \
  --role writer

calypr-cli --profile calypr permissions access revoke-user \
  --resource /programs/Ellrott_Lab/projects/git_drs_test \
  --user user@ohsu.edu \
  --role writer
```

Use these commands for ordinary direct access. Use ownership add/remove for
owner changes.

## Deliberate Non-Support

`calypr-cli permissions` does not support raw Arborist catalog/admin routes such
as:

- user CRUD
- role CRUD
- resource CRUD
- raw policy mutation
- arbitrary-user auth mapping lookup

Those routes are not part of the supported public Gen3 revproxy surface, so the
CLI does not expose them.

The legacy backend-oriented command name `calypr-cli arborist` still works as a
compatibility alias, but `permissions` is the supported user-facing name.

## Templates

### Understanding Ownership Templates

Ownership templates define **what kind of resource you are creating** in the Gen3 authorization hierarchy. Rather than creating an empty node, a template tells Arborist how to initialize the resource, including its default ownership model, permissions, and inheritance behavior.

Think of a template as analogous to a project template in GitHub or a class in object-oriented programming: it provides the initial structure and behavior for a new resource.

---

### The Resource Hierarchy

Most Gen3 deployments organize resources into a hierarchy similar to:

```text
/programs
└── Ellrott_Lab
    └── projects
        ├── git_drs_test
        ├── cohort_analysis
        └── clinical_validation
```

Templates determine what kind of node is created within this hierarchy.

---

### Available Templates

The current CLI documentation references two ownership templates.

#### `gen3-program`

Creates a new **organization (program)** beneath `/programs`.

Example:

```bash
calypr-cli --profile calypr permissions ownership create-descendant \
  --parent /programs \
  --name Ellrott_Lab \
  --template gen3-program
```

Result:

```text
/programs
└── Ellrott_Lab
```

Use this template when:

* Creating a new organization
* Creating a new laboratory
* Creating a new consortium
* Creating a new institution
* Creating the top-level container that will own one or more projects

Typically, administrators perform this operation only occasionally.

---

#### `gen3-project`

Creates a new project beneath an existing organization.

Example:

```bash
calypr-cli --profile calypr permissions ownership create-descendant \
  --parent /programs/Ellrott_Lab/projects \
  --name git_drs_test \
  --template gen3-project
```

Result:

```text
/programs
└── Ellrott_Lab
    └── projects
        └── git_drs_test
```

The creator automatically becomes the owner of the new project.

Use this template when:

* Starting a new research project
* Creating a new data collection effort
* Organizing files into an independent security boundary
* Creating a project with its own collaborators and permissions

This is the template most users will use.

---

### Choosing the Right Template

| If you want to create...                                   | Use            |
| ---------------------------------------------------------- | -------------- |
| A new organization, laboratory, consortium, or institution | `gen3-program` |
| A project within an existing organization                  | `gen3-project` |

In most environments:

* Administrators create **programs**.
* Researchers and project administrators create **projects**.

---

### What Does a Template Actually Do?

A template tells Arborist how to initialize a newly created resource.

Although the exact implementation is deployment-specific, a template typically defines:

* the resource type
* the ownership model
* inherited permissions
* default access policies
* any metadata required by the authorization system

The CLI simply passes the template name to the server. The server is responsible for interpreting the template and creating the appropriate authorization resources.

---

### Where Are Templates Defined?

Templates are **not defined by the CLI**.

They are configured on the Gen3/Arborist server, meaning different deployments may provide additional templates beyond those documented here.

The CLI currently documents these templates:

* `gen3-program`
* `gen3-project`

Additional templates may exist depending on your deployment.

---

### Discovering Available Templates

At present, the CLI does not provide a command to list available templates.

If you need to know which templates are installed, you have several options:

* consult your Gen3 administrator
* review your deployment's Arborist configuration
* attempt to create a resource using a template name (the server will reject unknown templates)

A future version of the CLI may provide a command such as:

```bash
calypr-cli permissions ownership templates
```

to enumerate the templates available on the connected server.

---

### Typical Workflow

Most users follow this lifecycle:

1. Configure a CLI profile.

   ```bash
   calypr-cli configure ...
   ```

2. Create (or be added to) an organization.

3. Create a project.

   ```bash
   calypr-cli permissions ownership create-descendant \
       --parent /programs/Ellrott_Lab/projects \
       --name git_drs_test \
       --template gen3-project
   ```

4. Grant collaborators access.

   ```bash
   calypr-cli permissions access grant-user ...
   ```

5. Upload data into the project.

For most researchers, **`gen3-project` is the only template they will routinely use.**

