# Multi-Robot Warehouse Demo

A simulation where robots work together to find and pick up objects in a warehouse. An AI-powered command center coordinates everything.

## What Does This Demo Do?

Imagine a warehouse with three robots:

1. **H1** - The main walking robot. It walks through the warehouse looking for objects on the floor.
2. **H1_2** - A second walking robot. It patrols the warehouse independently.
3. **Manipulator (Franka Arm)** - A robotic arm on wheels. It picks up objects that H1 finds.

### How It Works (Step by Step)

1. The simulation starts and both H1 and H1_2 begin walking toward the object in the warehouse.
2. H1 is closer to the object, so it reaches the object first and stops near it.
3. H1_2 also spots the object from a distance but since H1 is already handling it, H1_2 just keeps walking ahead.
4. When H1 stops, it sends a message to the **Command Center** saying "I've stopped near the object."
5. When H1_2 spots the object, it also sends a message to the Command Center saying "I spotted the object too."
6. The **Command Center** (powered by an AI model) receives these messages, thinks about the situation, and decides:
   - **H1**: Stay where you are (you're closest to the object)
   - **H1_2**: Keep patrolling ahead (you're not needed for pickup)
   - **Manipulator**: Go pick up the object
7. The Franka robotic arm then drives to the object, picks it up, and places it at a designated location.
8. After the pickup is complete, H1 resumes walking.

### What You'll See in the Demo

- Two humanoid robots walking in a warehouse environment
- One robot (H1) stopping near an object on the floor
- The other robot (H1_2) continuing to walk past
- A robotic arm driving up and picking up the object
- The n8n workflow dashboard showing how the command center makes decisions in real time

## Project Structure

```
franka-groot-h1/
|
|-- n8n_complete_Script.py        # Main simulation script (runs in Isaac Sim)
|-- spot_groot_server.py          # AI vision server (detects new objects using camera)
|-- mqtt/
|   |-- mqtt_ros2.py              # Bridges messages between MQTT and ROS2
|
|-- n8n_workflows/
|   |-- Demo_Robots_Workflow_(3).json    # Main command center workflow
|   |-- h1_(1).json                      # H1 robot's own workflow (camera + analysis)
|   |-- mobile-manipulator (3).json      # Franka arm's workflow (pick and place)
```

## Key Components

### 1. Simulation Script (`n8n_complete_Script.py`)

This is the main file that runs inside NVIDIA Isaac Sim. It:
- Creates the warehouse environment with all three robots
- Makes H1 and H1_2 walk forward toward the object
- Uses an AI vision model (GR00T) to detect objects through H1's camera
- Stops H1 when it gets close to the object
- Keeps H1_2 walking (it never stops)
- Runs the Franka arm pick-and-place when triggered
- Sends and receives messages using MQTT (a lightweight messaging system)

### 2. AI Vision Server (`spot_groot_server.py`)

This runs separately and provides the "eyes" for H1:
- Receives camera images from H1's head-mounted camera
- Uses the NVIDIA GR00T N1 model to analyze what H1 sees
- Detects when something new appears in the camera view (novelty detection)
- Tells H1 whether an object is detected or not

### 3. Command Center Workflow (`Demo_Robots_Workflow_(3).json`)

This is an n8n workflow that acts as the brain of the operation:
- **Listens** for status updates from robots via MQTT messages
- **Checks** what happened (did H1 stop? did H1_2 spot the object?)
- **Thinks** using an AI model (NEXUS) to decide what each robot should do
- **Shows the decision** in the "Command Center Result" node so viewers can see what was decided
- **Routes commands** to each robot:
  - H1 gets told to stay
  - H1_2 gets told to continue patrol
  - Manipulator gets told to pick and place

### 4. H1 Workflow (`h1_(1).json`)

H1's own workflow:
- Receives camera images from the simulation
- Analyzes images using an AI vision model to detect objects on the floor
- Reports findings to the command center

### 5. Mobile Manipulator Workflow (`mobile-manipulator (3).json`)

The Franka arm's workflow:
- Receives pick-and-place commands from the command center
- Triggers the Franka arm in the simulation
- Reports back when the task is done

## How the Robots Communicate

All robots communicate through **MQTT** (a simple messaging system, like a group chat):

| Topic | Who sends | Who listens | What it says |
|-------|-----------|-------------|--------------|
| `command_center/topic` | H1, H1_2 | Command Center | "I stopped" or "I spotted the object" |
| `h1/status` | H1 | Command Center | H1's current state (walking, stopped) |
| `h1_2/status` | H1_2 | Command Center | H1_2 spotted the object |
| `h1_2/control` | Command Center | H1_2 | "Continue patrolling" |
| `franka/control` | Command Center | Franka Arm | "Start pick and place" |

## The Command Center Decision Flow

```
Robot sends status update
        |
        v
Command Center receives it
        |
        v
AI model (NEXUS) analyzes the situation
        |
        v
Command Center Result (shows the decision)
        |
        v
Commands are sent to each robot
   /       |        \
  H1      H1_2    Manipulator
 (stay)  (patrol)  (pick up)
```

## Technologies Used

- **NVIDIA Isaac Sim** - Robot simulation environment
- **NVIDIA GR00T** - AI model for robot vision and control
- **n8n** - Workflow automation (the command center dashboard)
- **Ollama** - Runs the AI language models locally
- **MQTT** - Messaging between robots and the command center
- **ROS2** - Robot communication framework (used for some camera bridging)

## Running the Demo

### Prerequisites
- NVIDIA Isaac Sim installed
- n8n running with Ollama connected
- MQTT broker running (e.g., Mosquitto) on `localhost:1883`
- GR00T policy server running (`spot_groot_server.py`)

### Steps

1. **Start the MQTT broker** (if not already running):
   ```
   mosquitto
   ```

2. **Start the GR00T vision server**:
   ```
   python spot_groot_server.py
   ```

3. **Import the n8n workflows**:
   - Open n8n in your browser
   - Import all three JSON files from the `n8n_workflows/` folder
   - Make sure the MQTT and Ollama credentials are configured

4. **Activate the workflows** in n8n (toggle them on)

5. **Run the simulation**:
   ```
   # From Isaac Sim's Python environment
   python n8n_complete_Script.py
   ```

6. **Watch the demo**:
   - The Isaac Sim window shows the robots moving in the warehouse
   - The n8n dashboard shows the command center making decisions in real time
   - Click on the "Command Center Result" node after execution to see the decision details

## Robot Speeds

- **H1**: Walks at speed 1.0 (faster, reaches object first)
- **H1_2**: Walks at speed 0.3 (slower, continues patrolling)

## Important Notes

- H1_2 **never stops** - it always keeps walking regardless of what it spots
- All decisions are made by the AI command center - nothing is pre-programmed
- The Command Center Result node in n8n shows exactly what the AI decided and why
- The Franka arm only activates after H1 has fully stopped near the object
