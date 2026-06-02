# Contribution #1: Add `preferred_pickup_distance_from_top` Attribute to PyLabRobot Resource Class

---

| Field | Details |
|---|---|
| **Contribution #** | 1 |
| **Student** | Akash Tiloda |
| **Project** | [PyLabRobot/pylabrobot](https://github.com/PyLabRobot/pylabrobot) |
| **Issue** | [#719 — resources should have a preferred_pickup_distance_from_top attribute](https://github.com/PyLabRobot/pylabrobot/issues/719) |
| **Status** | Phase I — In Progress |

---

## Why I Chose This Issue

When I came across PyLabRobot, the topic immediately stood out. An open-source SDK that translates Python code into precise robotic arm movements used in real research labs — the application and potential here are hard to ignore. The maintainers are also exploring LLM integration into the robotics planning layer, which adds another dimension to what this codebase could become.

Issue #719 caught my attention because it's a small, well-scoped change with real downstream impact. Adding a per-resource grip height preference sounds simple, but it requires understanding how the `Resource` class hierarchy works, how attributes flow through serialization, and how the movement planner consumes metadata at runtime. As a Teaching Fellow for AI301, I also want to experience the same friction my students will face — the unfamiliar codebase, the moment where a five-line change turns into tracing inheritance across multiple files. This issue is exactly that kind of problem.

---

## Understanding the Issue

### Problem Description

PyLabRobot is a hardware-agnostic Python SDK for controlling lab automation robots — Hamilton STAR, Tecan EVO, Opentrons OT-2, and others. When a robotic arm picks up a resource (a microplate, a tip rack, a reservoir), it needs a vertical offset: how far from the top of the resource should the gripper descend before closing?

Currently, that value defaults to `0` everywhere. There is no per-resource preferred grip height. Every plate and every tip rack is treated identically, even though their physical geometries differ significantly. A 96-well plate and a 384-well plate do not have the same ideal grip point.

### Expected Behavior

After the fix, each `Resource` subclass should be able to declare its own `preferred_pickup_distance_from_top` — an optional float representing the ideal grip offset in millimeters from the top of the resource. When a movement is planned and no explicit offset is passed by the caller, the robot should fall back to this preferred value instead of always defaulting to `0`.

From the maintainer's description, the goal is to "facilitate smart defaults on a resource by resource basis for robotic arm movements."

### Current Behavior

- `Resource.__init__` does not accept or store a `preferred_pickup_distance_from_top` parameter.
- Existing resource definitions (plates, tip racks, reservoirs) have no grip-height metadata.
- The movement planner has no mechanism to consult a per-resource preferred offset; it always uses `0` or whatever the caller explicitly provides.

### Affected Components

| Component | Role |
|---|---|
| `pylabrobot/resources/resource.py` | Base `Resource` class — needs the new attribute |
| Individual resource definition files (plates, tip racks, etc.) | Need to be updated with preferred values |
| Robot arm movement planner | Needs to consult `preferred_pickup_distance_from_top` as a fallback default |
| Serialization / deserialization (`to_dict` / `from_dict`) | Attribute must round-trip through JSON correctly |

---

## Reproduction Process

### Environment Setup

> To be completed in Phase II — will document Python version, virtual environment setup, and `pip install -e .` steps after local environment is confirmed working.

### Steps to Reproduce

1. Clone the repository: `git clone https://github.com/PyLabRobot/pylabrobot.git`
2. Inspect `Resource.__init__` in `pylabrobot/resources/resource.py` — observe that no `preferred_pickup_distance_from_top` parameter exists.
3. Inspect an existing plate definition (e.g., a 96-well plate file) — observe that no grip-height preference is declared.
4. Trace the arm movement planner to confirm it has no mechanism to look up a per-resource preferred offset.

### Reproduction Evidence

> To be completed in Phase II — will include code traces, screenshots, and test output confirming the current default-zero behavior.

---

## Solution Approach

### Analysis

> To be completed in Phase II — will include a full trace of the class hierarchy, the movement planner call stack, and a mapping of which resource files need updating.

### Proposed Solution (High Level)

1. **Add the attribute to the base class.** In `Resource.__init__`, add an `Optional[float]` parameter:
   ```python
   preferred_pickup_distance_from_top: Optional[float] = None
   ```
   Store it as `self.preferred_pickup_distance_from_top`.

2. **Thread it through serialization.** Update `Resource.to_dict()` to include the field, and `Resource.from_dict()` / `deserialize()` to restore it, so resource definitions round-trip correctly through JSON.

3. **Update the movement planner.** In the arm movement logic, when computing the pickup offset, fall back to `resource.preferred_pickup_distance_from_top` if no explicit offset was provided by the caller (and fall back to `0` only if the resource also has `None`).

4. **Update existing resource definitions.** Audit the existing plate, tip rack, and reservoir definition files and add `preferred_pickup_distance_from_top` values where the physical geometry makes a non-zero grip height appropriate.

5. **Add tests.** Cover the attribute on the base class, the serialization round-trip, the movement planner fallback logic, and at least one updated resource definition.

### Implementation Plan (UMPIRE Framework)

| Phase | Step | Description |
|---|---|---|
| **U — Understand** | Read the issue, trace the codebase | Understand `Resource`, the movement planner, and serialization before touching any code |
| **M — Match** | Identify analogous patterns | Find how similar optional attributes (e.g., `max_volume`) are handled in `Resource` and follow the same pattern |
| **P — Plan** | Draft the change set | List every file that needs editing; confirm with a maintainer comment if uncertain about scope |
| **I — Implement** | Write the code | Add the attribute, update serialization, update the planner, update resource definitions |
| **R — Review** | Self-review and test | Run the existing test suite, write new tests, check for regressions |
| **E — Evaluate** | Open the PR | Submit the pull request, respond to maintainer feedback, iterate |

---

## Testing Strategy

### Unit Tests

- `test_resource_preferred_pickup_distance_default_is_none` — verify that a plain `Resource` instance has `preferred_pickup_distance_from_top = None` by default.
- `test_resource_preferred_pickup_distance_can_be_set` — verify that passing a float at construction time stores it correctly.
- `test_resource_serialization_round_trip` — verify that `to_dict()` / `from_dict()` preserves the attribute (both `None` and a non-None value).

### Integration Tests

- `test_arm_movement_uses_preferred_distance_when_no_explicit_offset_given` — verify that the movement planner reads `preferred_pickup_distance_from_top` from the resource when the caller does not supply an explicit offset.
- `test_arm_movement_explicit_offset_overrides_preferred` — verify that an explicit caller-supplied offset takes precedence over the resource's preferred value.

### Manual Testing

> To be completed in Phase II — will include running the existing PyLabRobot simulator or test fixtures to confirm end-to-end behavior matches expectations before submitting the PR.

---

## Implementation Notes

### Week 1 Progress

- [x] Identified and researched the issue
- [x] Read the PyLabRobot paper (Device, 2023) and project documentation
- [ ] Traced the `Resource` class hierarchy and located `resource.py`
- [ ] Identified the serialization pattern used by existing attributes
- [ ] Set up local development environment (Phase II)
- [ ] Confirmed scope with maintainer via issue comment (Phase II)

### Code Changes

> None yet — implementation begins in Phase II.

### Pull Request

> To be opened in Phase IV after implementation and testing are complete.

---

## Learnings & Reflections

> To be completed as the contribution progresses.

---

## Resources Used

- [PyLabRobot GitHub Repository](https://github.com/PyLabRobot/pylabrobot)
- [Issue #719](https://github.com/PyLabRobot/pylabrobot/issues/719)
- [PyLabRobot paper — Device (2023)](https://www.cell.com/device/fulltext/S2666-9986(23)00046-4)
- [PyLabRobot documentation](https://docs.pylabrobot.org)
- CodePath AI301 course materials