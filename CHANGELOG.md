# Changelog

New features, fixed bugs, known defects and other noteworthy changes to each release of the Catena-X Portal helm chart.

## 1.3.0-RC1

### Change

* changed to v1.3.0-RC1 images
* enabled additional config for portal-backend

### Technical Support

* upgraded workflow actions

## 1.2.1

### Bugfix

* changed to versioned image tag for portal-migrations job in portal-backend

## 1.2.0

### Change

* changed to v1.2.0 images
* enabled additional config for portal-backend
* removed PORTAL_FRONTEND_URL environment variable for portal-frontend

### Technical Support

* trg: added repo metafile

## 1.1.0

### Change

* changed to v1.1.0 images
* enabled additional config for portal-backend

## 1.0.0

### Change

* enabled apps and jobs to use config from the environment variables.
* enabled initdb configmap with custom db user creation, removed initdb container.
* set replica count from 1 to 3.
* moved pgAdmin4 in a separate helm chart.

### Feature

* added checklist-worker job to chart.
* enabled option for external database.
* added rollingupdate strategy for apps.
* added option for resource management for apps.
* added livenessProbe and readinessProbe for frontend apps.
* added assign-pod-node: affinity-and-anti-affinity for apps.
* added for nodeSelector and toleration management for apps.

### Technical Support

* added chart test workflow for lint and install.
* added documentation for installation and changelog.

## 0.6.0

### Change

* added product helm chart for portal, combining frontend and backend chart.
* moved repository to eclipse-tractusx.