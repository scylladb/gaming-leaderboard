# Getting Started: Gaming Leaderboard Example


## Introduction

This guide will show you how to create an monstrously fast and scalable Gaming Leaderboard application from scratch in many languages using ScyllaDB as datastore. It'll walk you through all the stages of the development process, from gathering requirements to building and running the application.

## Requirements

The example application uses ScyllaDB Cloud to run a three-node ScyllaDB cluster. You can claim your free Scylla Cloud account [here](https://scylladb.com/cloud).


## Project Details

Since we're building an API, here's some spefications on what we're going to work with. Each language has your own way to implement it, so let's focus on the I/O. 


### 1. Find a Submission

After you submit a specific gameplay score to the API, you'll be able to fetch it. 

* **Route:** /submissions/{submission_id}
* **Verb:** GET

#### Response Example
```json
{
    "message": "Submission found.",
    "submission": {
        "difficulty": "daniel-reiss",
        "id": "01abf900-dcc9-4f61-94fe-a353d4e85bf6",
        "instrument": "guitar",
        "modifiers": [
            "no-modifiers"
        ],
        "played_at": "2024-02-14T18:34:46.985Z",
        "player_id": "daniel-reiss",
        "score": 30005,
        "song_id": "stone-audioslave"
    }
}
```


### 2. Submit a Gameplay 

After you submit a specific gameplay score to the API, you'll be able to fetch it. 

* **Route:** /submissions/{submission_id}
* **Verb:** POST


#### Example Payload

```json
{
    "song_id": "stone-audioslave",
    "player_id": "startupme",
    "modifiers": [
        "no-modifiers"
    ],
    "score": 19999,
    "difficulty": "expert",
    "instrument": "guitar"
}
```

#### Response Example

```json 

{
    "message": "Submission created successfully",
    "submission": {
        "difficulty": "expert",
        "id": "531be3f3-2d4f-4d7b-bfb3-1366e2a632c4",
        "instrument": "guitar",
        "modifiers": [
            "no-modifiers"
        ],
        "played_at": "2024-02-26T13:13:51.331231046Z",
        "player_id": "startupme",
        "score": 19999,
        "song_id": "stone-audioslave"
    }
}
```

### 3. Retrieve Leaderboard by Song

With many players submitting a specific song, now it's time to check who did the best score. However, you will have to filter by specific items:

* **Route:** /leaderboard/{song_id}
* **Verb:** POST
* **Filters:**
  * Instrument: String
  * Difficulty: String
  * Modifiers: String[]

#### Example Request

**URI:** /leaderboard/{song_id}?instrument=guitar&difficulty=expert&modifiers[]=no-modifiers

#### Example Response

```json
[
    {
        "difficulty": "expert",
        "id": "8151cb1c-a1a0-473d-b491-74c7a3988189",
        "instrument": "guitar",
        "modifiers": [
            "no-modifiers"
        ],
        "played_at": "2024-02-14T18:34:46.999Z",
        "player_id": "daniel-reis",
        "score": 30005,
        "song_id": "stone-audioslave"
    },
    {
        "difficulty": "expert",
        "id": "8e15add1-020e-4a84-90cc-2c48d6c246cb",
        "instrument": "guitar",
        "modifiers": [
            "no-modifiers"
        ],
        "played_at": "2024-02-26T13:13:51.346Z",
        "player_id": "startupme",
        "score": 19999,
        "song_id": "stone-audioslave"
    }
]
```


### Additional Resources

-   [Scylla Essentials](https://university.scylladb.com/courses/scylla-essentials-overview/) course on Scylla University. It provides an introduction to Scylla and explains the basics.
-   [Data Modeling and Application Development](https://university.scylladb.com/courses/data-modeling/) course on Scylla University. It explains basic and advanced data modeling techniques, including information on workflow application, query analysis, denormalization, and other NoSQL data modeling topics.
-   [Scylla Documentation](https://docs.scylladb.com/)
-   Scylla users [slack channel](http://slack.scylladb.com/)

