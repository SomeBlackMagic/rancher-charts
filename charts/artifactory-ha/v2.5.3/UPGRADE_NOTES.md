# JFrog Artifactory Chart Upgrade Notes
This file describes special upgrade notes needed at specific versions
## Upgrade from 1.X to 2.X (Chart Versions)

* If this is a new deployment or you already use an external database (`postgresql.enabled=false`), these changes **do not affect you!**
* To upgrade from a version prior to 1.x, you first need to upgrade to latest version of 1.x as described in https://github.com/jfrog/charts/blob/master/stable/artifactory-ha/CHANGELOG.md.

## Upgrade from 0.X to 1.X (Chart Versions)
**DOWNTIME IS REQUIRED FOR AN UPGRADE!**
* If this is a new deployment or you already use an external database (`postgresql.enabled=false`), these changes **do not affect you!**
* PostgreSQL sub chart was upgraded to version `6.5.x`. This version is not backward compatible with the old version (`0.9.5`)!
* Note the following **PostgreSQL** Helm chart changes
  * The chart configuration has changed! See [values.yaml](values.yaml) for the new keys used
  * **PostgreSQL** is deployed as a StatefulSet
  * See [PostgreSQL helm chart](https://hub.helm.sh/charts/stable/postgresql) for all available configurations
* Upgrade
  * Due to breaking changes in the **PostgreSQL** Helm chart, a migration of the database is needed from the old to the new database
  * The recommended migration process is the [full system export and import](https://www.jfrog.com/confluence/display/RTF/Importing+and+Exporting)
    * **NOTE:** To save time, export only metadata and configuration (check `Exclude Content` in the `System Import & Export`) since the Artifactory filestore is persisted
    * Upgrade steps:
      1. Block user access to Artifactory (do not shutdown)
        a. Scale down the cluster to primary node only (`node.replicaCount=0`) so the exported db and configuration will be kept on one known node (the primary)
        b. If your Artifactory HA K8s service is set to member nodes only (`service.pool=members`) you will need to access the primary node directly (use `kubectl port-forward`)
      2. Perform `Export System` from the `Admin` -> `Import & Export` -> `System` -> `Export System`
        a. Check `Exclude Content` to save export size (as Artifactory filestore will persist across upgrade)
        b. Choose to save the export on the persisted Artifactory volume (`/var/opt/jfrog/artifactory/`)
        c. Click `Export` (this can take some time)
      3. Run the `helm upgrade` with the new version. Old PostgreSQL will be removed and new one deployed
        a. You must pass explicit "ready for upgrade flag" with `--set databaseUpgradeReady=yes`. Failing to provide this will block the upgrade!
      4. Once ready, open Artifactory UI (you might need to re-enter a valid license). Skip all onboarding wizard steps
        a. **NOTE:** Don't worry you can't see the old config and files. It will all restore with the system import in the next step 
      5. Perform `Import System` from the `Admin` -> `Import & Export` -> `System` -> `Import System`
        a. Browse to where the export was saved Artifactory volume (`/var/opt/jfrog/artifactory/<directory-you-set>`)
        b. Click `Import` (this can take some time)
      6. Restore access to Artifactory
        a. Scale the cluster member nodes back to the original size
    * Artifactory should now be ready to get back to normal operation