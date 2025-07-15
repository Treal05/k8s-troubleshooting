h1. Kustomize: Troubleshooting Common Issues

This guide provides a brief overview of Kustomize and then dives into common troubleshooting scenarios you might encounter when using it.

h2. What is Kustomize?

Kustomize is a standalone tool to customize Kubernetes objects through a file called `kustomization.yaml`. It allows you to manage your Kubernetes configurations in a declarative way, without templating. You start with a "base" configuration and then apply "overlays" to customize it for different environments (e.g., development, staging, production). This makes it easier to manage application configuration for multiple environments without duplicating a lot of YAML.

Key features of Kustomize include:

*   Adding common labels or annotations to all resources.
*   Changing resource names with a prefix or suffix.
*   Merging or patching existing resources.
*   Generating ConfigMaps and Secrets from files or literals.

---

h2. How to Use This Guide for Kustomize Troubleshooting

When you encounter an issue with your Kustomize configurations, the primary tool for diagnosis is the `kustomize build` command. This command processes your `kustomization.yaml` and all referenced resources, outputting the final, merged Kubernetes YAML. Any errors in your Kustomize setup will typically manifest during this build process.

*   **`kustomize build <path-to-kustomization>`**
    *   **Purpose:** Generates the final Kubernetes manifest by applying all Kustomize transformations (bases, overlays, patches, etc.). This command *does not* interact with your Kubernetes cluster; it only performs local validation and rendering.
    *   **Usage:**
        *   `kustomize build .`: Builds the Kustomize configuration in the current directory.
        *   `kustomize build overlays/production`: Builds a specific overlay.
    *   **What to look for:**
        *   **Error Messages:** Kustomize will output detailed error messages if it encounters issues like missing files, invalid YAML syntax, or patching conflicts. These messages are crucial for identifying the problem.
        *   **Generated YAML:** Even if there are no errors, inspecting the generated YAML can help you verify that Kustomize is producing the desired output. You can pipe the output to a file for easier review: `kustomize build . > output.yaml`.

Once you observe an error message from `kustomize build`, you can then refer to the scenarios below to find a matching problem description and its solution.

---

h2. Troubleshooting Scenarios

h3. Scenario 1: Resource File Not Found

This is one of the most common errors, where a resource file specified in the `kustomization.yaml` does not exist at the given path.

*Error Message:*

When you run `kustomize build .`, you will see an error similar to this:

{code}
Error: accumulating resources: accumulation err='accumulating resources from '../base': '../base/deployment.yaml': no such file or directory': recursed from path '.'
{code}

*Cause:*

The `kustomization.yaml` file contains a reference to a resource file that does not exist at the specified location. This could be due to a typo in the filename, an incorrect path, or the file being deleted or moved.

*Example `kustomization.yaml` with the error:*

{code:yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml  # Correct file is my-deployment.yaml
- service.yaml
{code}

*Solution:*

1.  **Check the path:** Verify that the file path in the `kustomization.yaml` is correct.
2.  **Check for typos:** Ensure the filename in the `kustomization.yaml` exactly matches the actual filename.
3.  **Verify the file exists:** Use `ls` or `tree` to confirm that the file is present in the expected directory.

*Corrected `kustomization.yaml`:*

{code:yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- my-deployment.yaml
- service.yaml
{code}

h3. Scenario 2: Invalid YAML or Kustomize Syntax

This error category covers syntax problems in any of the files Kustomize needs to parse, which can be the `kustomization.yaml` itself or any of the referenced resource files (e.g., `deployment.yaml`, `service.yaml`).

*Error Message:*
The error message can vary, but it will typically indicate a problem with unmarshaling or decoding YAML/JSON.

{code}
Error: unable to load kustomization file: unable to unmarshal JSON: while decoding JSON: json: unknown field "resorces"
{code}
or
{code}
Error: accumulating resources: malformed file from path 'deployment.yaml': error converting YAML to JSON: yaml: line 10: could not find expected ':'
{code}

*Cause A: Syntax error in `kustomization.yaml`*

This happens when the `kustomization.yaml` file has a typo, incorrect indentation, or uses a field name that is not part of the Kustomize schema.

*Example `kustomization.yaml` with the error:*

{code:yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resorces: # Typo here, should be "resources"
- deployment.yaml
{code}

*Solution A:*

Carefully review the `kustomization.yaml` file for any syntax errors. Pay close attention to indentation and field names. Refer to the official Kustomize documentation for the correct schema.

*Corrected `kustomization.yaml`:*

{code:yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
{code}

*Cause B: Syntax error in a referenced resource file*

This happens when a file listed in the `resources` section (like `deployment.yaml`) is not a valid YAML file. Kustomize must parse all resource files to process them, and it will fail if it encounters a syntax error.

*Example `deployment.yaml` with the error:*

{code:yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas 3 # Missing colon here
  selector:
    matchLabels:
      app: my-app
  template:
    ...
{code}

*Solution B:*

1.  The error message usually points to the file and sometimes the line number causing the issue.
2.  Examine the referenced file (e.g., `deployment.yaml`) for YAML syntax errors.
3.  Common errors include incorrect indentation (YAML is very sensitive to spaces), missing colons, or mismatched quotes.
4.  Use a YAML linter or an IDE with YAML validation to help find the error.

*Corrected `deployment.yaml`:*

{code:yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3 # Corrected line
  selector:
    matchLabels:
      app: my-app
  template:
    ...
{code}

h3. Scenario 3: Patching a Non-existent Field

This happens when you try to apply a patch to a field that does not exist in the base resource.

*Error Message:*

{code}
Error: strategic merge patch for "..." failed to apply: missing field "spec.template.spec.containers[name=my-app].ports[port=8080]" in existing object
{code}

*Cause:*

The patch is trying to modify a part of the resource that doesn't exist. For example, you might be trying to patch a container port that isn't defined in the base deployment.

*Example `deployment.yaml` (base):*

{code:yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: nginx
{code}

*Example `kustomization.yaml` with the error:*

{code:yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml

patchesStrategicMerge:
- |- 
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      template:
        spec:
          containers:
          - name: my-app
            # This tries to patch a port that doesn't exist in the base
            ports:
            - containerPort: 8080
{code}

*Solution:*

Ensure that the patch is targeting a field that actually exists in the base resource. If you want to add a new field, you might need to use a different patching mechanism like JSON patches, or ensure the base object has the structure you intend to patch.

*Corrected approach (adding the port in the base or using a different patch type):*

If the intention is to *add* the port, a JSON patch is more appropriate:

{code:yaml}
patches:
- path: patch.yaml
  target:
    group: apps
    version: v1
    kind: Deployment
    name: my-app
{code}

*`patch.yaml`:*
{code:yaml}
- op: add
  path: /spec/template/spec/containers/0/ports
  value:
  - containerPort: 8080
{code}

h3. Scenario 4: Name Prefix/Suffix Conflicts

Using name prefixes or suffixes can sometimes lead to conflicts if not managed carefully, especially with service selectors.

*Problem Description:*

When you add a `namePrefix` or `nameSuffix` to your `kustomization.yaml`, Kustomize will update the `metadata.name` of all your resources. However, it does *not* automatically update all references to those names, such as a Service's `spec.selector`.

*Example `kustomization.yaml`:*

{code:yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namePrefix: dev-

resources:
- deployment.yaml
- service.yaml
{code}

*Resulting issue:*

*   The Deployment will be named `dev-my-app`.
*   The Service will be named `dev-my-service`.
*   The Service's selector will still be pointing to `app: my-app`, but the Deployment's pods will now have a label `app: dev-my-app` (Kustomize updates this). This mismatch means the Service won't select any pods.

*Solution:*

Kustomize is smart enough to update the labels and selectors to match the new names. The problem usually arises when the selectors are not defined in a way Kustomize understands. To fix this, ensure your base resources have consistent and clear labels, and that your Service selectors match those labels. Kustomize will then update both the Pod labels and the Service selector correctly.

*Base `deployment.yaml`:*
{code:yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      ...
{code}

*Base `service.yaml`:*
{code:yaml}
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
{code}

With this structure, Kustomize will correctly update the `name`, the `selector`, and the `template.metadata.labels` to include the prefix.

h3. Scenario 5: Base or Overlay Not Found

This occurs when an overlay `kustomization.yaml` references a base that doesn't exist.

*Error Message:*

{code}
Error: accumulating resources: accumulation err='accumulating resources from '../base': '../base' does not exist or is not a directory': recursed from path '.'
{code}

*Cause:*

The `resources` field in an overlay's `kustomization.yaml` points to a directory that does not exist or does not contain a `kustomization.yaml` file.

*Example directory structure:*

{code}
. 
├── overlays
│   └── production
│       └── kustomization.yaml
└── base # This directory is missing
{code}

*Example `overlays/production/kustomization.yaml`:*

{code:yaml}
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base # This path is incorrect or the directory is missing
{code}

*Solution:*

Ensure that the path in the `resources` field of your overlay's `kustomization.yaml` correctly points to the directory containing your base `kustomization.yaml` file. Verify the directory structure and check for any typos in the path.
