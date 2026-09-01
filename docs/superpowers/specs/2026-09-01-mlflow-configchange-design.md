# MLflow ConfigChange Trigger Design

## Goal

Ensure a newly created MLflow BuildConfig starts the initial image build without a manual `oc start-build` command.

## Design

Add one `ConfigChange` entry to `spec.triggers` in `manifests/mlflow/buildconfig.yaml`. Keep the existing inline Dockerfile, output ImageStreamTag, and deployment image reference unchanged.

The trigger addresses clean installations. Applying it to the existing BuildConfig does not require forcing another build because `mlflow-1` has already been created.

## Verification

- Validate the manifest with the OpenShift API server.
- Confirm the live BuildConfig contains the `ConfigChange` trigger after Argo CD syncs.
- Confirm Argo CD remains synced to the published revision.
