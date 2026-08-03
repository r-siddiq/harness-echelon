# Closed-Task History

This root log preserves closed-task history. Each section contains the completed task's full JSON object and only the chronological `progressEntries` whose `taskId` exactly matches that task.

After orchestrator verification, archive the completed task's full JSON object and its exact matching task-specific progress here, then remove those records from `.tasks.json` and recompute `phaseHealth`. Blocked, paused, and all other non-completed work stays active in `.tasks.json`.

## `PHASE-X-Task-Y`

### Task object

```json
{
  "id": "PHASE-X-Task-Y",
  "description": "Granular action item required to achieve the active phase goal",
  "state": "completed",
  "agent": "general-purpose:<name>:<session-id>:phase-b-complete",
  "retryCounter": 0,
  "maxRetryLimit": 3,
  "turnSelfAssessment": {
    "ruleAdherence": "",
    "contextSanitation": "",
    "frictionTrace": ""
  },
  "dependsOn": [],
  "conflictsWith": [],
  "targetFiles": [],
  "successCriteria": [],
  "dispatches": {
    "researchAgent": "Explore | general-purpose",
    "implementAgent": "general-purpose"
  },
  "phaseA_Research": {
    "summary": "",
    "decisions": [],
    "context": {
      "fileStructure": "",
      "apiPattern": "",
      "dataFlow": "",
      "gotchas": []
    }
  },
  "researchBasis": {
    "sourceFiles": [],
    "upstreamAssumptions": [],
    "status": "current"
  },
  "phaseB_ImplementationSpec": {
    "instructions": "",
    "specificFilesToModify": [],
    "verificationChecklist": []
  },
  "blockerLog": {
    "reason": "",
    "researchNeeded": "",
    "escalation": {
      "needsGateReversion": false,
      "affectedPlanPhase": "",
      "recommendedArchitectAction": ""
    }
  }
}
```

### Matching progress entries

```json
[
  {
    "timestamp": "YYYY-MM-DDTHH:MM:SS.ssssssZ",
    "event": "phase_b_implementation_verified_complete",
    "phaseId": "PHASE-X",
    "taskId": "PHASE-X-Task-Y",
    "agent": "",
    "summary": ""
  }
]
```
