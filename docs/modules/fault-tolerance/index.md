---
title: Fault Tolerance
date: 2021-07-11

---

# Fault Tolerance

Currently the inner handling of commands and state changes is not always robust, calls from the REST-API can fail to be carried out to devices.

Zigbee commands may fail for various reasons, like:

* End-devices are sleeping
* Routing issues
* Send queue is full
* Network overloaded with group- or broadcasts
* Target devices aren't powered

There are parts in the current code which deal with such issues, for example bindings and reporting configurations are *mostly* verified continuously.

## Tasks that might fail

### Groupcasts

Sending a new state to a light group is currently implemented as a "fire and forget" groupcast command. It has been reported multiple times that this fails partly for IKEA lights, where some lights won't react albeit the command was sent. Likely because the device local BTT table is full and the command is silently discarded.

### Unicasts

Same applies for here, for example light states are sent, but we don't verify the are actually applied. Further REST-API clients need to throttle how often they send a request. Setting the state for 50 lights ain't work because the send queue won't allow it.

### Configure a sleeping device

Sending a configuration, e.g. to a sensor, can only be done if the sensor is awake. The same is true for thermostats and blinds which albeit being light sleepers may miss the command.

### Scenes

The Zigbee scenes cluster is of rather mixed quality across vendors. The REST-API already knows what light states a certain scene **should** have, it is stored in the database. But it happens that just calling a scene doesn't restore it correctly in the device, for example my IKEA GU10 lights decided to mess up scenes after an OTA firmware update (I couldn't repair the scenes either). Handling of color temperature is another hopeless task.

### Device setup

Some devices need to be configured after joining by writing attributes or sending certain commands. This works kind of ok but isn't robust.

## Ideas

The biggest problem is the "fire and forget" approach. In the case that not all lights in a group react on the press of a remote control button it's really annoying for the user, pressing a button twice is a workaround but ugly. For fully automated systems this becomes an even larger issue, for example when thermostats are not controlled properly based on schedules or rules, or when light states aren't set based on schedules (football fields, malls, museums). More and more people having networks with over 100 nodes, here the problems increase.

#### Things we know
* The state a device **should** have;
* When sleeping devices wake up `event/awake`;
* The state a device actually has, by reading or reporting ZCL attributes;

From a very simplified point of view all we need to do is to remember the state a device **should** have, verify the actual state and carry out respective commands in case the state isn't the desired one — piece of cake — ... not really but we can get there.


#### Possible generic approach

In my device description experiments I've added a small helper class `StateChange` which holds one little state machine per logical change. A logical change is something like set on/off, set brightness with transition time, write a ZCL value.

High level view:
* Multiple `StateChange` objects can be added to any Resource
* The Device class pokes the state machine based on events (rx, polling, timeouts etc.)
* A `StateChange` doesn't know anything about lights or sensors
* It holds only references to ResourceItems which relate to ZCL values
* It holds a function pointer to carry out a certain command
* It may have an *arbitrary* long timeout to try to push the change
* It is removed from the Resource as soon as the change is finished (verified) or the timeout hits

A working very early implementation can be found at:
https://github.com/manup/deconz-rest-plugin/commit/08cc3188693757a87008be93c9f2bca80628be60

This commit implements a robust (but lazy) verification of the on/off state of a light resource.

Excerpt from `PUT /lights/<id>/state`

```c++
const quint8 cmd = taskRef.onTime > 0
                    ? ONOFF_COMMAND_ON_WITH_TIMED_OFF
                    : ONOFF_COMMAND_ON;
ok = addTaskSetOnOff(task, cmd, taskRef.onTime, 0);

StateChange change(StateChange::StateWaitSync, SC_SetOnOff, task.req.dstEndpoint());
change.addTargetValue(RStateOn, 0x01);
change.addParameter(QLatin1String("cmd"), cmd);
if (cmd == ONOFF_COMMAND_ON_WITH_TIMED_OFF)
{
    change.addParameter(QLatin1String("ontime"), taskRef.onTime);
}
taskRef.lightNode->addStateChange(change); // addStateChange() is inherited from Resource*
```
The `SC_SetOnOff` is a function pointer which will be called if the state can't be verified or on timeout.
The `addTaskSetOnOff()` isn't even needed anymore, but I kept it here to demonstrate that these two can coexist.

In future the `StateChange` or a similar approach is supposed to completely replace the `Task` based system in order to provide fault tolerance.