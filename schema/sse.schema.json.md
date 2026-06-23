{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/YOUR_USERNAME/YOUR_REPO/schemas/sse.schema.json",
  "title": "Structured State Envelope (SSE)",
  "description": "Typed semantic state representation for Derived State Reconstruction (DSR) in the Serializable Latent State (SLS) framework.",
  "type": "object",
  "additionalProperties": false,

  "required": [
    "envelope_version",
    "session_id",
    "created_at",
    "model_id",
    "task_summary",
    "fidelity_level",
    "task_graph",
    "artifacts",
    "decisions",
    "active_context"
  ],

  "properties": {
    "envelope_version": {
      "type": "string",
      "description": "Version of the SSE schema (e.g., '0.1')."
    },

    "session_id": {
      "type": "string",
      "description": "Unique identifier for the session this envelope was extracted from."
    },

    "parent_session_id": {
      "type": ["string", "null"],
      "description": "Optional ID of the previous session in a session chain."
    },

    "created_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp when this envelope was created."
    },

    "model_id": {
      "type": "string",
      "description": "Identifier of the model used (e.g., 'claude-3.7', 'gpt-5-2026-04')."
    },

    "task_summary": {
      "type": "string",
      "description": "Short human-readable summary of the overall task."
    },

    "fidelity_level": {
      "type": "integer",
      "enum": [3, 4],
      "description": "Fidelity level of this envelope: 3 = structured state, 4 = structured state + critical excerpts."
    },

    "task_graph": {
      "type": "object",
      "required": ["root_task", "subtasks"],
      "properties": {
        "root_task": {
          "type": "string",
          "description": "Primary task being worked on."
        },
        "subtasks": {
          "type": "array",
          "description": "List of subtasks with status and dependencies.",
          "items": {
            "type": "object",
            "required": ["id", "description", "status"],
            "properties": {
              "id": {
                "type": "string",
                "description": "Unique subtask identifier."
              },
              "description": {
                "type": "string",
                "description": "Description of the subtask."
              },
              "status": {
                "type": "string",
                "enum": ["completed", "in_progress", "not_started"],
                "description": "Current status of the subtask."
              },
              "outcome": {
                "type": "string",
                "description": "Outcome or result for completed tasks."
              },
              "current_focus": {
                "type": "string",
                "description": "Current focus for in-progress tasks."
              },
              "depends_on": {
                "type": "array",
                "items": { "type": "string" },
                "description": "IDs of subtasks this one depends on."
              },
              "blocked_by": {
                "type": ["string", "null"],
                "description": "ID of subtask blocking this one, if any."
              }
            }
          }
        }
      }
    },

    "artifacts": {
      "type": "array",
      "description": "Registry of files, documents, or artifacts referenced in the session.",
      "items": {
        "type": "object",
        "required": ["id", "type", "path", "description"],
        "properties": {
          "id": {
            "type": "string",
            "description": "Unique artifact identifier."
          },
          "type": {
            "type": "string",
            "description": "Artifact type (e.g., 'source_file', 'test_file', 'schema', 'document')."
          },
          "path": {
            "type": "string",
            "description": "Logical or physical path to the artifact."
          },
          "description": {
            "type": "string",
            "description": "Human-readable description of the artifact."
          },
          "last_known_state": {
            "type": "string",
            "description": "Summary of the artifact’s current state (e.g., tests passing, size, known issues)."
          },
          "key_structures": {
            "type": "array",
            "items": { "type": "string" },
            "description": "Important structs, classes, schemas, or components in this artifact."
          }
        }
      }
    },

    "decisions": {
      "type": "array",
      "description": "Log of key decisions made during the session.",
      "items": {
        "type": "object",
        "required": ["id", "decision"],
        "properties": {
          "id": {
            "type": "string",
            "description": "Unique decision identifier."
          },
          "decision": {
            "type": "string",
            "description": "The decision that was made."
          },
          "rationale": {
            "type": "string",
            "description": "Reasoning behind the decision."
          },
          "made_at": {
            "type": "string",
            "description": "Session or turn where the decision was made."
          },
          "alternatives_considered": {
            "type": "array",
            "items": { "type": "string" },
            "description": "Alternatives that were considered and rejected."
          }
        }
      }
    },

    "active_context": {
      "type": "object",
      "required": ["current_problem"],
      "properties": {
        "current_problem": {
          "type": "string",
          "description": "The immediate problem being investigated when the session ended."
        },
        "hypotheses": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Current working hypotheses about the problem."
        },
        "last_action": {
          "type": "string",
          "description": "Most recent action taken related to the problem."
        },
        "next_steps": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Planned next steps when the session resumes."
        },
        "constraints": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Constraints that must be respected (performance, architecture, etc.)."
        }
      }
    },

    "working_model": {
      "type": "object",
      "description": "Optional verbatim excerpts for Level 4 envelopes.",
      "properties": {
        "critical_excerpts": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["artifact_id", "region", "content"],
            "properties": {
              "artifact_id": {
                "type": "string",
                "description": "ID of the artifact this excerpt comes from."
              },
              "region": {
                "type": "string",
                "description": "Region description (e.g., 'lines 34–67')."
              },
              "content": {
                "type": "string",
                "description": "Verbatim excerpt of critical code/schema/passage."
              },
              "annotation": {
                "type": "string",
                "description": "Optional explanation of why this excerpt is important."
              }
            }
          }
        }
      }
    }
  }
}
