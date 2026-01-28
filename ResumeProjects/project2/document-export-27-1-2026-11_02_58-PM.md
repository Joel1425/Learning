# A4C Service

_Problem earlier: A server used to be allocated to N devs. The devs used to SSH to that same server. All the tools were on the server, to get the source code to used to do perforce sync on a particular directory._

### **1. Problem Statement**
Design a workspace provisioning service that can create workspaces on demand based on user requirments.

---

### **2. Requirements**
#### Functional Requirements
- User should be able to create a workspace based on a project.
- User should be able to manage(list, delete, etc) their workspaces.
- User should be able to schedule image builds of a project.
#### Non-Functional Requirements
- Highly available (no single point of failure)
- Workspace list latency < 100ms
- Handling image builds under heavy load
---

### **3. Core Entities**
- User
- Workspace
- Image
- BuildSchedule
---

### **4. API routes**
User should be able to create a workspace based on a project.

4.1 `POST /v1/users/{userId}/workspaces` 

`{` 

`      projectName,` 

`      parentName,` 

`      arch,` 

`      ...` 

`}` 

4.2 `GET /v1/users/{userId}/workspaces` -> `[]workspace` 

4.2 `DELETE /v1/users/{userId}/workspaces/{workspaceId}` 

4.3 `POST /v1/users/{userId}/image-schedules` 

`{` 

`imageName,` 

`project,` 

`change,` 

`}` 

---

### **5. High level design**
#1 A4C service gets the list of servers having the image needed for creation and sends the request to the Disk service to get the server with the least diskUsage. Now the A4C service connects with the docker daemon of the server to create the container based on the found image( assigns IP, and a hostname etc : SRE Team handles that part) and returns the hostname to the client.

#2 [DELETION] A4C service connects with the docker daemon of the server to delete the container, thereafter deletes the entry from the database.

#3 Add an entry in the buildSchedule table with the status as [SCHEDULED | ...]. Once any worker (bare metal server) gets free, the imagebuilder daemon polls for any image that needs to be scheduled. The DB gives such an entry and updates the status as IN_PROGRESS (On Transactional basis).



FAQs

1. How does the image builder daemon builds an image as per config?
-> There is a dockerFile template, which is populated based on user config which runs the following things:

-     Install all tools like curl/bash/vim/emacs ... etc
-     Run p4-sync for the given change number
-> Once done, the worker sends the API request on the status to the A4C service which marks it as COMPLETED and sends an email notification to the user/client.

---

### 6. Deep dives
How do you make the system highly available (avoid SPOF)?

-> There are multiple replicas of A4C service and Disk Service, hosted on K8S cluster.



What is DB is down?

-> We have a Primary/replicas architecture, so the writes are propagated using WAL asynchronously to all the replicas.  



What if workspace creation fails in between say,

1. Container create on host
2. setup 
3. ip assign
4. hostname entry
fails at #2?

-> After every successful step the states are updated. Let's say we fail in between, then while retrying(only in case of 5xx error) the stale containers are cleaned up first an then retry happens with a fresh state.



-> What is deletion fails in between?

cleanup daemon check for all the containers present for this host, if not present in host, API call is made to A4C service to clean up those containers from the DB 









