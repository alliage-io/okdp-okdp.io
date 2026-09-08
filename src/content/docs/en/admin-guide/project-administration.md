---
title: Project administration
description: Managing projects and monitoring their ressources as well as monitoring instances in the OKDP control plane.
---

## Projects

Projects are Kubernetes [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) containing the label `okdp.io/project` isolating the application services deployed within from the ones in the other projects. It is not possible to bind two services in different namespaces. Trino, for instance, cannot communicate with a Hive Metastore from another project.

The name of the project is the name of the namespace.

Here is an example of a project in YAML format:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  labels:
    okdp.io/project: demo
  annotations:
    okdp.io/description: Demonstration project, full data chain
```

To create a new project with the user interface, click on the top left button indicating your current project and then on `all projects`. Here, all your projects are listed, and you can create a new one by clicking on `+ Create project`. A window opens and asks for the project name, optionally a description and choose a font color, which you can change later, and click on `CREATE` to initialize the project.

![Project](../../assets/project.png)

Switch between the projects by clicking on the top-left tab indicating your current project name and choosing between all created projects.

To permanently delete a project, go to `Project Panel` -> `Settings` and in the section `Delete this project` click on `Delete`. The permanent deletion is executed after you confirm it.

Note: deleting a project corresponds to the deletion of the namespace. All instances and objects within the namespace will be deleted with it and cannot be retrieved.

## Service Catalog customization

The OKDP control plane enables you to add, remove, and edit services in the service catalog. By clicking on the top right on your username, then on `Administration` -> `Service Catalog`. You are then redirected to the following page with all current services listed in your catalog:

![Service catalog](../../assets/service-catalog.png)

On each service line, the three dots at the right enable you to edit or delete a service. To add a service, click on the top right `+ Add service` button. You then have to give a service name, a version for the service, and the kubocd package repository where it is located. The other fields are optional, and by clicking on `Add service`, it then appears in the list.

## Monitoring

An important task is to monitor the functioning of the instances as well as to check the resource consumption.

### Resource consumption

[Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server) must be installed in order to have the resource consumption displayed in the user interface.

CPU and memory are displayed in the OKDP control plane for each project and each instance. For each instance a colored bar shows the consumption relative to the maximum capacity.

### Operationability and debugging

The status of the instances, pods, connections, secret stores, and external secrets reveals if there is a potential problem. If it is the case, look at the details of these objects by clicking on them or on the little eye icon when it concerns an instance. The details section of an instance at the top displays error and warning messages for your information. For deeper investigation, look at the pod's logs.

![Pod logs](../../assets/pod-logs.png)

Logs can also be downloaded by clicking on the download icon on the top right above the log display.

However, these debugging tools in the user interface are not always sufficient, and you may have to switch to manual debugging in a terminal with command line instructions.
