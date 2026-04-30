So our situation was likely:

```text
main.tf
  ↓
Create Kafka connector through REST API ✅
  ↓
Run next Terraform resource/task/provisioner/dependency ❌
  ↓
terraform apply fails overall

```

Terraform may still have created the connector, but because the full apply failed, your state could be one of these:

```text
Connector exists in Kafka Connect
Connector may or may not be cleanly recorded in Terraform state
Later resources failed

```

So next run can behave differently depending on state:

### If connector is in state

Terraform says:

```text
Connector already managed
Now continue with remaining failed resources

```

### If connector is not in state

Terraform says:

```text
I need to create connector

```

But Kafka Connect says:

```text
Connector already exists

```

Then you get duplicate/already-exists errors.

### If connector is in state but tainted

Terraform may say:

```text
Destroy and recreate connector

```

Best cleanup commands:

```bash
terraform state list

```

Then inspect:

```bash
terraform state show <connector_resource_address>

```

If connector exists but is missing from state:

```bash
terraform import <connector_resource_address> <connector_name>

```

Then run:

```bash
terraform plan

```

Main lesson:

> Terraform does not roll back already-created infrastructure when a later step fails. It stops at the failure, and you must reconcile state vs real infrastructure before re-running.
<!--stackedit_data:
eyJoaXN0b3J5IjpbNzIyODk2NjQ2XX0=
-->