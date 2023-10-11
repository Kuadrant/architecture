# Definition of Common Terms

## Infrastructure

- <a name="hub"/>**Hub**:
    - **Definition**: A kubernetes cluster that is running OCM, typically connected to one or more [spokes](#spoke).
    - Common alternatives to avoid:
        - Control plane
        - Control cluster
- <a name="spoke"/>**Spoke**:
    - **Definition**: A Kubernetes cluster connected to a [hub](#hub) where resources can be placed via a manifestwork.
    - Common alternatives to avoid:
        - Workload cluster
        - Cluster
        - Gateway cluster
        - Data plane
        - Managed cluster
- <a name="workload"/>**Workload**:
    - **Definition**: A set of resources deployed to one or more [spokes](#spoke) - typically accessed via a [gateway](#gateway) - and designed to complete a task, e.g. an API or web-page.
    - Common Alternatives to avoid:
        - Application
        - API
        - deployment
        - backend
- <a name="gateway"/>**Gateway**:
    - **Definition**:
        - On a [spoke](#spoke): A resource designed to facilitate ingress to a [workload](#workload)
        - On a [hub](#hub): A resource which follows specific mutation and replication rules to be replicated into [spokes](spokes).

## Personas
- <a name="gateway-admin"/>**Gateway Admin**
    - **Definition**: A user who is primarily concerned with configuring gateways across one or more [spokes](#spoke)
    - Common alternatives to avoid:
        - Sysadmin
        - Admin
        - SRE
        - cluster-admin
- <a name="cluster-admin" />**Cluster admin**:
    - **Definition**: A user who is primarily concerned with the configuration of a particular [workload](#workload) connected to a [gateway](#gateway) on a [spoke](#spoke).
    - Common alternatives to avoid:
        - Workload admin
- <a name="developer"/>**Developer**:
    - **Definition**: A user who is primarily concerned with the code running in a [workload](#workload).
    - Common alternatives to avoid:
        - N/A
- <a name="end-user"/>**End User**:
  - **Definition**: A user who is consuming a [workload](#workload) and usually accessing it via a [gateway](#gateway).
  - Common alternatives to avoid:
    - user
    - customer