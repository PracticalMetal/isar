# ISAR

[![Python package](https://github.com/equinor/isar/actions/workflows/pythonpackage.yml/badge.svg)](https://github.com/equinor/isar/actions/workflows/pythonpackage.yml)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![License](https://img.shields.io/badge/License-EPL_2.0-blue.svg)](https://opensource.org/licenses/EPL-2.0)

ISAR - Integration and Supervisory control of Autonomous Robots - is a tool for integrating robot applications into
operator systems. Through the ISAR API you can send commands to a robot to do missions and collect results from the
missions.

## Components

The system consists of two main components.

1. State machine
1. FastAPI

### State machine

The state machine handles interaction with the robots API and monitors the execution of missions. It also enables
interacting with the robot before, during and after missions.

The state machine is based on the [transitions](https://github.com/pytransitions/transitions) package for Python. The
main states are:

- Idle: The robot is in an idle state and ready to perform new missions.
- Send: The state machine has received a mission and is sending the mission to the robot.
- Monitor: The robot has received a mission from the state machine, and the state machine is monitoring the current
  mission.
- Collect: The state collects data during a mission.
- Cancel: The state machine has received a request to abort, or an event has occurred which requires the mission to be
  canceled. The cancel state also functions as a wrap-up state when a mission is finished, prior to the state machine
  returning to idle.

### FastAPI

The FastAPI establishes an interface to the state machine for the user. As the API and state machine are separate
threads, they communicate through python queues. FastAPI runs on an ASGI-server, specifically uvicorn. The
FastAPI-framework is split into routers where the endpoint operations are defined.

## Installation

ISAR may be installed through pip as

```bash
pip install isar
```

However, to run ISAR you are required to clone this repository and run:

```bash
python main.py
```

Note, running the full system requires that an implementation of a robot has been installed. See
this [section](#robot-integration) for installing a mocked robot or a Turtlebot3 simulator.

## Robot integration

To connect the state machine to a robot in a separate repository, it is required that the separate repository implements
the [robot interface](https://github.com/equinor/isar/blob/main/src/robot_interface/robot_interface.py). A mocked robot
can be found in [this repository](https://github.com/equinor/isar-robot). Install the repo, i.e:

```bash
pip install git+https://@github.com/equinor/isar-robot.git@main
```

Then, ensure the `robot_package` variable
in [default.ini](https://github.com/equinor/isar/blob/62312ac6844c06f93d98dce8db05d8172f22bca0/src/isar/config/default.ini#L4)
is set to the name of the package you installed. `isar_robot` is set by default.

If you have the robot repository locally, you can simply install through

```bash
pip install -e /path/to/robot/repo/
```

### Running ISAR with a robot simulator

A simulator based on the open source robot Turtlebot3 has been implemented for use with ISAR and may be
found [here](https://github.com/equinor/isar-turtlebot). Follow the installation instructions for the simulator and
install `isar-turtlebot` in the same manner as given in the [robot integration](#robot-integration) section. Set the
following configuration variables
in [default.ini](https://github.com/equinor/isar/blob/62312ac6844c06f93d98dce8db05d8172f22bca0/src/isar/config/default.ini#L4):

```bash
robot_package = isar_turtlebot
default_map = turtleworld
```

## Running a robot mission

Once the application has been started the swagger site may be accessed at

```
http://localhost:3000/docs
```

Execute the `/schedule/start-mission` endpoint with `mission_id=1` to run a mission.

In [this](./src/isar/config/predefined_missions) folder there are predefined default missions, for example the mission
corresponding to `mission_id=1`. A new mission may be added by adding a new json-file with a mission description. Note,
the mission IDs must be unique.

## <a name="dev"></a>Development

For local development, please fork the repository. Then, clone and install in the repository root folder:

```
git clone https://github.com/equinor/isar
cd isar
pip install -e .[dev]
```

For `zsh` you might have to type `".[dev]"`

Verify that you can run the tests:

```bash
pytest .
```

## Mission planner

The mission planner that is currently in use is defined by the `mission_planner` configuration variable. This can be set
in the [default configuration](./src/isar/config/default.ini). The available options are

```
mission_planner = local
mission_planner = echo
```

By default, the `local` planner is used.

### Implement your own planner

You can create your own mission planner by implementing
the [mission planner interface](./src/isar/mission_planner/mission_planner_interface.py) and adding your planner to the
selection [here](./src/isar/modules.py). Note that you must add your module as an option in the dictionary.

## Storage

The storage module that is currently in use is defined by the `storage` configuration variable. This can be set in
the [default configuration](./src/isar/config/default.ini). The available options are

```
storage = local
storage = blob
```

### Implement your own storage module

You can create your own storage module by implementing the [storage interface](./src/isar/storage/storage_interface.py)
and adding your storage module to the selection [here](./src/isar/modules.py). Note that you must add your module as an
option in the dictionary.

## API authentication

The API has an option to include user authentication. This can be set in
the [default configuration](./src/isar/config/default.ini).

```
authentication_enabled = false
authentication_enabled = true
```

By default, the `local` storage module is used and API authentication is disabled. If using Azure Blob Storage a set of
environment variables must be available which gives access to an app registration that may use the storage account.
Enabling API authentication also requires the same environment variables. The required variables are

```
AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_CLIENT_SECRET
```

## Running tests

After following the steps in [Development](#dev), you can run the tests:

```bash
pytest .
```

To create an interface test in your robot repository, use the function `interface_test` from `robot_interface`. The
argument should be an interface object from your robot specific implementation.
See [isar-robot](https://github.com/equinor/isar-robot/blob/main/tests/interfaces/test_robotinterface.py) for example.

### Integration tests

Integration tests can be found [here](https://github.com/equinor/isar/tree/main/tests/integration) and have been created
with a simulator in mind. The integration tests will not run as part of `pytest .` or as part of the CI/CD pipeline. To
run the integration tests please follow the instructions in [this section](#running-isar-with-a-robot-simulator) for
setting up the `isar-turtlebot` implementation with simulator and run the following command once the simulation has been
launched.

```bash
pytest tests/integration
```

Note that these tests will run towards the actual simulation (you may monitor it through Gazebo and RVIZ) and it will
take a long time.

## Documentation

To build the project documentation, run the following commands:

```bash
cd docs
make docs
```

The documentation can now be viewed at `docs/build/html/index.html`.

## Contributing

We welcome all kinds of contributions, including code, bug reports, issues, feature requests, and documentation. The
preferred way of submitting a contribution is to either make an [issue](https://github.com/equinor/isar/issues) on
GitHub or by forking the project on GitHub and making a pull requests.
