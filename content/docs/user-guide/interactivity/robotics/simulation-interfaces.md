---
linkTitle: SimulationInterfaces
title: SimulationInterfaces Gem
description: Introduction to the SimulationInterfaces Gem
toc: true
weight: 520
---

## Overview

The `SimulationInterfaces` Gem has been created to integrate O3DE with the ROS
2 [simulation_interfaces](https://github.com/ros-simulation/simulation_interfaces) package, which is a new standard for
simulation usage. The new Gem contains the O3DE implementation together with the API based on the EBuses, as well as the
ROS 2 interface to allow communication with O3DE via the ROS 2 framework.

## Requirements

The `SimulationInterfaces` Gem requires:

- [ROS2 Gem](https://github.com/o3de/o3de-extras/tree/development/Gems/ROS2) version 3.3.0 or newer
- [simulation_interfaces](https://github.com/ros-simulation/simulation_interfaces) package installed.

## Adding SimulationInterfaces Gem to the project

Add the `SimulationInterfaces` Gem to the dependency list in the `project.json` file to use it within your project.
Gem's functionality is implemented as _System Components_, hence making all interfaces available from the start.

To verify that the Gem has been added successfully, check the list of available ROS 2 _services_ and _actions_ after
launching the GameLauncher (or running the Game Mode from the Editor).

Get available _services_ command:

```
ros2 service list
```

Get available _actions_ command:

```
ros2 action list
```

The details of the listed interfaces are provided in a later section.

## Motivation

The `SimulationInterfaces` is ROS 2 API that allows to affect simulated entities in various ways.
It can be thought of as ROS 2 API to talk with :

- [Simulated Bodies](/docs/user-guide/interactivity/physics/nvidia-physx/simulated-bodies/),
- [Spawning and despawning](/docs/learning-guide/tutorials/entities-and-prefabs/spawn-a-prefab/),
- pausing simulation,
- resetting simulation.

With `SimulationInterfaces` you can query and modify state of simulated bodies.
Those are entities with components such
as [Rigid Body Component](docs/user-guide/components/reference/physx/rigid-body/)
or [Static Rigid Body](/docs/user-guide/components/reference/physx/static-rigid-body/) and others.

{{< note >}}
Entities that have no physics, will not be available by `SimulationInterfaces`
{{< /note >}}

There is a recommendation to build your simulation assets in a tree-like structure where the trunk is
a [Rigid Body Component](docs/user-guide/components/reference/physx/rigid-body/)
or [Static Rigid Body](/docs/user-guide/components/reference/physx/static-rigid-body/) and children are other
components (like meshes, lights, and others).

The correctly designed prefab should have the following structure:\
![prefab](/static/images/user-guide/interactivity/robotics/correct_structure.png)

The `Rack` entity has [Static Rigid Body](/docs/user-guide/components/reference/physx/static-rigid-body/)
and [PhysX Mesh Collider Component](/docs/user-guide/components/reference/physx/mesh-collider/) and is available in
`SimulationInterfaces`.
Entities `Boxes`, `Lv0`, `Lv1`, `Lv2` are only decorative.
However, the entities mentioned above will follow `Rack`, due to parent-child relation
and [Transform Component](/docs/user-guide/components/reference/transform/).

The `SimulationInterfaces` allows to perform bound search.
You can specify the region of interest and ask for Simulated Entities inside.
This feature uses [Overlap Scene Query](/docs/user-guide/interactivity/physics/nvidia-physx/scene-queries/#overlap).
The results can be correctly interpreted only if the collider shapes are added to the entities.

{{< note >}}
The number of results in scene queries is pretty limited by default (32).
We highly recommend to increase this limit to Overlap Query Buffer Size
in [PhysX Configuration](/docs/user-guide/interactivity/physics/nvidia-physx/configuring/).
{{< /note >}}

## Design

### Namespaces

Users looking at the source code of the `SimulationInterfaces` Gem might be confused why there are two namespaces within
one gem - `SimulationInterfaces` and `ROS2SimulationInterfaces`. The reason for this is to distinguish the part of the
code that implements features of the `simulation_interfaces` standard in O3DE and defines API using EBuses (
`SimulationInterfaces` namespace), from the part that acts as an adapter between ROS 2 and O3DE (
`ROS2SimulationInterfaces` namespace). This also makes it possible in the future to split them into two gems -
`SimulationInterfaces` and `ROS2SimulationInterfaces` to allow users to use features implemented by the
`SimulationInterfaces` without adding the dependency to the ROS 2 framework.

### O3DE implementation

The O3DE implementation of the `simulation_interfaces` is split into four _System Components_. The reason for this is to
group features that are related into separate classes which also allows to reduce the complexity of the implementation.
Implemented _System Components_:

- `SimulationEntitiesManager` - implements the part of the API responsible for dealing with the entities (spawning,
  deleting, getting the entity state, etc.)
- `SimulationManager` - implements the part of the API responsible for controlling the simulation (pausing, reloading,
  getting the simulation state, etc.)
- `SimulationFeaturesAggregator` - implements the `GetSimulatiorFeatures` (which is part of the standard) and extends
  the API with some additional functions (e.g. `HasFeature` which allows to check if the simulator supports a certain
  feature)
- `ROS2SimulationInterfacesSystemComponent` - responsible for creating the handlers for the ROS 2 services and actions

### Services and actions

In our system, service and action handlers are designed to follow a unified architecture that promotes consistency and
extensibility. All handlers for services and action servers inherit from their respective base classes—`ROS2ServiceBase`
for service handlers and `ROS2ActionBase` for action handlers. Both of these base classes implement a common interface,
`IROS2HandlerBase`, ensuring that all handlers adhere to a standardized structure.

#### Inheritance Structure

- **Service Handlers**: Inherit from `ROS2ServiceBase`, which in turn implements the `IROS2HandlerBase`.
- **Action Handlers**: Inherit from `ROS2ActionBase`, which also implements the `IROS2HandlerBase`.

This design enforces a commonality among all handlers while allowing for the specific functionalities required by
services and actions.

#### Main Interface `IROS2HandlerBase`

The `IROS2HandlerBase` interface mandates that all handler implementations include several crucial functions. These
functions are necessary for:

1. **Registering Handlers in the System Component**: Handlers must implement methods that allow them to properly
   register within the _ROS2SimulationInterfacesSystemComponent_ - `GetTypeName` which is used as a key to identify
   certain handler object and `Initialize` if the default initialization, implemented in the base class, is not
   sufficient.

2. **Informing About Supported Features**: Every handler is required to provide information about the feature(s) it
   supports. This is essential for ensuring that `GetSimulatorFeatures` method knows about futures supported by the
   simulator.

3. **Initializing Handlers**: Handlers must implement the `GetDefaultName` method which, as the name suggests, returns
   the name of the newly created service or action server.

By following this design, developers can ensure that all new handlers are consistent, maintainable, and easily
integrable within the overall system architecture.

#### Adding new handlers

All new handler implementations must adhere to this design pattern by extending either `ROS2ServiceBase` or
`ROS2ActionBase`. This will facilitate robust integration and foster an organized approach to future development.

## Supported Features overview

This section presents the detailed description of the currently implemented and supported simulation features.

### GetSimulatorFeatures service

The `GetSimulatorFeatures` _service_ allows users to get the list
of [simulation_interfaces](https://github.com/ros-simulation/simulation_interfaces) features that are supported by the
simulator.

ROS 2 _service_
definition: [GetEntityState.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/GetSimulatorFeatures.srv) \
SimulatorFeatures
definition: [SimulatorFeatures](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/SimulatorFeatures.msg) \
Default _service_ name: `/get_simulation_features` \
O3DE EBus: `SimulationInterfaces::SimulationFeaturesAggregatorRequests::GetSimulationFeatures`

The returned list contains numbers that can be mapped to the supported features. The mapping is defined in
the [SimulatorFeatures](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/SimulatorFeatures.msg)
message.

The following features are currently supported:

- SPAWNING
- DELETING
- ENTITY_BOUNDS_BOX
- ENTITY_STATE_GETTING
- ENTITY_STATE_SETTING
- SPAWNABLES
- SIMULATION_RESET
- SIMULATION_RESET_TIME
- SIMULATION_RESET_SPAWNED
- SIMULATION_RESET_STATE
- SIMULATION_STATE_GETTING
- SIMULATION_STATE_SETTING
- SIMULATION_STATE_PAUSE
- STEP_SIMULATION_SINGLE
- STEP_SIMULATION_MULTIPLE
- STEP_SIMULATION_ACTION

### GetSpawnables service

The `GetSpawnables` _service_ outputs a list of all simulated _spawnables_ (e.g., models, robots, objects) that can be
spawned into the simulation environment.

ROS 2 _service_
definition: [GetSpawnables](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/GetSpawnables.srv) \
Individual _spawnables_ ROS 2 _message_
definition: [Spawnable.msg](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/Spawnable.msg) \
Default _service_ name: `/get_spawnables` \
O3DE EBus: `SimulationInterfaces::SimulationEntityManagerRequests::GetSpawnables`

### SpawnEntity service

The `SpawnEntity` _service_ lets you add simulated _spawnables_ into the simulation environment. As explained earlier,
you can get the available _spawnables_ with the `GetSpawnables` _service_.

ROS 2 _service_
definition: [SpawnEntity](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/SpawnEntity.srv) \
Default _service_ name: `/spawn_entity` \
O3DE EBus: `SimulationInterfaces::SimulationEntityManagerRequests::SpawnEntity`

If you want to spawn a new object into the simulation environment, you need to know the valid URI of the _spawnable_,
e.g.,`product_asset:///prefabs/robot/foorobot.spawnable`. `Simulation Entities Manager` will find corresponding Asset ID
based on the URI.

**Note:** Spawning assets from the file system e.g., `spawnable://home/username/robots/FooRobot.spawnables` is not
supported.

### GetEntities service

The `GetEntities` _service_ outputs a list of all entities that fulfill requirements provided by the standard.

ROS 2 _service_
definition: [GetEntities](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/GetEntities.srv) \
EntityFilters
definition: [EntityFilters](https://github.com/ros-simulation/simulation_interfaces/blob/main/msg/EntityFilters.msg) \
Default _service_ name: `/get_entities` \
O3DE EBus: `SimulationInterfaces::SimulationEntityManagerRequests::GetEntities`

This service works in three ways:

- returning to the caller whole cache of entities (when no filter in the query)
- returning results of Overlap Scene Query (when Bounds where set the query)
- returning results that matches provided regular expression and/or Bounds (both parameters are provided as part of the
  `EntityFilters` field)

The result is filtered by the regular expression parameter (given in filter string) and the category.

**Note:** Filtering by tags is not yet supported.

### GetEntitiesStates service

The `GetEntitiesStates` _service_ lets you get the state (speed, location, acceleration) of chosen entities.

ROS 2 _service_
definition: [GetEntitiesStates.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/GetEntitiesStates.srv) \
Individual _entity state_ ROS 2 _message_
definition: [EntityState](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/EntityState.msg) \
Default _service_ name: `/get_entities_states` \
O3DE EBus: `SimulationInterfaces::SimulationEntityManagerRequests::GetEntitiesStates`

**Note:** Acceleration is not supported. Therefore, it is not filled in the response.

### SetEntityState service

The `SetEntityState` _service_ lets you modify the state of the chosen entity. This includes the pose and the twist
parameters.

ROS 2 _service_
definition: [SetEntityState.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/SetEntityState.srv) \
Individual _entity state_ ROS 2 _message_
definition: [EntityState](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/EntityState.msg) \
Default _service_ name: `/set_entity_state` \
O3DE EBus: `SimulationInterfaces::SimulationEntityManagerRequests::SetEntityState`

### DeleteEntity service

The `DeleteEntity` _service_ lets you despawn the previously spawned entities.

ROS 2 _service_
definition: [DeleteEntity.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/DeleteEntity.srv) \
Default _service_ name: `/delete_entity` \
O3DE EBus: `SimulationInterfaces::SimulationEntityManagerRequests::DeleteEntity`

**Note:** This mechanism allows you to delete the entities that are a part of the level prefab (e.g., prefab
instantiated in Editor).

### ResetSimulation service

The `ResetSimulation` _service_ lets you reset the simulation via ROS 2 interface.

ROS 2 _service_
definition: [ResetSimulation.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/ResetSimulation.srv) \
Default _service_ name: `/reset_simulation` \
O3DE EBus: There is no additional interface for resetting the scene.

| Scope         | Planned API and usage                                                               |
|---------------|-------------------------------------------------------------------------------------|
| SCOPE_ALL     | `ConsoleRequestBus` and `LoadLevel` command.                                        |
| SCOPE_SPAWNED | Internal API to destroy all spawn tickets using `Simulation Entities manager`.      |
| SCOPE_STATE   | Move all spawned entities to initial poses cached in `Simulation Entities manager`. |
| SCOPE_TIME    | New call using `ROS2Bus`                                                            |

### SetSimulationState service

The `SetSimulationState` _service_ lets you set the state of the simulation (*STOPPED*, *PAUSED*, *PLAYING*,
*QUITTING*).

ROS 2 _service_
definition: [SetSimulationState](https://github.com/ros-simulation/simulation_interfaces/blob/main/srv/SetSimulationState.srv) \
SimulationState
definition: [SimulationState](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/SimulationState.msg) \
Default _service_ name: `/set_simulation_state` \
O3DE EBus: `SimulationInterfaces::SimulationManagerRequests::SetSimulationState`

The transition from *PLAYING* or *PAUSED* to *STOPPED* triggers level reloading.

The transition from *PLAYING* to *PAUSED* asks the default physics scene to be disabled. Hence, it stops movement of all
_PhysX_ articulations, rigid bodies (both kinematic and simulated), and characters. Some animations are played. The
transition from *PAUSED* to *PLAYING* does the opposite.

The transition from *PLAYING*, *PAUSED*, or *STOPPED* to *QUITTING* closes simulator by calling `quit` command over
`ConsoleRequestBus`.

Transitions from *STOPPED* to *PAUSED* and from *QUITTING* to any other state are forbidden.

### GetSimulationState service

The `GetSimulationState` _service_ lets you get the current state of the simulation (*PLAYING*, *PAUSED*, or *STOPPED*).

ROS 2 _service_
definition: [GetSimulationState.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/GetSimulationState.srv) \
SimulationState
definition: [SimulationState](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/SimulationState.msg) \
Default _service_ name: `/get_simulation_state` \
O3DE EBus: `SimulationInterfaces::SimulationManagerRequests::GetSimulationState`

### GetEntityState service

The `GetEntityState` _service_ lets you get the state (speed, location, acceleration) of a single entity.

ROS 2 _service_
definition: [GetEntityState.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/GetEntityState.srv) \
Individual _entity state_ ROS 2 _message_
definition: [EntityState](https://github.com/ros-simulation/simulation_interfaces/tree/main/msg/EntityState.msg) \
Default _service_ name: `/get_entity_state` \
O3DE EBus: `SimulationInterfaces::SimulationEntityManagerRequests::GetEntityState`

**Note:** Acceleration is not supported. Therefore, it is not filled in the response.

### StepSimulation service

The `StepSimulation` _service_ lets you simulate a finite number of steps.

ROS 2 _service_
definition: [StepSimulation.srv](https://github.com/ros-simulation/simulation_interfaces/tree/main/srv/StepSimulation.srv) \
Default _service_ name: `/step_simulation` \
O3DE EBus: `SimulationInterfaces::SimulationManagerRequests::StepSimulation`

**Note:** The simulator has to be paused. Otherwise, `StepSimulation` request will fail immediately.

### SimulateSteps action

The `SimulateSteps` _action_ lets you simulate a finite number of steps in a non-blocking way.

Action
definition: [SimulateSteps.action](https://github.com/ros-simulation/simulation_interfaces/blob/main/action/SimulateSteps.action) \
Default _action_ name: `/simulate_steps` \
O3DE EBus: `SimulationInterfaces::SimulationManagerRequests::StepSimulation`

**Note:** The simulator has to be paused before calling this _action_. Otherwise, `SimulateSteps` goal will be accepted
and fail immediately.
