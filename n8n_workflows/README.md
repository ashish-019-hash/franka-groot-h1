# n8n Workflows Guide

This folder contains three n8n workflows that together form the brain of the multi-robot warehouse demo. Below is a simple explanation of every node in each workflow.

---

## 1. Demo: Robots Workflow (`Demo_Robots_Workflow_(3).json`)

This is the **main command center workflow**. It listens for messages from robots, uses an AI model to decide what each robot should do, and sends commands back to them.

### Nodes (in order of execution)

#### Entry Points (how the workflow gets triggered)

| Node | Type | What It Does |
|------|------|-------------|
| **comman_center_listener** | MQTT Trigger | Listens on the `command_center/topic` MQTT topic. When H1 sends a "stopped" message, this node picks it up and starts the workflow. |
| **h1_2_status_listener** | MQTT Trigger | Listens on the `h1_2/status` MQTT topic. When H1_2 spots the object, this node picks up that message and starts a separate path in the workflow. |
| **When chat message received** | Chat Trigger | Allows you to manually test the command center by typing messages into the n8n chat window. Useful for testing without running the simulation. |

#### Decision Nodes (checking what happened)

| Node | Type | What It Does |
|------|------|-------------|
| **Check CC Stopped** | If | Checks if the message from `comman_center_listener` contains the word "stopped." Both the yes and no paths go to the Robot Command Center, so the AI always gets to evaluate the situation. |
| **Check H1_2 Spotted** | If | Checks if the message from `h1_2_status_listener` says "object_spotted." If yes, it sends the message to the Robot Command Center. If no, nothing happens. |

#### AI Brain (where the thinking happens)

| Node | Type | What It Does |
|------|------|-------------|
| **Robot Command Center** | LLM Chain | This is the heart of the workflow. It receives the robot's status message and sends it to an AI model (NEXUS) to decide what each robot should do. The AI reads the situation and outputs a JSON response listing which robots need which commands. |
| **Ollama Chat Model** | Ollama LLM | The AI model that powers the Robot Command Center. Uses `qwen2.5:7b` running locally through Ollama. This is connected to the Robot Command Center as its language model. |
| **Structured Output Parser** | Output Parser | Makes sure the AI's response is properly formatted JSON. It expects the output to follow this structure: `{"assign_agents": [{"agent": "h1", "command": "stay"}, ...]}`. If the AI gives a bad response, this catches it. |

#### Processing the Decision

| Node | Type | What It Does |
|------|------|-------------|
| **Command Center Result** | Code | Sits right after the Robot Command Center. This node reads the AI's decision and enriches it. If the AI forgot to include H1_2, this node adds it. It also creates a readable summary of the decision (e.g., "Both robots spotted the object. H1 stops, H1_2 continues patrol."). **Click this node after execution to see the full decision.** |
| **Split Out** | Split Out | The AI returns one list with all robot assignments. This node splits that list into individual items — one per robot — so each robot's command can be handled separately. |
| **Switch** | Switch | Routes each robot's command to the right place. It checks the `agent` field and sends: `h1_2` to output 0, `h1` to output 1, `manipulator` to output 2. The h1_2 check uses exact match to prevent confusion with h1. |

#### Robot Command Nodes (sending commands to robots)

| Node | Type | What It Does |
|------|------|-------------|
| **H1_2 Control** | MQTT Publish | Sends the command for H1_2 to the `h1_2/control` MQTT topic. The message includes the agent name and the command decided by the AI (e.g., `{"agent":"h1_2","command":"continue_patrol"}`). |
| **Call My Sub-workflow** | Execute Workflow | When H1 gets a command, this node calls the **h1 workflow** to handle it. The h1 workflow has its own AI agent that processes the command. |
| **H1_Leo** | MQTT Publish | Sends a message to the `h1_2/control` topic. This is an additional control point for H1_2. |
| **franka_L1_Manipulator** | Execute Workflow | When the manipulator gets a command, this calls the **mobile-manipulator workflow** which handles the Franka arm pick-and-place. |
| **Manipulator Arm** | Execute Workflow | A second connection to the mobile-manipulator workflow (same workflow as franka_L1_Manipulator). |

#### End Nodes

| Node | Type | What It Does |
|------|------|-------------|
| **Merge** | Merge | Collects the results from H1's sub-workflow and the Franka manipulator after they finish their tasks. |
| **No Operation, do nothing** | NoOp | A simple end point after the Merge. Does nothing — just marks the end of the flow. |

### Flow Diagram

```
Path 1 (H1 reports "stopped"):
  comman_center_listener → Check CC Stopped → Robot Command Center (AI)

Path 2 (H1_2 spots object):
  h1_2_status_listener → Check H1_2 Spotted → Robot Command Center (AI)

Both paths continue the same way:
  Robot Command Center → Command Center Result → Split Out → Switch
      |-- h1_2       → H1_2 Control (sends MQTT message)
      |-- h1         → Call My Sub-workflow (runs h1 workflow)
      |-- manipulator → franka_L1_Manipulator / Manipulator Arm (runs Franka workflow)
```

---

## 2. H1 Workflow (`h1_(1).json`)

This is **H1's own brain**. It handles camera images from H1's head-mounted camera, analyzes them using AI vision, and reports what it sees to the command center.

### Nodes

#### Entry Points

| Node | Type | What It Does |
|------|------|-------------|
| **camera_feed** | MQTT Trigger | Listens on `/spot/camera/image_stream` for camera images coming from H1's head camera in the simulation. Every time a new image arrives, this node starts the flow. |
| **When Executed by Another Workflow** | Workflow Trigger | Allows this workflow to be called from the Demo Robots Workflow (via "Call My Sub-workflow"). When the command center sends H1 a command, it comes through here. |

#### Image Processing

| Node | Type | What It Does |
|------|------|-------------|
| **Convert to File** | Convert to File | Takes the raw image data from the MQTT message and converts it into a file format that the AI vision model can understand. |
| **Analyze image** | Ollama (Vision) | Uses an AI vision model (`qwen3-vl:8b`) to look at the camera image and determine if there's an object on the warehouse floor. If an object is found, it returns a message saying "H1 robot stopped. Object found on the floor." If nothing is found, it returns empty. |

#### H1's AI Agent

| Node | Type | What It Does |
|------|------|-------------|
| **H1 Agent** | AI Agent | H1's own AI brain. It receives the image analysis result and decides what to do. It can use MQTT tools to send messages to the command center and report its status. Uses the `gpt-oss:20b` model. |
| **Ollama Chat Model1** | Ollama LLM | The AI model powering the H1 Agent. Uses `gpt-oss:20b` through Ollama. |

#### H1's Communication Tools (used by H1 Agent)

| Node | Type | What It Does |
|------|------|-------------|
| **h1_topic** | MQTT Tool | Publishes commands to the `h1/hello` topic. The H1 Agent uses this tool to send control messages for itself. |
| **comman_center_topic** | MQTT Tool | Publishes messages to `command_center/topic`. The H1 Agent uses this to report findings (like "object detected on floor") to the command center. |
| **h1_status_topic** | MQTT Tool | Publishes status updates to `h1/status`. The H1 Agent uses this to report its current state (stopped, walking, etc.). |

#### End Node

| Node | Type | What It Does |
|------|------|-------------|
| **No Operation, do nothing** | NoOp | End point after the H1 Agent finishes processing. |

### Flow Diagram

```
Camera path:
  camera_feed → Convert to File → Analyze image → H1 Agent → No Operation

Command path (called from Demo Workflow):
  When Executed by Another Workflow → H1 Agent → No Operation

H1 Agent can use these tools:
  - h1_topic (send to h1/hello)
  - comman_center_topic (report to command center)
  - h1_status_topic (update h1/status)
```

---

## 3. Mobile Manipulator Workflow (`mobile-manipulator (3).json`)

This is the **Franka robotic arm's brain**. It receives pick-and-place commands from the command center and triggers the arm in the simulation.

### Nodes

#### Entry Point

| Node | Type | What It Does |
|------|------|-------------|
| **When Executed by Another Workflow** | Workflow Trigger | This workflow is called from the Demo Robots Workflow when the Switch routes a manipulator command. The pick-and-place command comes through here. |

#### Franka's AI Agent

| Node | Type | What It Does |
|------|------|-------------|
| **Mobile Manipu- -lator Agent** | AI Agent | The Franka arm's AI brain. It receives the pick-and-place command, triggers the arm in the simulation by publishing to `franka/control`, and reports progress to the command center. Uses `qwen2.5:7b` model. |
| **Ollama Chat Model2** | Ollama LLM | The AI model powering the Manipulator Agent. Uses `qwen2.5:7b` through Ollama. |

#### Franka's Communication Tools (used by Manipulator Agent)

| Node | Type | What It Does |
|------|------|-------------|
| **manipulator_topic** | MQTT Tool | Publishes to `franka/control`. **This is the most important tool** — publishing here is what actually triggers the Franka arm to move in the Isaac Sim simulation. Without this, the arm won't do anything. |
| **comman_center_topic1** | MQTT Tool | Publishes to `command_center/topic`. The Manipulator Agent uses this to tell the command center "Franka arm triggered, pick-and-place started." |
| **franka_status_topic** | MQTT Tool | Publishes to `franka/status`. Reports the arm's status (triggered, completed, etc.) so the command center knows when the task is done. |

#### End Node

| Node | Type | What It Does |
|------|------|-------------|
| **No Operation, do nothing** | NoOp | End point after the Manipulator Agent finishes. |

### Flow Diagram

```
When Executed by Another Workflow → Mobile Manipulator Agent → No Operation

Manipulator Agent can use these tools:
  - manipulator_topic (trigger Franka arm via franka/control)
  - comman_center_topic1 (report to command center)
  - franka_status_topic (update franka/status)
```

---

## How All Three Workflows Connect

```
                        Demo: Robots Workflow (Command Center)
                       /              |                \
                      /               |                 \
            Call My Sub-workflow   H1_2 Control    franka_L1_Manipulator
                    |            (MQTT message)          |
                    v                                    v
             h1 Workflow                      mobile-manipulator Workflow
          (camera + analysis)                 (triggers Franka arm)
```

- The **Demo Robots Workflow** is the boss. It receives messages from robots, thinks about what to do, and delegates tasks.
- The **h1 Workflow** is called when H1 needs to process a command. It also independently analyzes camera images.
- The **mobile-manipulator Workflow** is called when the Franka arm needs to pick up an object.
- **H1_2** doesn't have its own workflow. It gets commands directly via MQTT messages from the Demo Workflow.

## MQTT Topics Summary

| Topic | Purpose | Who Publishes | Who Listens |
|-------|---------|--------------|-------------|
| `command_center/topic` | Main command center inbox | H1, H1_2, Manipulator | Demo Workflow |
| `h1/hello` | H1 control commands | H1 Agent | Isaac Sim |
| `h1/status` | H1 status updates | H1 Agent, Isaac Sim | Command Center |
| `h1_2/status` | H1_2 object detection | Isaac Sim | Demo Workflow |
| `h1_2/control` | H1_2 commands from CC | Demo Workflow | (informational) |
| `franka/control` | Trigger Franka arm | Manipulator Agent | Isaac Sim |
| `franka/status` | Franka task status | Manipulator Agent | Command Center |
| `/spot/camera/image_stream` | H1 camera images | Isaac Sim | h1 Workflow |
