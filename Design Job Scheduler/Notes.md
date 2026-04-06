
### Use cases:
- Slack to send messages to offliine users
- Youtube to send email campaigns to subscribers
- Webcrawlers to crawl websites at specific times

### 1. Requirements:

**Functional:**
1. Immediate execution- no delays, no time wasted
2. Future & recurring execution- support jobs scheduled in future (every monday @ 8am, everyday at 2pm)
3. Observability- error/success rates, latency, status

**Non-functional:**
- 10k jobs per second
- low latency - stuff like bidding
- correctness- no jobs are lost
- availability > consistency- the system must be available even if there's outage

---
### 2. Entities

Two types of entities:
- **job (or task):** blueprint (what needs to be done)- "send a weekly summary email" but doesnt specific when or for whom
- **job_run (attempt):** specific instance - time, owner, recipient, etc- "send a weekly summary email to Bob", "send a weekly summary email to David." Multiple instances for one job.

---
### 3. High level design

#### Functional Req 1: Immediate execution, no delays.

Remember: **job = blueprint**, **job_run = attempts**- one job (blueprint) could have multiple job_runs (attempts)
	![[Pasted image 20260217022521.png  | 1000]]
Here, we're using Relational db --> ACID (like MySQL).


**Lease:**
- Prevents double execution (don't want two workers to: "charge same card twice" or "send same email twice")
- If a worker crashes (no heartbeat), new worker = new lease = new job.
- Fields like "claimed_by" or "lease_expires_at" indicate the lease and owner. 

#### Functional req. 2: Future & recurring execution

Now just add a scheduling layer on the top of the first functional requirement that scans "active schedules" with the "next_fire_at" in a window. 

Each schedule, the "next_fire_at" advances. 
![[Pasted image 20260217031303.png | 1000]]

#### Common strategies to publish a job
1. Publish as soon as the job becomes available
2. Push the job to the queue and ask it to hold (due time - current time)
3. Write to outbox (a db table), relay (a process) scans it, then publishes
	- easier to delete (delete db row), more control here

#### Functional Req 3
Create a new service called "Health Check" then connect every other component to it.
![[Pasted image 20260217032113.png | 1000]]

---

#### Scalability (non-functional reqs.)
1. **Workers polling db too much "anything for me"**
   Fix: The scheduler pushes the jobs to a priority queue. And workers consume it directly from there.

2. **Scaling workers**
   Requirement = 10k jobs/sec. ASK THIS, we're assuming:
   each worker = 10 jobs/sec --> we need 1000 workers in parallel. Add more workers

3. Set index on "next_fire_at"
   Imagine you have jobs table. It's monday 8am - the scheduler has to go thru each line to see where **next_fire_at <= current_time**. Inefficient.

	**Jobs table (blueprints):**

| job_id | type  | payload                  | next_fire_at   |
| ------ | ----- | ------------------------ | -------------- |
| 1      | slack | "send standup reminder"  | Mon 8:00 AM ✅  |
| 2      | email | "weekly summary to Bob"  | Wed 9:00 AM    |
| 3      | crawl | "scrape pricing page"    | Mon 8:00 AM ✅  |
| 4      | slack | "send EOD report"        | Mon 5:00 PM    |
| 5      | email | "monthly newsletter"     | Mar 1 10:00 AM |
| 6      | crawl | "scrape competitor blog" | Mon 8:00 AM ✅  |

Instead we can set a db index on next_fire_at (which is basically sorting). And push each job to the queue.

**Index (sorted shortcut):**

| next_fire_at   | job_id |
|----------------|--------|
| Mon 8:00 AM    | 1      |
| Mon 8:00 AM    | 3      |
| Mon 8:00 AM    | 6      |
| Mon 5:00 PM    | 4      |
| Wed 9:00 AM    | 2      |
| Mar 1 10:00 AM | 5      |
