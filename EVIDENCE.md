# FestPass Jenkins assignment: evidence

**Student (Git author):** Prakhar Shashank Srivastava  ·  **class.code:** 001  ·  **challenge.code:** 00605
**Job:** `festpass-ci` (Pipeline script from SCM) · **Repo:** `git://127.0.0.1:9418/festpass-java` · **Branch:** `*/main` · **Script path:** `Jenkinsfile` · **Agent:** `student-laptop` (built-in node executors = 0) · Jenkins 2.580.1 LTS on my laptop

## Build summary

| Build | Cause | Commit | Result | Tests | JAR | Evidence |
|---|---|---|---|---|---|---|
| #1 | Started by user (bootstrap run, loads the trigger) | `d7dcc62` Configure my Jenkins assignment | FAILURE | 28 run, 3 failed | not published (Build JAR / Smoke test / Save artifact skipped) | [CSV](evidence/build-1-manual-red/submission-evidence.csv) |
| #2 | **Started by an SCM change** | `427000e` Personalize my help introduction | FAILURE (automatic red) | 28 run, 3 failed | not published | [CSV](evidence/build-2-auto-red/submission-evidence.csv) |
| #3 | **Started by an SCM change** | `d58ce0c` Fix cancel so it removes the booking and frees the seat | **SUCCESS** (automatic green) | **28 run, 0 failed** | published, SHA-256 `d3628c18…3fd346` | [CSV](evidence/build-3-auto-green/submission-evidence.csv), [JAR](evidence/build-3-auto-green/festpass.jar) |

Jenkins' checked-out revision matched `git log -1 --oneline` for each build. Builds #2 and #3 ran from polling alone; I didn't click Build Now.

## Trigger (the only Jenkinsfile change)

```groovy
triggers {
    pollSCM('* * * * *')
}
```

This was committed with `submission.properties` in `d7dcc62`. Polling checks Git once a minute and only builds when there's a new commit, unlike *Build periodically*.

## Diagnosis

The failing tests in builds #1 and #2 were:

- `cancellationFreesSeat`: *"SOLD OUT. Even the developer needs a ticket."*
- `cancellationsSurviveReload`: *"Cancellation was lost"*
- `cliWorkflowPersistsAcrossCommands`: *"List does not reflect persisted cancellation"*

**The broken business rule:** cancelling a booking must remove that student from the event and free the seat. In `BookingService.cancel()`, the code checked that the booking existed, then returned without removing it. The cancelled student kept the seat, so the next booking for a full event was rejected as SOLD OUT, and the saved data still listed the cancelled booking.

## Fix (one line, tests unchanged)

```diff
         if (!event.students.contains(studentId)) {
             throw new IllegalArgumentException("No booking for student: " + studentId);
         }
+        event.students.remove(studentId);
     }
```

Before committing, I checked locally with `java ci/Build.java verify` (28 tests, 0 failures, JAR built) and `java ci/SmokeTest.java` (7 real-JAR commands pass).

## Final JAR (build #3)

`java -jar festpass.jar help` output:

```
FestPass - campus event registration (CI by Prakhar Shashank Srivastava)
Usage: java -jar build/festpass.jar [--data FILE] COMMAND
  ...
```

A short run: 2-seat event, book S1 and S2, then S3 is rejected with `SOLD OUT` (exit 2). Cancelling S1 frees the seat and S3 is accepted. The roster is S2, S3.

Smoke test (archived by Jenkins): `PASS: seven real-JAR commands; persistence, sold-out rejection, cancellation, and seat reuse verified.`
