---
pageid: 1490944013
title: km-events

---

- [km-events](#km-events)
  - [Event Segments](#event-segments)
  - [Event Lifecycle](#event-lifecycle)
    - [created](#created)
    - [opening](#opening)
    - [running](#running)
    - [tabulating](#tabulating)
    - [closing](#closing)
    - [closed](#closed)
    - [pendingDelete](#pendingDelete)
  - [km-events Subscriptions](#km-events-subscriptions)
  - [km-gateway Endpoints](#km-gateway-endpoints)
    - [collectPrize](#collectprize)
    - [getEventLeaderboard](#geteventleaderboard)
  - [km-sim Components](#km-sim-components)
    - [Quest](#quest)
  - [Event Types](#event-types)
    - [solo](#solo)
    - [solo-leaderboard](#solo-leaderboard)
    - [alliance](#alliance)
    - [alliance-leaderboard](#alliance-leaderboard)
  - [Analytics](#analytics)
    - [event_enter](#event_enter)
    - [event_leave](#event_leave)
    - [event_phase](#event_phase)
    - [event_prize_collected](#event_prize_collected)
    - [event_prize_earned](#event_prize_earned)
    - [quest_step_collect](#quest_step_collect)
- [TODO](#todo)

# km-events

The km-events service is responsible for the lifecycle of in-game events including scheduling, segmentation, leaderboards, and prizes.

## Event Segments

by shard, by time zone offset

## Event Lifecycle

km-events automatically progresses events through multiple phases during their lifetime:

### created

An event initially enters the `created` phase while km-events is initially creating it. During this phase players may be added to the event, but it should not yet be visible to them.

### opening

An event will transition from `created` to `opening` at its `openingTime`. During the `opening` phase, the event should be visible to the user and its planned start time displayed to them.

### running

An event will transition from `opening` to `running` at its `runningTime`. During the `running` phase, the event should be visible to the user and they may earn points in the event by completing quests associated with the event.

### tabulating

An event will transition from `running` to `tabulating` at its `tabulatingTime`. During the `tabulating` phase, the event should be visible to the user though no further progress in the event may be made. This period (which should be between 0 and 1 minutes) is used to collect points earned in the last seconds of combat and finalize leaderboards and prizes.

### closing

An event will transition from `tabluating` to `closing` at its `closingTime`. During the `closing` phase, players may collect their prizes and view leaderboards associated with the event/

### closed

An event will transition from `closing` to `closed` at its `closedTime`. When entering the closed phase, any prizes that a player has not collected will be collected for them. Once an event has reached the `closed` phase, it is complete and should no longer be visible to the player.

### pendingDelete

An event will transition from `closed` to `pendingDelete` once it has completed the automatic prize collection process (not when `deleteTime` has been reached). When an event is in the `pendingDelete` phase and `deleteTime` has been reached, all data related to the event will be purged. The `deleteTime` for an event defaults to its `closedTime` and is never changed, even if `closedTime` is moved forward. This effectively creates a lock on the event being rescheduled before its original completion time.

## km-events Subscriptions

There is a server-managed subscription associated with km-events that sends data to the client about the state of any events it should be aware of as well as the player's individual progress in that event. The full details can be found in `message EventsChanged` in `events.proto` in `km-core`.

At a high-level, `EventsChanged` contains a list of events the player is involved in as well as a list of participants records which contain the player's current points total as well as an earned and collected prizes.

## km-gateway Endpoints

### collectPrize

To collect a prize, use the km-gateway endpoint `collectPrize` specifiying the `eventId` and `prizeId` that you wish to collect. The reponse to that request will contain a list of awarded items.

For solo events, prizes may be collected during the `running`, `tabulating` or `closing` phases.

For leaderboard events, prizes may only be collected during the `closing` phase (there will be none earned until that point anyway).

### getEventLeaderboard

To retrieve a page of a leaderboard, use the km-gateway endpoing `getEventLeaderboard` specifying the `eventId`, `groupId` and `page` (0 indexed) of the leaderboard you wish to retrieve. To iterate through the leaderboard, the client should send increasingly larger values for `page` until you receive an empty result set at which point you have reached the end of the leaderboard.

The response contains an array `entries` of type `EventLeaderboardEntry`. Each entry in the leaderboard has the following fields:

| field    | description                                                                    |
| -------- | ------------------------------------------------------------------------------ |
| index    | The 0-based index of this entry in the leaderboard. It is independent of ties. |
| place    | The 1-based place of this entry in the leaderboard. It is dependent on ties.   |
| playerId | The playerId at this index in the leaderboard                                  |
| points   | The number of points the playerId has earned in the event                      |

The difference between index and place is best demonstrated when their is a tie on the leaderboard. If there is a tie, one of the players (chosen at random) will have the higher index and should display first on the leaderboard. However, all of the tied players will have the same place on the leaderboard. Additionally, the entry following a tie will see its place jump so that there is a gap in the leaderboard.

The following is an example of a leaderboard response with a tie on it -- there are three players tied for 2nd place and there are no players in either 3rd or 4th place.

```json
{
  "eventId": "ev.lb.test",
  "groupId": "0",
  "offset": 0,
  "limit": 10,
  "entries": [
    { "index": 0, "place": 1, "playerId": "abc", "points": 1000 },
    { "index": 1, "place": 2, "playerId": "def", "points": 500 },
    { "index": 2, "place": 2, "playerId": "ghi", "points": 500 },
    { "index": 3, "place": 2, "playerId": "jkl", "points": 500 },
    { "index": 4, "place": 5, "playerId": "mno", "points": 250 },
    { "index": 5, "place": 6, "playerId": "pqr", "points": 200 }
  ]
}
```

Leaderboards update approximately once per minute and the client SHOULD NOT request specific pages of the leaderboard more frequently than that. If necessary, the server could provide time of last update/time of next update to the client but it does not currently do so.

## km-sim Components

### Quest

The Quest component (`q2$`) has an additional field eventId (`eid`) that contains the eventId of the event that the quest is associated with (if one exists).

## Event Types

All current event types are based around running one or more quests for the user during the event which can be used to collect points.

NOTE: The points earned for completing a quest are currently stored in two places. The correct place to look is the `eventPoints` field on steps in the quest prototype. Ignore the previous `mat.pt.xxx` records that exist in the `rewards` field, those will be removed. This was done to remove the need for having n unique `mat.pt.xxx` prototypes, one for each event.

### solo

A solo event has no leaderboard and the player earns prizes during the running phase of the event as they hit certain point milestones on the prize list. This is identical to the previous daily goals model (and in fact the new daily gaols event is a solo event).

#### Redis structures
* KME:\${eventId}
  * Type: HASHSET
  * Key: \${playerId}
  * Value: player total points

### solo-leaderboard

A solo-leaderboard event has one leaderboard for each group in the event. Players earn points during the event and a leaderboard is constantly updated with the player's place in the event. Prizes are earned when the event enters the closing phase based on the player's final place on the leaderboard.

#### Redis structures
* KME:\${eventId}-\${groupId}
  * Type: SORTEDSET
  * Key: \${playerId}
  * Value: player total points
* KME:\${eventId}-\${groupId}:tiebreaker
  * Type: HASHSET
  * Key: \${playerId}
  * Value: last time the player scored points

### alliance

not yet implemented

#### Redis structures
* KME:\${eventId}
  * Type: HASHSET
  * Key: \${allianceId}
  * Value: alliance total points
* KME:\${eventId}:alliance:\${allianceId}
  * Type: HASHSET
  * Key: \${playerId}
  * Value: player total points

### alliance-leaderboard

not yet implemented

#### Redis structures
* KME:\${eventId}
  * Type: SORTEDSET
  * Key: \${allianceId}
  * Value: alliance total points
* KME:\${eventId}:alliance:\${allianceId}
  * Type: HASHSET
  * Key: \${playerId}
  * Value: player total points
* KME:\${eventId}:tiebreaker
  * Type: HASHSET
  * Key: \${allianceId}
  * Value: last time the alliance scored points

### battle-pass

A battle pass internally behaves the same as a solo event.

#### Redis structures
* KME:\${eventId}
  * Type: HASHSET
  * Key: \${playerId}
  * Value: player total points

## Analytics

### event_enter

Sent by km-sim when a player starts participating in an event

| field      | desc                                                |
| ---------- | --------------------------------------------------- |
| event_id   | Unique ID for a specific event run                  |
| group_id   | The group the player was assigned to in the event   |
| player\_\* | The standard fields associated with a player entity |

### event_leave

Sent by km-sim when a player stops participating in an event

| field      | desc                                                |
| ---------- | --------------------------------------------------- |
| event_id   | Unique ID for a specific event run                  |
| group_id   | The group the player was assigned to in the event   |
| player\_\* | The standard fields associated with a player entity |

### event_phase

Sent when an event enters a new phase

| field                    | desc                                                       |
| ------------------------ | ---------------------------------------------------------- |
| event_id                 | Unique ID for a specific event run                         |
| prototype_id             | Prototype ID for the event                                 |
| phase                    | Phase of the event (see Event Lifecycle)                   |
| opening_time             | Time the event transitions to opening (UTC)                |
| running_time             | Time the event transitions to running (UTC)                |
| tabulating_time          | Time the event transitions to tabulating (UTC)             |
| closing_time             | Time the event transitions to closing (UTC)                |
| closed_time              | Time the event transitions to closed (UTC)                 |
| segment.shard            | For a segmented event, the shard this event run is for     |
| segment.time_zone_offset | For a segmented event, the time zone this event run is for |

### event_prize_collected

Sent when an event prize is collected by a player. Because of eventual consistency issues, it is possible that this event will be fired multiple times for the collection of the same prize for the same player in the same event, so deduplication may be required.

| field        | desc                                                                         |
| ------------ | ---------------------------------------------------------------------------- |
| event_id     | Unique ID for a specific event run                                           |
| prototype_id | Prototype ID for the event                                                   |
| player_id    | Player collecting the prize                                                  |
| prize_id     | Prototype ID for the prize list or the leaderboard (depending on event type) |
| prize_index  | The prize index within the prize prototype                                   |
| contents     | The actual contents of the prize (after loot boxes are opened)               |

### event_prize_earned

Sent when an event prize is earned by a player. Because of eventual consistency issues, it is possible that this event will be fired multiple times for the collection of the same prize for the same player in the same event, so deduplication may be required.

| field        | desc                                                                         |
| ------------ | ---------------------------------------------------------------------------- |
| event_id     | Unique ID for a specific event run                                           |
| prototype_id | Prototype ID for the event                                                   |
| player_id    | Player collecting the prize                                                  |
| prize_id     | Prototype ID for the prize list or the leaderboard (depending on event type) |
| prize_index  | The prize index within the prize prototype                                   |

### quest_step_collect

The quest_step_collect is not sent by km-events, but it is amended to include two new fields if the quest step that was collected earned the player points in an event.

| field        | desc                                                   |
| ------------ | ------------------------------------------------------ |
| event_id     | Unique ID for a specific event run                     |
| event_points | The number of points earned by the player in the event |

# TODO

- [] admin support for events
  - [ ] display participants
  - [ ] add/remove player to/from event?
- [ ] verify that changing segments does not rejoin into existing events
- [ ] make sure shard consolidation works with events
  - [ ] how should it even work
- [ ] write real e2e tests of lifecycle of event
- [ ] BUG: client only gets rank update when their point totals change, but that doesn't account for other player's changing their point total and bumping you around

- [ ] @analytics do we need an analytics event specifically for the leaderboard results?
- [ ] #backlog KM-25160 objective trigger for event prize collection
  - oddly difficult because prize collection is in the events service and triggers are in sim
