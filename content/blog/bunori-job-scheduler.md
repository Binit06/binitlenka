---
title: Job Scheduler
description: Job Scheduling, Automated backoffs, Resume after Crash and how Bunori handles the downloading while trying to be minimal in nature
date: 2026-11-23
tags:
  - kotlin
  - android
---
## A minimal Job Scheduler that attains the function of a download manager

Most readers out there give you an option to download chapters from a novel, but many of
them don't work properly (some spin forever, and when you try to read offline, the files aren't even there). Downloading can be improved substantially.

For me it is not just a chapter that I need to download, I need to download entire books based on various parameters that a simple `touch -> click -> download` one chapter does not actually solve, this is where a download manager comes into the picture.

The idea is to break the entire thing to be downloaded into single `tasks` entities and then handle the process over them but this model breaks apart when we want to put into place things like `Grouped Resume`, `Grouped Pauses`, `Grouped Cancels`.

## Batch & Task Model

Rather than having just a `task` I moved to structure where task is just a child of a `batch` any operation directed towards a group of tasks are to be passed through a **Batch**  this model gives a lot more freedom for both the end-users and me myself to implement features that I can mostly target in these conditions.
- a special feature for tasks that I want to achieve can now easily be achieved by adding some code just to target the tasks and this will not flow through the batch
- a feature that needs to be handled as a group or on a group of tasks on different batches can just be passed through batches `easy development`.

```kotlin
data class BatchEntity(
    @PrimaryKey
    val id: String,
    val name: String,
    val novelUrl: String,
    val type: JobType,
    val priority: Int = 0,
    val status: JobStatus = JobStatus.PENDING,
    val error: String? = null,
    val metadata: String? = null,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis(),
    val completedAt: Long? = null
)
```

```kotlin
data class TaskEntity(
    @PrimaryKey
    val id: String,
    val batchId: String,
    val name: String,
    val url: String?,
    val novelUrl: String,
    val type: JobType,
    val priority: Int = 0,
    val status: JobStatus = JobStatus.PENDING,
    val attemptCount: Int = 0,
    val maxAttempts: Int = 3,
    val error: String? = null,
    val metadata: String? = null,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis(),
    val completedAt: Long? = null
)
```

### Why metadata in Task Entity?

well tasks are of various types each one of them has its own handler that look for specific metadata for something that they wish to do.

![[assets/bunori-job-scheduler/handler.png|296]]

these handlers require special configs for instance **ArtifactHandler** requires `start` and `end` to determine the chapters to export and all that

## Job Scheduler

Now that the baseline has been set up it would be easier to mention the models and how they are used as the context of how Scheduler calls and links things around the entire application.

A Scheduler service acts as the core of the entire infrastructure to handle core components such as `Notification` `Handler Calls` `Task Termination` and various other core features.

This service is started at the start of the application **for handling specific functions such as resume from crash and auto task take off if the application was killed** 

```kotlin
lifecycleScope.launch {
	SchedulerService.startService(this@MainActivity)
	
	// secondary activity to check for app updates
	try {
		UpdateRepository.getInstance(this@MainActivity).checkForUpdates()
	} catch (e: Exception) {
		android.util.Log.e("MainActivity", "Failed to check for updates on startup: ${e.message}", e)
	}
}
```

as already mentioned every download or export task has its own handler that can call on the database these are registered on the scheduler when it is started

```kotlin
val novelRepository = NovelRepository.getInstance(this)
val chapterRepository = ChapterRepository.getInstance(this)
val preferenceRepository = PreferenceRepository.getInstance(this)
val storageRepository = StorageRepositoryImpl.getInstance(this)
val artifactRepository = ArtifactRepository.getInstance(this)
val downloadRepository = DownloadRepositoryImpl.getInstance(this)

val epubGenerator = EpubGenerator(...)
val pdfGenerator = PdfGenerator(...)
val generatorFactory = ArtifactGeneratorFactory(...)

val registry = JobHandlerRegistry()
val crawlerFactory = CrawlerFactory

registry.register(JobType.CHAPTER, ChapterHandler(...))
registry.register(JobType.NOVEL_METADATA, NovelMetadataHandler(...))
registry.register(JobType.ARTIFACT, ArtifactHandler(...))
```


## How this architecture failed in the worst way possible after the first launch of the alpha version

This entire architecture at the end of day this had a major roadblock that is the **race condition** (this always trips me at the best moment possible)

How things operate at this level is take an assumption the `number of concurrent` tasks that can run at a moment is let's say 3 and a batch is running while there are three tasks that are taking the `total available capacity` of the source and at this exact moment the user presses cancel on the entire batch.

What my dumb ass did on the current architecture was set up a loop to go through all the task in the batch and then cancel the task that are being found under the same. while this works under certain conditions perfectly fine, but in my case I had written a block of code that does this

```psudocode
private suspend fun schedule() {
	check_cancel_task();
	if (canceled) skip;
	else do the task
}
```

*Since I don't have the actual code snippet as of now since I fixed this issue a long time ago it was mostly something like that used to happen.*

now since the task had already started and it was cancelled mid operation there were only two possible states a task could achieve from here on out
 - Completed (this would handle this gracefully and everything works normally)
 - Failed

The problems arises when the task failed and since its a download manager of course there was a retry system in place for failure restoration. This used to put the task back in the line after a fixed time.

If this restoration has already started while the task was being set as cancelled then cancellation policy mark it as `CANCELLED` and just the next moment the restoration service that is already ready to put the task back into the queue marks it as `PENDING`. (**RACE CONDITION**)

Now what this means is the scheduler adds a tag of `PENDING` on the task and now this loops again and again since check_cancel_task says that task has not been cancelled it moves normally and then does its job again and again until its retry attempts are not finished.

at the end a running task while ran was never cancelled in reality and made the downloader just show that the tasks under the batch are still doing on (**since the progress is calculated based on the tasks that are happening**)

## What was the fix for this race condition?

Well, the solution was pretty simple rather than checking the task in `check_cancelled()` I just shifted to checking the `batch` since the recovery service does not affect the batch it was quite easily fixed and also just for more security put a special section to verify the batch status before trying to recover on the recovery function.

```kotlin
private suspend fun recoverAbandoned() {
	val now = System.currentTimeMillis()
	val runningTasks = taskDao.getRunningTasks()

	for (task in runningTasks) {
		if (!activeJobs.containsKey(task.id) && leaseMonitor.isExpired(...)) {
			val batch = batchDao.getBatchById(task.batchId)
			when (batch?.status) {
				JobStatus.CANCELLED -> taskDao.updateStatus(..., CANCELLED)
				JobStatus.PAUSED -> taskDao.updateStatus(..., PAUSED)
				JobStatus.BLOCKED -> taskDao.updateStatus(..., BLOCKED)
				else -> taskDao.updateStatus(..., PENDING)
			}
		}
	}
}
```

## Is the scheduler done?

No, there is one major hurdle still standing in the path of scheduling that has to be cleared before we can say, yes this is a scheduler and download system that can be used by anyone without any issues.

The Question arises - 
##### How do you show progress and what is currently being run and updated?

Well you might say its pretty easy just show the status, well there's the problem the status can be shown on the task while the batch in together shows the status of the batch neither does it show how many tasks have till now `completed`, `failed`, `blocked`, `paused` or for instance any other status that can be possible.

Well I just wrote a query that can give the status of a batch completely by looking through the task table and knowing the related queries that are there:

```sql
SELECT 
	b.*,
	COUNT(t.id) AS totalTasks,
	COUNT(CASE WHEN t.status = 'SUCCESS' THEN 1 END) AS completedTasks,
	COUNT(CASE WHEN t.status = 'FAILED' THEN 1 END) AS failedTasks,
	COUNT(CASE WHEN t.status = 'CANCELLED' THEN 1 END) AS cancelledTasks,
	COUNT(CASE WHEN t.status = 'RUNNING' THEN 1 END) AS runningTasks,
	COUNT(CASE WHEN t.status = 'PENDING' THEN 1 END) AS pendingTasks,
	COUNT(CASE WHEN t.status = 'BLOCKED' THEN 1 END) AS blockedTasks,
	COUNT(CASE WHEN t.status = 'PAUSED' THEN 1 END) AS pausedTasks
FROM batches b
LEFT JOIN tasks t ON b.id = t.batchId
WHERE b.id = :id
GROUP BY b.id
```

One might say well this is not that good with progress updating fast and let's assume someone is downloading in batches a lot then the Data Flow would be changing frequently leading to multiple queries and just load on the database which can slow down other operation that user might want to do while the downloads are happening.

Well good point this issue has not yet been completely solved but it has been mitigated enough that is does not become a bottleneck in daily usage, while this is based on certain assumptions
- the current progress does not strictly show the progress in real time there is a delay of approximately 5 seconds before a new update is shown
- most of the tasks in `Tasks` table are indexed so it does not do a full table query each time
- the UI connects to the database using a flow, that removed the multiple database pooling overhead
- Crawlers have been rate limited to prevent IP bans as well to slow the speed of the download (**slowing down things here makes more sense**)

This reduces the issue of showing progress on the batch in the UI but still another problem still resides and that includes how are we going to progress on specific tasks that are based batched with just a single task inside

For instance, an export activity is just a `single` task that is batched under `Export Batch` a single task only has states like `COMPLETED`, `PAUSED`, `FAILED`, `BLOCKED` , `CANCELLED` where this works for batches that have multiple tasks this fails miserably when you put on a single task it can never show progress on how much has been exported and what is the status.

**WELL TIME FOR A HACK**

```kotlin
@Query("UPDATE tasks SET name = :name, updatedAt = :updatedAt WHERE id = :id")
suspend fun updateTaskName(id: String, name: String, updatedAt: Long = System.currentTimeMillis())
```

Just update the name of the task in such cases from the export activity to emit how much of the activity has been completed 😆

The ArtifactHandler throttles the task name update every `250ms` so it does not hammer SQLite.

When a batch has only one task the Download Notification Manager `Regex("""\((\d+)/(\d+)\)""")` pulls out this to get the progress and show the same on the UI

## VIP Priority Lanes

This is a major part that would not have been necessary if the app was a standalone download manager but the download manager is a part of the main application that handles a lot more things than just downloading.

Every queue has the required capacity of the `source` along side a `reservedSpace` this space is kept empty unless the user takes a manual action that is put in this space and handles as fast as possible since it becomes a part of the queue the `Scheduler` immediately takes it on without any delays.

## Final Takes

This is still a project in maintenance and is still being continuously improved most of what I have improved and added has been found after user testing and been promptly fixed while there has been some implementation that can be termed as not up to the mark or weird to some extent, I agree on the same things can still improve a lot can still be worked upon and things will become better as I take on things like `database engineering`, `distributed systems`, `workflow engineering`, and a lot of other things.