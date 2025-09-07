# StatefulSet

A StatefulSet is a Kubernetes controller designed for stateful applications. Unlike a Deployment, where pods are considered interchangeable and disposable, a StatefulSet gives each pod a stable, unique network identifier and guarantees stable, persistent storage.

It is preferred for applications that require stable identities, such as databases (e.g., MySQL, PostgreSQL) or distributed systems (e.g., Kafka, Cassandra).

TODO: Add example StatefulSet configuration and commands to create and manage it.
