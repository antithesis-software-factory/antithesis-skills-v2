# Mapping what the workload must reach

## 1. List the system's whole surface

Read the code that receives requests (API definitions and their handlers), not just the docs. List:

- **Every operation** a client can send.
- **Every input that sends a handler down a different branch:** fields, flags, headers, modes.
- **Background work** the system does on its own.
- **Configuration that changes behavior.**
- **The real clients**, and the request shapes they send. Prefer those over a generic use of the API. Note also what the client library does between requests: retries, sessions, transactions, cached routing.

Prioritize what real clients depend on and the code with the most logic. Cover those deeply before adding rarely used operations.

## 2. Decide coverage, item by item

For every item, name the action or setup that exercises it, or decide it's out of scope and why.

## 3. Design the deployment to reach it

- Run multiple nodes or replicas if the system supports them.
- Make the system's process the container's main process (`exec` it) so it receives signals, and give its containers `restart: on-failure` rather than restarting it in a shell loop.
