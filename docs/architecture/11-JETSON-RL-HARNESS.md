# Jetson RL Harness: Training Open-Source Models from Claude Trajectories

## Executive Summary

The Jetson RL Harness is a system for observing Claude's behavior inside Jetson, collecting structured trajectories, computing reward signals, and using this data to post-train open-source models (Llama, Mistral, Qwen, etc.). This creates a "teacher-student" pipeline where Claude's expert behavior trains smaller, deployable models.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         JETSON RL HARNESS OVERVIEW                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                 │
│   │   JETSON     │    │  TRAJECTORY  │    │   REWARD     │                 │
│   │   + CLAUDE   │───▶│  COLLECTOR   │───▶│   COMPUTER   │                 │
│   │  (Teacher)   │    │              │    │              │                 │
│   └──────────────┘    └──────────────┘    └──────────────┘                 │
│          │                   │                   │                          │
│          │                   ▼                   ▼                          │
│          │            ┌──────────────────────────────┐                     │
│          │            │     TRAJECTORY DATASET       │                     │
│          │            │  (state, action, reward, next)│                     │
│          │            └──────────────────────────────┘                     │
│          │                          │                                       │
│          │                          ▼                                       │
│          │            ┌──────────────────────────────┐                     │
│          │            │      TRAINING PIPELINE       │                     │
│          │            │   SFT → DPO → RLHF → Eval    │                     │
│          │            └──────────────────────────────┘                     │
│          │                          │                                       │
│          ▼                          ▼                                       │
│   ┌──────────────┐    ┌──────────────────────────────┐                     │
│   │   JETSON     │    │    OPEN-SOURCE MODEL         │                     │
│   │  + STUDENT   │◀───│  (Llama/Mistral/Qwen)        │                     │
│   │   MODEL      │    │       (Student)              │                     │
│   └──────────────┘    └──────────────────────────────┘                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Core Concepts

### 1.1 Why This Approach?

**Problem**: Claude is powerful but expensive/proprietary. Open-source models are cheap but less capable at agentic tasks.

**Solution**: Use Claude as an expert "teacher" - observe its behavior on real tasks, then distill that knowledge into open-source "student" models.

**Key Insight**: The Jetson environment provides a controlled, observable setting where every action Claude takes can be logged, evaluated, and used as training signal.

### 1.2 Learning Paradigms

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LEARNING PARADIGM OPTIONS                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. SUPERVISED FINE-TUNING (SFT)                                     │   │
│  │    - Direct imitation: Copy what Claude does                        │   │
│  │    - Input: (context, task) → Output: Claude's response             │   │
│  │    - Simple but limited to seen scenarios                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. DIRECT PREFERENCE OPTIMIZATION (DPO)                             │   │
│  │    - Learn from comparisons: Claude's choice vs alternatives        │   │
│  │    - Input: (context, chosen, rejected) → Learn preference          │   │
│  │    - More efficient than RLHF, no reward model needed               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. REINFORCEMENT LEARNING (PPO/GRPO)                                │   │
│  │    - Learn from outcomes: Did the task succeed?                     │   │
│  │    - Reward signal from task completion, code quality, tests        │   │
│  │    - Most powerful but requires careful reward design               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 4. HYBRID: SFT → DPO → RL                                           │   │
│  │    - Start with imitation, refine with preferences, optimize        │   │
│  │    - Best results but most complex pipeline                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Collection Architecture

### 2.1 What to Observe

Every Claude action in Jetson generates observable data:

```python
@dataclass
class AgentObservation:
    """Single observation of Claude's behavior."""

    # === CONTEXT (State before action) ===
    session_id: str
    step_number: int
    timestamp: datetime

    # Task context
    original_task: str                    # User's request
    conversation_history: List[Message]   # Full conversation so far
    current_files: Dict[str, str]         # Files in context
    codebase_summary: str                 # Compressed codebase understanding

    # Environment state
    working_directory: str
    git_status: GitStatus
    open_files: List[str]
    terminal_state: Optional[str]

    # === ACTION (What Claude chose) ===
    thinking: Optional[str]               # Claude's reasoning (if available)
    action_type: ActionType               # tool_call, response, question
    tool_name: Optional[str]              # read_file, edit_file, bash, etc.
    tool_arguments: Optional[Dict]        # Full tool parameters
    raw_response: str                     # Complete model output

    # === RESULT (Outcome of action) ===
    tool_result: Optional[str]            # Tool output
    error: Optional[str]                  # Any error message
    files_changed: List[FileDiff]         # What changed in codebase

    # === METADATA ===
    model_id: str                         # claude-3-opus, etc.
    tokens_used: TokenCount
    latency_ms: int
```

### 2.2 Trajectory Structure

A complete task trajectory:

```python
@dataclass
class Trajectory:
    """Complete record of a task from start to finish."""

    # Identity
    trajectory_id: str
    session_id: str

    # Task definition
    task_description: str
    task_type: TaskType                   # bug_fix, feature, refactor, etc.
    codebase_id: str                      # Which repo
    initial_state: CodebaseSnapshot       # State before task

    # Sequence of observations
    observations: List[AgentObservation]

    # Final state
    final_state: CodebaseSnapshot
    task_completed: bool
    completion_reason: str                # success, error, timeout, abandoned

    # Computed metrics (see Section 3)
    rewards: TrajectoryRewards
    quality_scores: QualityScores

    # Human feedback (optional)
    human_rating: Optional[int]           # 1-5 rating
    human_feedback: Optional[str]         # Free-form feedback
```

### 2.3 Collection Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DATA COLLECTION ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         JETSON RUNTIME                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │   │
│  │  │  Agent   │  │  Tool    │  │ Provider │  │Permission│           │   │
│  │  │  Loop    │  │Executor  │  │  Layer   │  │  System  │           │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘           │   │
│  │       │             │             │             │                  │   │
│  │       └─────────────┴─────────────┴─────────────┘                  │   │
│  │                           │                                        │   │
│  │                           ▼                                        │   │
│  │              ┌────────────────────────┐                           │   │
│  │              │      EVENT BUS         │                           │   │
│  │              │  (All events flow here)│                           │   │
│  │              └────────────────────────┘                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    OBSERVATION COLLECTOR                            │   │
│  │                                                                     │   │
│  │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐       │   │
│  │   │ Event Listener │  │ State Snapshot │  │ Diff Computer  │       │   │
│  │   │                │  │                │  │                │       │   │
│  │   │ - tool_called  │  │ - files        │  │ - file changes │       │   │
│  │   │ - tool_result  │  │ - git status   │  │ - git diff     │       │   │
│  │   │ - llm_request  │  │ - env vars     │  │ - state delta  │       │   │
│  │   │ - llm_response │  │ - cwd          │  │                │       │   │
│  │   │ - task_start   │  │                │  │                │       │   │
│  │   │ - task_end     │  │                │  │                │       │   │
│  │   └────────────────┘  └────────────────┘  └────────────────┘       │   │
│  │            │                  │                  │                  │   │
│  │            └──────────────────┼──────────────────┘                  │   │
│  │                               ▼                                     │   │
│  │                    ┌────────────────────┐                          │   │
│  │                    │ Observation Builder│                          │   │
│  │                    └────────────────────┘                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    TRAJECTORY STORAGE                               │   │
│  │                                                                     │   │
│  │   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐   │   │
│  │   │  SQLite    │  │   Parquet  │  │    S3/     │  │  HuggingFace│   │   │
│  │   │  (local)   │  │  (export)  │  │    GCS     │  │   Datasets  │   │   │
│  │   └────────────┘  └────────────┘  └────────────┘  └────────────┘   │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Event Types to Capture

```python
class EventType(Enum):
    """All events the RL harness captures."""

    # Session lifecycle
    SESSION_START = "session_start"
    SESSION_END = "session_end"

    # Task lifecycle
    TASK_RECEIVED = "task_received"
    TASK_COMPLETED = "task_completed"
    TASK_FAILED = "task_failed"
    TASK_ABANDONED = "task_abandoned"

    # LLM interactions
    LLM_REQUEST = "llm_request"
    LLM_RESPONSE = "llm_response"
    LLM_STREAM_CHUNK = "llm_stream_chunk"

    # Tool calls
    TOOL_CALLED = "tool_called"
    TOOL_RESULT = "tool_result"
    TOOL_ERROR = "tool_error"

    # Specific tools (for detailed analysis)
    FILE_READ = "file_read"
    FILE_WRITE = "file_write"
    FILE_EDIT = "file_edit"
    BASH_EXECUTED = "bash_executed"
    GIT_OPERATION = "git_operation"

    # Code changes
    CODE_CHANGED = "code_changed"
    TESTS_RUN = "tests_run"
    BUILD_RUN = "build_run"

    # Human interaction
    HUMAN_INPUT = "human_input"
    HUMAN_FEEDBACK = "human_feedback"
    PERMISSION_REQUESTED = "permission_requested"
    PERMISSION_GRANTED = "permission_granted"
    PERMISSION_DENIED = "permission_denied"
```

---

## 3. Reward Signal Design

### 3.1 Reward Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         REWARD SIGNAL HIERARCHY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ OUTCOME REWARDS (Did it work?)                              Weight: 40% │
│  │                                                                     │   │
│  │   task_completed:     +10.0  (binary: did the task succeed?)       │   │
│  │   tests_pass:         +5.0   (all tests green after changes)       │   │
│  │   build_succeeds:     +3.0   (code compiles/builds)                │   │
│  │   no_regressions:     +2.0   (didn't break existing functionality) │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ EFFICIENCY REWARDS (How well?)                              Weight: 25% │
│  │                                                                     │   │
│  │   steps_efficiency:   +0.0 to +5.0 (fewer steps = better)          │   │
│  │   token_efficiency:   +0.0 to +3.0 (fewer tokens = better)         │   │
│  │   file_reads:         -0.1 per unnecessary read                    │   │
│  │   failed_attempts:    -0.5 per failed tool call                    │   │
│  │   backtracking:       -1.0 per undo/revert                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ CODE QUALITY REWARDS (How good?)                            Weight: 20% │
│  │                                                                     │   │
│  │   linter_score:       +0.0 to +3.0 (no new warnings)               │   │
│  │   type_safety:        +0.0 to +2.0 (proper types)                  │   │
│  │   test_coverage:      +0.0 to +2.0 (tests for new code)            │   │
│  │   code_style:         +0.0 to +1.0 (follows project conventions)   │   │
│  │   complexity:         -0.5 per high-complexity function added      │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PROCESS REWARDS (Good practices?)                           Weight: 15% │
│  │                                                                     │   │
│  │   read_before_edit:   +1.0  (always read file before editing)      │   │
│  │   incremental_test:   +1.0  (test after changes)                   │   │
│  │   proper_commits:     +0.5  (good commit messages)                 │   │
│  │   safety_checks:      +0.5  (verify dangerous operations)          │   │
│  │   ask_when_unclear:   +0.5  (ask human rather than guess)          │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Reward Computation

```python
@dataclass
class RewardConfig:
    """Configuration for reward computation."""

    # Outcome weights
    task_completed_reward: float = 10.0
    tests_pass_reward: float = 5.0
    build_succeeds_reward: float = 3.0
    no_regressions_reward: float = 2.0

    # Efficiency weights
    optimal_steps_reward: float = 5.0
    step_penalty_factor: float = 0.1      # Penalty per extra step
    token_efficiency_reward: float = 3.0
    failed_attempt_penalty: float = 0.5
    backtrack_penalty: float = 1.0

    # Quality weights
    max_linter_reward: float = 3.0
    max_type_reward: float = 2.0
    max_coverage_reward: float = 2.0
    complexity_penalty: float = 0.5

    # Process weights
    read_before_edit_reward: float = 1.0
    incremental_test_reward: float = 1.0
    safety_check_reward: float = 0.5


class RewardComputer:
    """Computes reward signals from trajectory."""

    def __init__(self, config: RewardConfig):
        self.config = config

    def compute_trajectory_reward(
        self,
        trajectory: Trajectory
    ) -> TrajectoryRewards:
        """Compute all rewards for a trajectory."""

        return TrajectoryRewards(
            outcome=self._compute_outcome_reward(trajectory),
            efficiency=self._compute_efficiency_reward(trajectory),
            quality=self._compute_quality_reward(trajectory),
            process=self._compute_process_reward(trajectory),
            total=self._compute_total_reward(trajectory),

            # Per-step rewards for RL training
            step_rewards=self._compute_step_rewards(trajectory)
        )

    def _compute_outcome_reward(self, trajectory: Trajectory) -> float:
        """Did the task succeed?"""
        reward = 0.0

        if trajectory.task_completed:
            reward += self.config.task_completed_reward

        if trajectory.final_state.tests_passing:
            reward += self.config.tests_pass_reward

        if trajectory.final_state.build_successful:
            reward += self.config.build_succeeds_reward

        if not trajectory.final_state.has_regressions:
            reward += self.config.no_regressions_reward

        return reward

    def _compute_efficiency_reward(self, trajectory: Trajectory) -> float:
        """How efficiently was the task completed?"""
        reward = 0.0

        # Compare to baseline (could be learned or heuristic)
        optimal_steps = self._estimate_optimal_steps(trajectory.task_type)
        actual_steps = len(trajectory.observations)

        if actual_steps <= optimal_steps:
            reward += self.config.optimal_steps_reward
        else:
            excess = actual_steps - optimal_steps
            reward -= excess * self.config.step_penalty_factor

        # Penalize failed attempts
        failed_tools = sum(
            1 for obs in trajectory.observations
            if obs.error is not None
        )
        reward -= failed_tools * self.config.failed_attempt_penalty

        # Penalize backtracking (git revert, undo edits)
        backtracks = self._count_backtracks(trajectory)
        reward -= backtracks * self.config.backtrack_penalty

        return max(0, reward)  # Floor at 0

    def _compute_step_rewards(
        self,
        trajectory: Trajectory
    ) -> List[float]:
        """
        Compute per-step rewards for RL training.
        Uses reward shaping to provide dense signal.
        """
        step_rewards = []

        for i, obs in enumerate(trajectory.observations):
            reward = 0.0

            # Immediate tool success/failure
            if obs.error is None:
                reward += 0.1
            else:
                reward -= 0.2

            # Progress toward goal (requires task-specific heuristics)
            progress = self._estimate_progress(trajectory, i)
            reward += progress * 0.5

            # Process quality at this step
            if self._read_before_edit_at_step(trajectory, i):
                reward += 0.1

            step_rewards.append(reward)

        # Add terminal reward to last step
        if trajectory.task_completed:
            step_rewards[-1] += self.config.task_completed_reward

        return step_rewards
```

### 3.3 Automatic Reward Signals

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AUTOMATIC REWARD SIGNALS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Signal Source          How to Compute                       Reliability   │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  Test Suite             Run tests before/after, compare       ★★★★★       │
│                         pytest --tb=short → pass/fail                      │
│                                                                             │
│  Build System           Run build, check exit code            ★★★★★       │
│                         npm run build → 0 or 1                             │
│                                                                             │
│  Type Checker           Run type checker, count errors        ★★★★☆       │
│                         mypy . → error count                               │
│                                                                             │
│  Linter                 Run linter, count warnings            ★★★★☆       │
│                         eslint . → warning count                           │
│                                                                             │
│  Git Diff               Analyze changes for quality           ★★★☆☆       │
│                         Lines added/removed, files changed                 │
│                                                                             │
│  Complexity Metrics     AST analysis of code changes          ★★★☆☆       │
│                         Cyclomatic complexity, nesting depth               │
│                                                                             │
│  Execution Trace        Did code run without exceptions?      ★★★★☆       │
│                         Run changed code, catch errors                     │
│                                                                             │
│  LLM-as-Judge           Ask another LLM to rate quality       ★★☆☆☆       │
│                         "Rate this code change 1-10"                       │
│                                                                             │
│  Human Feedback         Explicit thumbs up/down               ★★★★★       │
│                         But expensive and slow                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Gym Environment Interface

### 4.1 JetsonGym Environment

```python
"""
JetsonGym: OpenAI Gym-compatible environment for training coding agents.

Usage:
    env = JetsonGym(codebase="my-project", task_distribution="bug_fixes")
    obs = env.reset()

    while not done:
        action = agent.predict(obs)
        obs, reward, done, info = env.step(action)
"""

import gymnasium as gym
from gymnasium import spaces
from typing import Dict, Any, Tuple, Optional
import numpy as np


class JetsonGym(gym.Env):
    """
    Gym environment wrapping Jetson for RL training.

    Observation Space:
        - conversation_history: Tokenized conversation
        - file_contents: Currently open files
        - codebase_embedding: Compressed codebase representation
        - tool_history: Recent tool calls and results

    Action Space:
        - Discrete: Choose which tool to call
        - Continuous: Tool arguments (tokenized)

    Or simplified:
        - Text: Full model output (tool call as text)
    """

    metadata = {"render_modes": ["human", "ansi"]}

    def __init__(
        self,
        codebase: str,
        task_distribution: str = "mixed",
        max_steps: int = 50,
        reward_config: Optional[RewardConfig] = None,
        teacher_model: str = "claude-3-opus",
        render_mode: Optional[str] = None,
    ):
        super().__init__()

        self.codebase = codebase
        self.task_distribution = task_distribution
        self.max_steps = max_steps
        self.reward_config = reward_config or RewardConfig()
        self.teacher_model = teacher_model
        self.render_mode = render_mode

        # Initialize Jetson environment
        self.jetson = JetsonEnvironment(codebase)
        self.reward_computer = RewardComputer(self.reward_config)
        self.task_generator = TaskGenerator(task_distribution)

        # Current episode state
        self.current_task: Optional[Task] = None
        self.trajectory: Optional[Trajectory] = None
        self.step_count: int = 0

        # Define spaces (simplified for text-based actions)
        self.observation_space = spaces.Dict({
            "task": spaces.Text(max_length=10000),
            "conversation": spaces.Text(max_length=100000),
            "files": spaces.Text(max_length=500000),
            "last_result": spaces.Text(max_length=50000),
        })

        self.action_space = spaces.Text(max_length=10000)

    def reset(
        self,
        seed: Optional[int] = None,
        options: Optional[Dict] = None,
    ) -> Tuple[Dict[str, Any], Dict[str, Any]]:
        """Reset environment with new task."""
        super().reset(seed=seed)

        # Generate or use provided task
        if options and "task" in options:
            self.current_task = options["task"]
        else:
            self.current_task = self.task_generator.generate()

        # Reset Jetson environment
        self.jetson.reset(self.current_task)

        # Initialize trajectory
        self.trajectory = Trajectory(
            trajectory_id=generate_id(),
            session_id=self.jetson.session_id,
            task_description=self.current_task.description,
            task_type=self.current_task.type,
            codebase_id=self.codebase,
            initial_state=self.jetson.snapshot(),
            observations=[],
        )

        self.step_count = 0

        obs = self._get_observation()
        info = {"task": self.current_task}

        return obs, info

    def step(
        self,
        action: str,
    ) -> Tuple[Dict[str, Any], float, bool, bool, Dict[str, Any]]:
        """
        Execute action in environment.

        Args:
            action: Model output (tool call as text, or response)

        Returns:
            observation: New state
            reward: Immediate reward
            terminated: Task completed
            truncated: Max steps reached
            info: Additional information
        """
        self.step_count += 1

        # Parse and execute action
        parsed_action = self._parse_action(action)
        result = self.jetson.execute(parsed_action)

        # Record observation
        observation = AgentObservation(
            session_id=self.jetson.session_id,
            step_number=self.step_count,
            timestamp=datetime.now(),
            original_task=self.current_task.description,
            conversation_history=self.jetson.conversation,
            action_type=parsed_action.type,
            tool_name=parsed_action.tool_name,
            tool_arguments=parsed_action.arguments,
            raw_response=action,
            tool_result=result.output,
            error=result.error,
            files_changed=result.file_changes,
            model_id="student",  # Training a student model
            tokens_used=TokenCount(input=0, output=len(action)),
            latency_ms=result.latency_ms,
        )
        self.trajectory.observations.append(observation)

        # Compute reward
        step_reward = self.reward_computer.compute_step_reward(
            self.trajectory,
            self.step_count - 1
        )

        # Check termination
        terminated = self._check_task_completed()
        truncated = self.step_count >= self.max_steps

        if terminated or truncated:
            self.trajectory.final_state = self.jetson.snapshot()
            self.trajectory.task_completed = terminated
            self.trajectory.rewards = self.reward_computer.compute_trajectory_reward(
                self.trajectory
            )

        obs = self._get_observation()
        info = {
            "step": self.step_count,
            "tool_result": result,
            "trajectory": self.trajectory if (terminated or truncated) else None,
        }

        return obs, step_reward, terminated, truncated, info

    def _get_observation(self) -> Dict[str, Any]:
        """Get current observation."""
        return {
            "task": self.current_task.description,
            "conversation": self.jetson.format_conversation(),
            "files": self.jetson.format_open_files(),
            "last_result": self.jetson.last_result or "",
        }

    def _parse_action(self, action: str) -> ParsedAction:
        """Parse model output into structured action."""
        # Parse tool calls from text
        # Handle various formats (XML, JSON, etc.)
        return ActionParser.parse(action)

    def _check_task_completed(self) -> bool:
        """Check if task is successfully completed."""
        return self.jetson.check_task_completion(self.current_task)

    def get_teacher_action(self) -> str:
        """Get Claude's action for the current state (for imitation)."""
        return self.jetson.get_teacher_response(self.teacher_model)

    def collect_teacher_trajectory(self) -> Trajectory:
        """Let Claude complete the task, return full trajectory."""
        obs, info = self.reset()

        while True:
            # Get Claude's action
            action = self.get_teacher_action()

            obs, reward, terminated, truncated, info = self.step(action)

            if terminated or truncated:
                break

        return self.trajectory
```

### 4.2 Task Generator

```python
class TaskGenerator:
    """Generates diverse coding tasks for training."""

    def __init__(
        self,
        distribution: str = "mixed",
        codebase_path: Optional[str] = None,
    ):
        self.distribution = distribution
        self.codebase_path = codebase_path

        # Task type weights
        self.weights = {
            "mixed": {
                "bug_fix": 0.3,
                "feature": 0.25,
                "refactor": 0.15,
                "test": 0.1,
                "docs": 0.05,
                "debug": 0.1,
                "explain": 0.05,
            },
            "bug_fixes": {"bug_fix": 1.0},
            "features": {"feature": 1.0},
        }[distribution]

    def generate(self) -> Task:
        """Generate a random task."""
        task_type = self._sample_task_type()

        generators = {
            "bug_fix": self._generate_bug_fix,
            "feature": self._generate_feature,
            "refactor": self._generate_refactor,
            "test": self._generate_test,
            "docs": self._generate_docs,
            "debug": self._generate_debug,
            "explain": self._generate_explain,
        }

        return generators[task_type]()

    def _generate_bug_fix(self) -> Task:
        """Generate a bug fix task."""
        # Option 1: Inject synthetic bug
        # Option 2: Use known bugs from issue tracker
        # Option 3: Introduce mutation testing style bugs

        if self.codebase_path:
            # Find a function and introduce a bug
            file, function = self._random_function()
            bug_type = random.choice(["off_by_one", "null_check", "typo", "logic"])

            mutated_code = self._inject_bug(file, function, bug_type)

            return Task(
                type="bug_fix",
                description=f"There's a bug in the {function} function. Tests are failing. Please fix it.",
                setup_commands=[f"git apply {mutated_code}"],
                validation=lambda: self._run_tests(),
                metadata={"file": file, "function": function, "bug_type": bug_type},
            )

        # Synthetic task
        return Task(
            type="bug_fix",
            description="Fix the failing test in test_calculator.py",
            validation=lambda: self._run_tests(),
        )
```

---

## 5. Training Pipeline

### 5.1 Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          TRAINING PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 1: DATA COLLECTION                                             │  │
│  │                                                                      │  │
│  │   Run Claude on diverse tasks  ───▶  Collect 10k+ trajectories      │  │
│  │   Filter successful completions  ───▶  Quality filter               │  │
│  │   Human verification (sample)  ───▶  Gold standard subset           │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 2: SUPERVISED FINE-TUNING (SFT)                                │  │
│  │                                                                      │  │
│  │   Base model: Llama-3.1-70B or Qwen2.5-72B                          │  │
│  │   Train on (context, claude_response) pairs                         │  │
│  │   Loss: Cross-entropy on Claude's outputs                           │  │
│  │   Output: SFT model that imitates Claude                            │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 3: PREFERENCE LEARNING (DPO/IPO)                               │  │
│  │                                                                      │  │
│  │   Create preference pairs:                                          │  │
│  │     - Claude's choice vs SFT model's choice                         │  │
│  │     - Successful vs failed trajectories                             │  │
│  │     - High reward vs low reward actions                             │  │
│  │   Train with DPO objective                                          │  │
│  │   Output: Model with better action selection                        │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 4: REINFORCEMENT LEARNING (PPO/GRPO)                           │  │
│  │                                                                      │  │
│  │   Use JetsonGym environment                                         │  │
│  │   Reward from automated signals (tests, build, quality)             │  │
│  │   Train policy with PPO or GRPO                                     │  │
│  │   Output: Model optimized for task completion                       │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 5: EVALUATION & DEPLOYMENT                                     │  │
│  │                                                                      │  │
│  │   Benchmark on held-out tasks                                       │  │
│  │   Compare to Claude baseline                                        │  │
│  │   Human evaluation on sample                                        │  │
│  │   Deploy as Jetson backend option                                   │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 SFT Training

```python
"""
Supervised Fine-Tuning on Claude trajectories.
"""

from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from trl import SFTTrainer
from datasets import Dataset


def prepare_sft_dataset(trajectories: List[Trajectory]) -> Dataset:
    """Convert trajectories to SFT format."""

    examples = []

    for traj in trajectories:
        # Only use successful trajectories
        if not traj.task_completed:
            continue

        # Convert to conversation format
        for i, obs in enumerate(traj.observations):
            # Context: everything up to this point
            context = format_context(
                task=traj.task_description,
                conversation=traj.observations[:i],
                files=obs.current_files,
            )

            # Target: Claude's response
            target = obs.raw_response

            examples.append({
                "prompt": context,
                "completion": target,
                "task_type": traj.task_type.value,
                "step": i,
                "total_steps": len(traj.observations),
            })

    return Dataset.from_list(examples)


def train_sft(
    base_model: str = "meta-llama/Llama-3.1-70B-Instruct",
    trajectories: List[Trajectory] = None,
    output_dir: str = "./jetson-sft",
):
    """Train SFT model on Claude trajectories."""

    # Load base model
    model = AutoModelForCausalLM.from_pretrained(
        base_model,
        torch_dtype=torch.bfloat16,
        device_map="auto",
    )
    tokenizer = AutoTokenizer.from_pretrained(base_model)

    # Prepare dataset
    dataset = prepare_sft_dataset(trajectories)

    # Training arguments
    training_args = TrainingArguments(
        output_dir=output_dir,
        num_train_epochs=3,
        per_device_train_batch_size=4,
        gradient_accumulation_steps=4,
        learning_rate=2e-5,
        warmup_ratio=0.1,
        logging_steps=10,
        save_steps=500,
        bf16=True,
    )

    # Initialize trainer
    trainer = SFTTrainer(
        model=model,
        tokenizer=tokenizer,
        args=training_args,
        train_dataset=dataset,
        formatting_func=lambda x: f"{x['prompt']}\n{x['completion']}",
        max_seq_length=8192,
    )

    # Train
    trainer.train()
    trainer.save_model()

    return model
```

### 5.3 DPO Training

```python
"""
Direct Preference Optimization for better action selection.
"""

from trl import DPOTrainer, DPOConfig


def prepare_dpo_dataset(
    trajectories: List[Trajectory],
    sft_model: AutoModelForCausalLM,
) -> Dataset:
    """Create preference pairs for DPO."""

    examples = []

    for traj in trajectories:
        for i, obs in enumerate(traj.observations):
            context = format_context(
                task=traj.task_description,
                conversation=traj.observations[:i],
                files=obs.current_files,
            )

            # Chosen: Claude's action
            chosen = obs.raw_response

            # Rejected: Options
            # 1. SFT model's response (if different)
            sft_response = generate_response(sft_model, context)
            if sft_response != chosen:
                examples.append({
                    "prompt": context,
                    "chosen": chosen,
                    "rejected": sft_response,
                    "source": "sft_comparison",
                })

            # 2. Failed alternatives from other trajectories
            # 3. Random tool choices (negative mining)

    # Also add trajectory-level comparisons
    successful = [t for t in trajectories if t.task_completed]
    failed = [t for t in trajectories if not t.task_completed]

    for succ, fail in zip(successful, failed):
        if succ.task_type == fail.task_type:
            # Compare first actions
            examples.append({
                "prompt": format_context(succ.task_description, [], {}),
                "chosen": succ.observations[0].raw_response,
                "rejected": fail.observations[0].raw_response,
                "source": "trajectory_comparison",
            })

    return Dataset.from_list(examples)


def train_dpo(
    sft_model_path: str,
    trajectories: List[Trajectory],
    output_dir: str = "./jetson-dpo",
):
    """Train DPO model on preference pairs."""

    # Load SFT model
    model = AutoModelForCausalLM.from_pretrained(sft_model_path)
    ref_model = AutoModelForCausalLM.from_pretrained(sft_model_path)
    tokenizer = AutoTokenizer.from_pretrained(sft_model_path)

    # Prepare dataset
    dataset = prepare_dpo_dataset(trajectories, model)

    # DPO config
    config = DPOConfig(
        output_dir=output_dir,
        num_train_epochs=1,
        per_device_train_batch_size=2,
        gradient_accumulation_steps=8,
        learning_rate=5e-7,
        beta=0.1,  # KL penalty coefficient
        bf16=True,
    )

    # Initialize trainer
    trainer = DPOTrainer(
        model=model,
        ref_model=ref_model,
        tokenizer=tokenizer,
        config=config,
        train_dataset=dataset,
    )

    trainer.train()
    trainer.save_model()

    return model
```

### 5.4 RL Training (PPO)

```python
"""
Reinforcement Learning with PPO on JetsonGym.
"""

from trl import PPOTrainer, PPOConfig, AutoModelForCausalLMWithValueHead


def train_ppo(
    dpo_model_path: str,
    output_dir: str = "./jetson-ppo",
    total_episodes: int = 10000,
):
    """Train with PPO on live environment."""

    # Load DPO model with value head
    model = AutoModelForCausalLMWithValueHead.from_pretrained(dpo_model_path)
    tokenizer = AutoTokenizer.from_pretrained(dpo_model_path)

    # PPO config
    config = PPOConfig(
        output_dir=output_dir,
        learning_rate=1e-6,
        batch_size=16,
        mini_batch_size=4,
        gradient_accumulation_steps=4,
        ppo_epochs=4,
        kl_coef=0.1,
        cliprange=0.2,
    )

    # Initialize trainer
    trainer = PPOTrainer(
        model=model,
        tokenizer=tokenizer,
        config=config,
    )

    # Initialize environment
    env = JetsonGym(
        codebase="training-codebase",
        task_distribution="mixed",
        max_steps=30,
    )

    # Training loop
    for episode in range(total_episodes):
        obs, info = env.reset()

        queries = []
        responses = []
        rewards = []

        done = False
        while not done:
            # Get model's action
            query = tokenizer.encode(obs["conversation"], return_tensors="pt")
            response = model.generate(query, max_new_tokens=1000)
            action = tokenizer.decode(response[0])

            # Execute in environment
            next_obs, reward, terminated, truncated, info = env.step(action)

            queries.append(query)
            responses.append(response)
            rewards.append(reward)

            obs = next_obs
            done = terminated or truncated

        # PPO update
        stats = trainer.step(queries, responses, rewards)

        if episode % 100 == 0:
            print(f"Episode {episode}: reward={sum(rewards):.2f}, stats={stats}")

    trainer.save_model()
    return model
```

---

## 6. Data Infrastructure

### 6.1 Storage Schema

```sql
-- Trajectory storage schema

CREATE TABLE trajectories (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    task_description TEXT NOT NULL,
    task_type TEXT NOT NULL,
    codebase_id TEXT NOT NULL,

    -- State snapshots (JSON blobs)
    initial_state BLOB,
    final_state BLOB,

    -- Outcome
    task_completed BOOLEAN,
    completion_reason TEXT,

    -- Computed rewards
    outcome_reward REAL,
    efficiency_reward REAL,
    quality_reward REAL,
    process_reward REAL,
    total_reward REAL,

    -- Human feedback
    human_rating INTEGER,
    human_feedback TEXT,

    -- Metadata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    model_id TEXT,
    total_tokens INTEGER,
    total_latency_ms INTEGER
);

CREATE TABLE observations (
    id TEXT PRIMARY KEY,
    trajectory_id TEXT REFERENCES trajectories(id),
    step_number INTEGER NOT NULL,
    timestamp TIMESTAMP NOT NULL,

    -- Context
    conversation_history BLOB,  -- JSON
    current_files BLOB,         -- JSON

    -- Action
    action_type TEXT NOT NULL,
    tool_name TEXT,
    tool_arguments BLOB,        -- JSON
    raw_response TEXT,
    thinking TEXT,

    -- Result
    tool_result TEXT,
    error TEXT,
    files_changed BLOB,         -- JSON

    -- Metrics
    tokens_input INTEGER,
    tokens_output INTEGER,
    latency_ms INTEGER,

    -- Per-step reward
    step_reward REAL
);

CREATE INDEX idx_trajectories_task_type ON trajectories(task_type);
CREATE INDEX idx_trajectories_completed ON trajectories(task_completed);
CREATE INDEX idx_trajectories_reward ON trajectories(total_reward);
CREATE INDEX idx_observations_trajectory ON observations(trajectory_id);
```

### 6.2 Export Formats

```python
def export_to_huggingface(
    trajectories: List[Trajectory],
    dataset_name: str = "jetson-trajectories",
):
    """Export trajectories to HuggingFace dataset format."""

    # SFT format
    sft_examples = []
    for traj in trajectories:
        for obs in traj.observations:
            sft_examples.append({
                "prompt": format_prompt(traj, obs),
                "completion": obs.raw_response,
                "task_type": traj.task_type.value,
                "reward": obs.step_reward,
            })

    sft_dataset = Dataset.from_list(sft_examples)
    sft_dataset.push_to_hub(f"{dataset_name}-sft")

    # DPO format
    dpo_examples = create_preference_pairs(trajectories)
    dpo_dataset = Dataset.from_list(dpo_examples)
    dpo_dataset.push_to_hub(f"{dataset_name}-dpo")

    # Full trajectory format (for research)
    full_dataset = Dataset.from_list([
        traj.to_dict() for traj in trajectories
    ])
    full_dataset.push_to_hub(f"{dataset_name}-full")


def export_to_parquet(
    trajectories: List[Trajectory],
    output_path: str,
):
    """Export to Parquet for efficient storage and querying."""

    import pyarrow as pa
    import pyarrow.parquet as pq

    # Flatten to tabular format
    rows = []
    for traj in trajectories:
        for obs in traj.observations:
            rows.append({
                "trajectory_id": traj.trajectory_id,
                "task_type": traj.task_type.value,
                "task_completed": traj.task_completed,
                "total_reward": traj.rewards.total,
                "step_number": obs.step_number,
                "action_type": obs.action_type.value,
                "tool_name": obs.tool_name,
                "step_reward": traj.rewards.step_rewards[obs.step_number],
                "prompt": format_prompt(traj, obs),
                "response": obs.raw_response,
            })

    table = pa.Table.from_pylist(rows)
    pq.write_table(table, output_path, compression="snappy")
```

---

## 7. Integration with Jetson

### 7.1 Collection Mode

```python
# jetson/rl/collector.py

class TrajectoryCollector:
    """
    Integrates with Jetson event bus to collect trajectories.

    Usage:
        collector = TrajectoryCollector(storage_path="./trajectories.db")

        # Register with event bus
        event_bus.subscribe("*", collector.on_event)

        # Trajectories are automatically collected
    """

    def __init__(
        self,
        storage_path: str,
        reward_config: Optional[RewardConfig] = None,
        auto_compute_rewards: bool = True,
    ):
        self.storage = TrajectoryStorage(storage_path)
        self.reward_config = reward_config or RewardConfig()
        self.reward_computer = RewardComputer(self.reward_config)
        self.auto_compute_rewards = auto_compute_rewards

        # Active trajectories by session
        self.active_trajectories: Dict[str, Trajectory] = {}

    def on_event(self, event: Event) -> None:
        """Handle event from Jetson event bus."""

        handlers = {
            EventType.SESSION_START: self._on_session_start,
            EventType.SESSION_END: self._on_session_end,
            EventType.TASK_RECEIVED: self._on_task_received,
            EventType.LLM_RESPONSE: self._on_llm_response,
            EventType.TOOL_CALLED: self._on_tool_called,
            EventType.TOOL_RESULT: self._on_tool_result,
            EventType.TASK_COMPLETED: self._on_task_completed,
        }

        handler = handlers.get(event.type)
        if handler:
            handler(event)

    def _on_session_start(self, event: Event) -> None:
        """Initialize trajectory for new session."""
        session_id = event.data["session_id"]

        self.active_trajectories[session_id] = Trajectory(
            trajectory_id=generate_id(),
            session_id=session_id,
            task_description="",
            task_type=TaskType.UNKNOWN,
            codebase_id=event.data.get("codebase", "unknown"),
            initial_state=None,
            observations=[],
        )

    def _on_task_received(self, event: Event) -> None:
        """Record task start."""
        session_id = event.data["session_id"]
        traj = self.active_trajectories.get(session_id)

        if traj:
            traj.task_description = event.data["task"]
            traj.task_type = classify_task(event.data["task"])
            traj.initial_state = snapshot_codebase()

    def _on_llm_response(self, event: Event) -> None:
        """Record LLM response as observation."""
        session_id = event.data["session_id"]
        traj = self.active_trajectories.get(session_id)

        if traj:
            obs = AgentObservation(
                session_id=session_id,
                step_number=len(traj.observations),
                timestamp=datetime.now(),
                original_task=traj.task_description,
                conversation_history=event.data.get("conversation", []),
                raw_response=event.data["response"],
                action_type=parse_action_type(event.data["response"]),
                tool_name=parse_tool_name(event.data["response"]),
                tool_arguments=parse_tool_args(event.data["response"]),
                model_id=event.data.get("model", "unknown"),
                tokens_used=event.data.get("tokens", TokenCount()),
                latency_ms=event.data.get("latency_ms", 0),
            )
            traj.observations.append(obs)

    def _on_tool_result(self, event: Event) -> None:
        """Record tool result."""
        session_id = event.data["session_id"]
        traj = self.active_trajectories.get(session_id)

        if traj and traj.observations:
            # Update last observation with result
            traj.observations[-1].tool_result = event.data.get("result")
            traj.observations[-1].error = event.data.get("error")
            traj.observations[-1].files_changed = event.data.get("file_changes", [])

    def _on_session_end(self, event: Event) -> None:
        """Finalize and store trajectory."""
        session_id = event.data["session_id"]
        traj = self.active_trajectories.pop(session_id, None)

        if traj and traj.observations:
            traj.final_state = snapshot_codebase()
            traj.task_completed = event.data.get("completed", False)
            traj.completion_reason = event.data.get("reason", "session_end")

            # Compute rewards
            if self.auto_compute_rewards:
                traj.rewards = self.reward_computer.compute_trajectory_reward(traj)

            # Store
            self.storage.save(traj)
```

### 7.2 Configuration

```yaml
# jetson/config/rl_harness.yaml

rl_harness:
  enabled: true

  collection:
    storage_path: "./data/trajectories.db"
    export_format: "parquet"  # or "sqlite", "huggingface"
    auto_export_threshold: 1000  # Export every N trajectories

    # What to collect
    capture_thinking: true
    capture_files: true
    capture_git_state: true
    max_file_size: 100000  # Truncate large files

  rewards:
    # Outcome weights
    task_completed: 10.0
    tests_pass: 5.0
    build_succeeds: 3.0

    # Efficiency weights
    step_penalty: 0.1
    failed_attempt_penalty: 0.5

    # Quality weights
    linter_reward: 3.0

    # Automatic validation
    run_tests_on_complete: true
    run_build_on_complete: true
    run_linter_on_complete: true

  training:
    # Teacher model for comparison
    teacher_model: "claude-3-opus"

    # Student model to train
    student_base_model: "meta-llama/Llama-3.1-70B-Instruct"

    # Training stages
    sft:
      enabled: true
      epochs: 3
      learning_rate: 2e-5

    dpo:
      enabled: true
      epochs: 1
      beta: 0.1

    ppo:
      enabled: false  # Requires more infrastructure
      episodes: 10000
```

---

## 8. Evaluation Framework

### 8.1 Benchmark Suite

```python
class JetsonBenchmark:
    """
    Standardized benchmark for evaluating trained models.
    """

    TASK_SETS = {
        "swe_bench_lite": [
            # Subset of SWE-bench tasks
        ],
        "bug_fix_100": [
            # 100 curated bug fix tasks
        ],
        "feature_50": [
            # 50 feature implementation tasks
        ],
        "refactor_30": [
            # 30 refactoring tasks
        ],
    }

    def __init__(
        self,
        model: AutoModelForCausalLM,
        tokenizer: AutoTokenizer,
        task_set: str = "bug_fix_100",
    ):
        self.model = model
        self.tokenizer = tokenizer
        self.tasks = self.TASK_SETS[task_set]

    def evaluate(self) -> BenchmarkResults:
        """Run full evaluation."""

        results = []

        for task in self.tasks:
            env = JetsonGym(
                codebase=task.codebase,
                max_steps=task.max_steps,
            )

            obs, info = env.reset(options={"task": task})

            trajectory = []
            done = False

            while not done:
                # Get model action
                action = self._generate_action(obs)

                obs, reward, terminated, truncated, info = env.step(action)
                trajectory.append({"action": action, "reward": reward})

                done = terminated or truncated

            results.append({
                "task_id": task.id,
                "completed": info.get("trajectory", {}).get("task_completed", False),
                "steps": len(trajectory),
                "total_reward": sum(t["reward"] for t in trajectory),
            })

        return BenchmarkResults(
            task_set=self.task_set,
            results=results,
            completion_rate=sum(r["completed"] for r in results) / len(results),
            avg_steps=sum(r["steps"] for r in results) / len(results),
            avg_reward=sum(r["total_reward"] for r in results) / len(results),
        )
```

### 8.2 Comparison Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MODEL COMPARISON MATRIX                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Model                  Completion  Avg Steps  Avg Reward  Cost/Task       │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  Claude Opus (Teacher)     85%         12.3       18.5       $2.50         │
│  Claude Sonnet             72%         15.1       14.2       $0.50         │
│                                                                             │
│  Llama-70B (Base)          35%         28.4        5.2       $0.10         │
│  Llama-70B + SFT           58%         18.2       11.3       $0.10         │
│  Llama-70B + DPO           65%         15.8       13.1       $0.10         │
│  Llama-70B + PPO           71%         14.2       14.8       $0.10         │
│                                                                             │
│  Qwen-72B (Base)           38%         26.1        5.8       $0.10         │
│  Qwen-72B + SFT            61%         17.5       11.9       $0.10         │
│  Qwen-72B + Full Pipeline  73%         13.8       15.2       $0.10         │
│                                                                             │
│  * Cost estimated at typical cloud GPU pricing                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Practical Considerations

### 9.1 Data Quality

**Filtering Criteria:**
1. Only use trajectories where task was completed successfully
2. Remove trajectories with excessive backtracking (>3 reverts)
3. Filter out tasks that took >50 steps (likely too hard)
4. Verify test suite actually tests the changed code
5. Human spot-check 1% of trajectories for quality

**Negative Examples:**
- Failed trajectories are valuable for DPO (rejected examples)
- Common failure modes become negative training signal
- But don't train on corrupted or malicious examples

### 9.2 Compute Requirements

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       COMPUTE REQUIREMENTS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Stage                 GPU Memory    Time          Cost (Cloud)            │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  Data Collection       N/A           ~$50/1k traj  (API costs)             │
│  (10k trajectories)                  ~$500 total                           │
│                                                                             │
│  SFT (70B model)       8x A100 80GB  24-48 hours   ~$2,000                 │
│                                                                             │
│  DPO (70B model)       8x A100 80GB  12-24 hours   ~$1,000                 │
│                                                                             │
│  PPO (70B model)       16x A100 80GB 48-96 hours   ~$8,000                 │
│  (10k episodes)                                                            │
│                                                                             │
│  Full Pipeline                                     ~$12,000                │
│                                                                             │
│  * LoRA reduces cost by ~60% but with quality tradeoff                     │
│  * Smaller models (7B-13B) are ~10x cheaper                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Legal & Ethical Considerations

1. **Model License**: Check if distillation from Claude is permitted
2. **Data Privacy**: Don't include proprietary code in training data
3. **Attribution**: Credit source of training signal
4. **Capability Limits**: Trained model may have different safety properties
5. **Evaluation**: Thoroughly test for harmful behaviors before deployment

---

## 10. Future Directions

### 10.1 Self-Improvement Loop

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       SELF-IMPROVEMENT LOOP                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                    ┌─────────────────────┐                                 │
│                    │     Claude         │                                 │
│                    │   (Initial Teacher) │                                 │
│                    └──────────┬──────────┘                                 │
│                               │ Generate trajectories                      │
│                               ▼                                            │
│                    ┌─────────────────────┐                                 │
│                    │   Train Student v1  │                                 │
│                    └──────────┬──────────┘                                 │
│                               │                                            │
│                               ▼                                            │
│      ┌────────────────────────────────────────────────┐                   │
│      │                                                │                   │
│      ▼                                                ▼                   │
│  ┌───────────┐                                 ┌───────────────┐          │
│  │ Student v1│  ──── Generate new data ────▶  │ Filter by     │          │
│  │ runs tasks│                                 │ reward signal │          │
│  └───────────┘                                 └───────┬───────┘          │
│                                                        │                   │
│                                                        ▼                   │
│                                               ┌───────────────┐           │
│                                               │ Train Student │           │
│                                               │      v2       │           │
│                                               └───────┬───────┘           │
│                                                       │                    │
│                                                       │ (repeat)           │
│                                                       ▼                    │
│                                                    ...                     │
│                                                                             │
│  Key: Each generation uses its own outputs (filtered) to train next        │
│       Eventually may exceed original teacher on specific tasks             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Research Questions

1. **Curriculum Learning**: What order of tasks produces best generalization?
2. **Reward Hacking**: How to prevent model from gaming reward signals?
3. **Distribution Shift**: How to handle new codebases/languages?
4. **Thinking Distillation**: Can we transfer reasoning, not just actions?
5. **Multi-Agent**: Train specialized models for different subtasks?

---

## Appendix A: Quick Start

```bash
# 1. Collect trajectories
jetson collect --output trajectories.db --count 1000

# 2. Export to training format
jetson export --input trajectories.db --format huggingface --output ./dataset

# 3. Train SFT
python train_sft.py \
  --base_model meta-llama/Llama-3.1-70B-Instruct \
  --dataset ./dataset \
  --output ./jetson-sft

# 4. Train DPO (optional)
python train_dpo.py \
  --sft_model ./jetson-sft \
  --dataset ./dataset \
  --output ./jetson-dpo

# 5. Evaluate
jetson benchmark --model ./jetson-dpo --task_set bug_fix_100

# 6. Deploy
jetson serve --model ./jetson-dpo --port 8000
```

---

## Appendix B: Related Work

- **SWE-bench**: Software engineering benchmark for agents
- **WebArena**: Web agent benchmark with reward signals
- **AgentTuning**: Instruction tuning for agent capabilities
- **FireAct**: Fine-tuning language agents with trajectories
- **ReAct**: Reasoning and acting in language models
- **Toolformer**: Self-taught tool use
- **CodeAct**: Code action agents

---

*Document Version: 1.0*
*Last Updated: Phase v0*
*Status: Design Document*
