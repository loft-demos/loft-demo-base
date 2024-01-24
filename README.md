# loft-demo-base
Base demo objects and GitHub setup for Loft demos.

## Demo Examples

### vCluster.Pro Projects

Projects are the highest level organizational unit in vCluster.Pro. In the simplest form, projects can be thought of as containers for virtual resources (spaces and virtual clusters), however, they also play an important role in enforcing role based access and quotas within vCluster.Pro.

- [Project CRD example](loft/projects.yaml)

### Virtual Cluster Templates

Virtual cluster templates are, as the name implies, templates that you can create which can then be applied when creating a virtual cluster. These templates can be used to configure any virtual cluster settings that you would configure on an individual virtual cluster.

- [VirtualClusterTemplate CRD example](loft/vcluster-templates.yaml)


