
### Script: `delete_stuck_namespace.sh`

```bash
#!/bin/bash

# Function to display usage
usage() {
    echo "Usage: $0 <namespace>"
    exit 1
}

# Function to check the namespace status
check_namespace_status() {
    local ns=$1
    kubectl get namespace $ns -o json | jq -r '.status.phase'
}

# Check if namespace parameter is provided
if [ -z "$1" ]; then
    usage
fi

NAMESPACE=$1

# Validate current namespace status
echo "Checking current status of namespace $NAMESPACE..."
current_status=$(check_namespace_status $NAMESPACE)
echo "Current status: $current_status"

if [ "$current_status" != "Terminating" ]; then
    echo "Namespace $NAMESPACE is not in terminating state. Exiting."
    exit 1
fi

# Step 1: Get the JSON representation of the namespace
echo "Getting JSON representation of namespace $NAMESPACE..."
kubectl get namespace $NAMESPACE -o json > namespace.json

# Step 2: Edit the JSON to set finalizers to an empty array
echo "Editing the JSON to set finalizers to an empty array..."
jq '.spec.finalizers = []' namespace.json > namespace_finalizer_removed.json

# Step 3: Apply the edited JSON to finalize the namespace
echo "Applying the edited JSON to remove finalizers..."
kubectl replace --raw "/api/v1/namespaces/$NAMESPACE/finalize" -f namespace_finalizer_removed.json

# Clean up temporary files
rm namespace.json namespace_finalizer_removed.json

# Validate namespace deletion
echo "Validating namespace $NAMESPACE deletion..."
sleep 5  # Wait for a few seconds to allow deletion to propagate

new_status=$(kubectl get namespace $NAMESPACE -o json | jq -r '.status.phase' 2>/dev/null)
if [ -z "$new_status" ]; then
    echo "Namespace $NAMESPACE has been successfully deleted."
else
    echo "Namespace $NAMESPACE is still in state: $new_status"
    echo "Further investigation may be needed."
fi
```

### README: `README.md`

```markdown
# Namespace Deletion Script

This script is designed to delete Kubernetes namespaces that are stuck in the terminating state by removing their finalizers.

## Usage

To use the script, run the following command:

```sh
./delete_stuck_namespace.sh <namespace-name>
```

Replace `<namespace-name>` with the actual name of the namespace you want to delete.

## Prerequisites

- Ensure you have `kubectl` and `jq` installed on your system.
- You need appropriate permissions to modify namespaces in your Kubernetes cluster.

## Script Details

The script performs the following steps:

1. **Check Current Namespace Status:**
   - Verifies if the namespace is in the terminating state before proceeding.

2. **Get Namespace JSON Representation:**
   - Retrieves the JSON representation of the specified namespace.

3. **Edit JSON to Remove Finalizers:**
   - Modifies the JSON to set the `finalizers` field to an empty array (`[]`).

4. **Apply Edited JSON:**
   - Uses `kubectl replace --raw` to apply the edited JSON and remove the finalizers.

5. **Cleanup:**
   - Deletes the temporary JSON files created during the process.

6. **Validate Namespace Deletion:**
   - Checks if the namespace has been successfully deleted after a brief wait.

## Notes

- The script includes checks to ensure it only attempts to delete namespaces that are in the terminating state.
- If the namespace is not in the terminating state, the script exits without making any changes.
- Ensure you have the necessary permissions and the `kubectl` context is set correctly to interact with your cluster.

## Troubleshooting

- If the namespace is still not deleted after running the script, further investigation may be needed. Ensure that no other resources or finalizers are blocking the deletion.
```

### Additional Considerations

- Ensure that there are no trailing spaces at the end of lines in both the script and the README file.
- Ensure that the files end with a newline character.
- The script should have executable permissions set.

This setup should comply with common pre-commit hooks and provide clear instructions on how to use the script.
