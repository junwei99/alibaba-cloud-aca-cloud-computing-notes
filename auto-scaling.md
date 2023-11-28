# Auto Scaling

## Introduction

### Scale out

Adds additional computing resource to the pool during peak times

### Scale in

Releases ECS instances when demand decreases

### Health Checks

When unhealthy instance is detected, automatically replaces it with a new one

## Concepts

### Key functions

1. Scaling operations

   - Scheduled scaling

     - Tell Auto Scaling to perform a scaling operation at a specific time. e.g. scale up to X instances at 13:00 everyday

   - Dynamic scaling

     - Dynamically scales up and down based on Cloud Monitor metric (CPU & Network usage)

2. Supports SLB auto configuration

3. Supports RDS access whitelist auto-configuration

### Terminology

| Concept               | Description                                                                                                                                                                                    |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Scaling group         | Collection of ECS instances with similar configuration deployed in an application scenario. It defines the max and min number of ECS instances in the group, SLB and RDS instances, and others |
| Scaling Configuration | Defines the configuration information of ECS instances                                                                                                                                         |
| Scaling Rule          | Defines specific scaling actions, e.g. adding or removing ECS instances                                                                                                                        |
| Scaling Activity      | When a scaling rule is triggered, a scaling activity is generated. Adds or removes ECS instances from Scaling Group                                                                            |
| Scaling Trigger Task  | Tasks that trigger a scaling rule, such as scheduled tasks or Cloud Monitor alert tasks                                                                                                        |
| Cool-down period      | Period which Auto Scaling cannot execute any new scaling activity (triggered by last successful scaling activity)                                                                              |

### Usage

#### Auto scaling usage steps :

1. Create a scaling group
2. Create scaling configuration
3. Enable scaling group
4. Create scaling rule
5. Create scheduled task
6. CloudMonitor API Put Alarm Rule

#### Relationship between core concepts :

- Scaling group includes **scaling configuration, scaling rules and scaling activities**

- Remove scaling group also deletes scaling configuration, rules & activites associated

- Scaling trigger tasks fall into 2 categories: scheduled tasks & cloud monitor alert tasks. Tasks are independent of the Scaling Group

### Auto Scaling considerations

#### Scaling Rules

- Automatically adjust the number ECS instances according to MaxSize & MinSize of scaling group

- e.g

  - if number of ECS instances is set to 50 in scaling rule, but MaxSize of scaling group is 45

  - Scaling rule will only be able to increase to 45 instances

#### Cooldown time

- During cooldown period, **only Scaling Activity requests from CloudMonitor alarm tasks are rejected by Scaling Group**

- Other tasks (manually executed scaling rules & scheduled tasks) can immediately trigger scaling activities without waiting for cooldown period to expire

- Starts after the last ECS instance is added to or removed from the Scaling Group by Scaling Activity

#### Scaling Activity

- **Only 1 scaling activity can be executed at a time in a Scaling Group**

- Scaling Activity cannot be interrupted.

  - e.g. if Scaling Activity to add 20 ECS instances is being executed, it cannot be forced to terminate when only 5 instances have been created

- if fails to add or remove ECS instances to or from a scaling group, the system rolls back ECS instances, not the Scaling Activity.

  - e.g. system created 20 ECS instances, but only 19 instances are added to SLB, it will release the failed ECS instances

- Uses Alibaba Cloud's RAM service to replace ECS instances through the ECS API, the rolled back ECS instance will still incur a cost for the time they were running.

### Billing

- Free service

- ECS instances that Auto Scaling automatically creates or manually adds to the scaling group will be charged.

- Take note: you will still be charged even if you **stop** the ECS instances, you must **release** the instances to avoid further fees

## Scaling Group

### Create a scaling group

- Enter Max & Min number of instances allowed for scaling.

- Enter Default Cool-down Time (Sec), Removal Policy and Network Type (VPC)

- Select the Server Load Balancer and RDS database instance info, as needed

### Query & Modify a scaling group

- lifecycle states: Active, Inactive, Deleting

- When modifying a Scaling Group :

  - Region, SLB, RDS database cannot be modified

  - Can only modify **Active** or **Inactive** Scaling Groups

  - If number of ECS instances does not meet new MaxSize or MinSize settings, Auto Scaling adds or removes ECS instances to/from the group until the MaxSize or MinSize value is reached

### Enable & Disable a Scaling Group

#### Enable

- Can only enable inactive Scaling Groups with active Scaling Configurations

- Single Scaling Group can have one active Scaling Configuration at a time

- Can enable a Scaling Group such that if the existing number of ECS is less than the MinSize of an active Scaling Group, Auto Scaling created PAYG ECS instances until the MinSize value is reached

### Disable

- Can only be disabled when it is not executing any Scaling Activity

- Can only execute for Active Scaling group

### Delete a Scaling Group

- By default, a Scaling Group is deleted in ForceDelete mode, from the console

- Deleting a Scaling Group:

  - Deleting a scaling group also deletes its Scaling Configurations, Scaling Rules, Scaling Activities, and scaling requests

  - Does not delete scheduled tasks, CloodMonitor alarm tasks, SLB or RDS

## Scaling Configuration

### Query a Scaling Configuration

#### Lifecycle state

- **Active**: uses Scaling Configuration in the active state to create ECS instances

- **Inactive**: still in Scaling Group, but not used to create ECS instances

### Create a Scaling Configuration

- Scaling Configuration defines the configuration of ECS instances which will be added to the Scaling Group by Scaling Activities.

- Auto Scaling automatically adds ECS instances into Scaling Groups according to the Scaling Configuration

- Can modify a Scaling Configuration after it is created

- Up to 10 Scaling Configurations for 1 Scaling Group

### Delete a Scaling Configuration

- Active Scaling Configurations cannot be deleted

- If contains instances which were created by a particular Scaling Configuration, then it cannot be deleted

## Scaling Rule

### Create a Scaling Rule

1. Defines specific scaling actions, how many instances should be added to or removed from the Scaling Group.

2. If as a result of executing Scaling Rule, the number of ECS in a scaling group is less than the MinSize or greater than the MaxSize, **Auto Scaling automatically adjusts the number of ECS instances to be added or removed**

   - Example 1

     - A scaling group with MaxSize 3, Total Capacity 2

     - A scaling rule is set to "Add 3 ECS instances".

     - Auto Scaling only add 1 ECS instance

   - Example 2

     - A scaling group with MaxSize 2, Total Capacity 3

     - A scaling rule is set to "Remove 5 ECS instances".

     - Auto Scaling only removes 1 ECS instance

### Execute a Scaling Rule

You may execute a scaling rule give the following conditions:

1. If the Scaling Group is active and no Scaling Activity is being executed, the Scaling Rule is executed directly without waiting for the cooldown time

2. If number of ECS to be added by the Scaling Rule + the number of existing ECS exceeds the MaxSize, total capacity is adjusted to the MaxSize value

3. If the number of existing ECS instances - number of ECS to be removed by the Scaling Rule is less than the MinSize value, the Total Capacity is adjusted to the MinSize value

### Add an ECS instance

- if Scaling Group is active & no Scaling Activity is being executed, adding an ECS is executed directly without waiting for cooldown time

- number of ECS to be added + number of existing ECS in the Scaling Group **exceeds MaxSize, the operation fails**

- Manually added ECS are not associated with active Scaling Configuration in the Scaling Group

### Remove an ECS instance

- Removal depends on "Reclaim Mode" you choose for a Scaling Group. Instances will either be stopped (Shutdown and Reclaim mode) or released (Release Mode)

- When a manually added ECS is removed from Scaling Group, the instance is not stopped or released

- If no Scaling Activity is being executed, removal is executed direcly without waiting for cooldown time

- if number of existing ECS - number of ECS to be added < MinSize, it fails

### Create & Manage a scheduled task

1. Can create up to 20 scheduled tasks

2. If scheduled task fails to trigger the execution of a Scaling Rule because it is executing a Scaling Activity or Scaling Group is disabled

   - the scheduled task is **abandoned**

3. If multiple tasks are scheduled at similar times to execture a Scaling Rule for a Scaling group

   - **Earliest task triggers** the scaling activity first

   - Other tasks will attempt to execute the rule within their Launch Expiration Time

   - Scaling group executes only 1 Scaling Activity at a time. If another scheduled task is still triggering attempts within its launch Expiration Time after the Scaling Activity is finished

   - the Scaling Rule is executed and the corresponding Scaling Activity is triggered

4. If multiple tasks are scheduled at the same time, the **latest scheduled** task is executed

### Create & manage the Event-Triggered task

- Do not need to be unique

- Cannot execute during a Scaling Groups' cooldown period

- Rejected if the Scaling Group is already executing another Scaling Activity
