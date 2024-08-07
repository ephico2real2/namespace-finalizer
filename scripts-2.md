Using `jq` to beautify the JSON output from the `kubectl replace --raw` command. Here is the updated script that includes this improvement:

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

# Function to force delete all resources in the namespace
force_delete_resources() {
    local ns=$1
    while true; do
        echo "Force deleting all resources in namespace $ns..."
        kubectl delete all --all -n $ns --force --grace-period=0
        if [ $? -eq 0 ]; then
            echo "All resources in namespace $ns have been force deleted."
            break
        else
            echo "Retrying force delete of all resources in namespace $ns..."
            sleep 2
        fi
    done
}

# Check if namespace parameter is provided
if [ -z "$1" ]; then
    usage
fi

NAMESPACE=$1

# Parameterized filenames
NAMESPACE_JSON="${NAMESPACE}.json"
NAMESPACE_FINALIZER_REMOVED_JSON="${NAMESPACE}_finalizer_removed.json"

# Validate current namespace status
echo "Checking current status of namespace $NAMESPACE..."
current_status=$(check_namespace_status $NAMESPACE)
echo "Current status: $current_status"

if [ "$current_status" != "Terminating" ]; then
    echo "Namespace $NAMESPACE is not in terminating state. Exiting."
    exit 1
fi

# Force delete all resources in the namespace
force_delete_resources $NAMESPACE

# Step 1: Get the JSON representation of the namespace
echo "Getting JSON representation of namespace $NAMESPACE..."
kubectl get namespace $NAMESPACE -o json > $NAMESPACE_JSON

# Step 2: Edit the JSON to set finalizers to an empty array
echo "Editing the JSON to set finalizers to an empty array..."
jq '.spec.finalizers = []' $NAMESPACE_JSON > $NAMESPACE_FINALIZER_REMOVED_JSON

# Step 3: Apply the edited JSON to finalize the namespace
echo "Applying the edited JSON to remove finalizers..."
kubectl replace --raw "/api/v1/namespaces/$NAMESPACE/finalize" -f $NAMESPACE_FINALIZER_REMOVED_JSON | jq .

# Clean up temporary files
rm $NAMESPACE_JSON $NAMESPACE_FINALIZER_REMOVED_JSON

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

This script is designed to delete Kubernetes namespaces that are stuck in the terminating state by removing their finalizers. It also includes a cleanup step to force delete all resources in the namespace before attempting to remove the finalizers.

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

2. **Force Delete All Resources in the Namespace:**
   - Deletes all resources within the namespace using the `--force` and `--grace-period=0` options.
   - Retries the deletion process until it completes successfully.

3. **Get Namespace JSON Representation:**
   - Retrieves the JSON representation of the specified namespace and saves it to a file named `<namespace-name>.json`.

4. **Edit JSON to Remove Finalizers:**
   - Modifies the JSON to set the `finalizers` field to an empty array (`[]`) and saves it to a file named `<namespace-name>_finalizer_removed.json`.

5. **Apply Edited JSON:**
   - Uses `kubectl replace --raw` to apply the edited JSON and remove the finalizers.
   - Beautifies the output using `jq`.

6. **Cleanup:**
   - Deletes the temporary JSON files created during the process.

7. **Validate Namespace Deletion:**
   - Checks if the namespace has been successfully deleted after a brief wait.

## Notes

- The script includes checks to ensure it only attempts to delete namespaces that are in the terminating state.
- If the namespace is not in the terminating state, the script exits without making any changes.
- Ensure you have the necessary permissions and the `kubectl` context is set correctly to interact with your cluster.

## Troubleshooting

- If the namespace is still not deleted after running the script, further investigation may be needed. Ensure that no other resources or finalizers are blocking the deletion.
```

This updated script uses `jq` to beautify the JSON output from the `kubectl replace --raw` command, making the output more readable. The README has been updated to reflect these changes.
