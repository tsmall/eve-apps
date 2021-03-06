# Countdown timer
## Kitchen timer
This program provides a countdown timer for use when cooking. For example, say you're following a recipe that says you need to boil potatoes for 8 minutes, and cook the salmon for 5 minutes. It's useful to have timers for both of those tasks running at the same time, and to label them so you know how much time is left for each task. That's what this program does.

That's the goal, at least. For now this program is simpler. It provides only a single timer.

## Data model
This program only lets the user have a single timer. So only a single timer is needed, and it can be created when the program starts running. The timer needs to know how much time it should run for, how much time has elapsed, whether it's running, and whether it has finished. Both the goal and elapsed time will be stored as seconds.

```
commit
  [#timer goal: 0 elapsed: 0 running: false completed: false]
```

## User interface
The UI lets the user set the amount of time desired, start a timer, see how much time remains before the timer is completed, and indicated when a timer has finished. It accesses the global timer directly.

### Stopped timer UI
This first part of the UI is the form, where the user can set the timer. It's only shown when the timer isn't running.

```
search
  timer = [#timer running: false]

bind @browser
  [#div children:
    [#span text: "Start a timer for "]
    [#input #goal-field
      type: "text"
      value: "5"
      style: [width: "2em"]]
    [#span text: " minutes"]]
  [#div children:
    [#button #start-button text: "Start timer"]]
```

### Running timer UI
When the timer is running, the UI shows the elapsed time and goal time.

The `elapsed` and `goal` times are in seconds, but the UI should show them as minutes and seconds in the standard `M:SS` format. To convert them, a #`time` record is written for each time. They're tagged so they can be easily identified below.

```
search
  [#timer running: true, elapsed, goal]

bind
  [#time-val #elapsed seconds: elapsed]
  [#time-val #goal seconds: goal]
```

With that, Eve worked its magic and we now have the formatted strings, which we'll use to show the timer's current value to the user.

```
search
  [#timer running: true]
  [#time-val #elapsed formatted: elapsed]
  [#time-val #goal formatted: goal]

bind @browser
  [#div children:
    [#span text: "{{elapsed}} / {{goal}}"]]
  [#div children:
    [#button #stop-button text: "Stop timer"]]
```

## UI behavior
### Starting a new timer
When the user clicks the start button, the timer's goal time should be set to the number of minutes the user entered and it should start running. The timer's elapsed time also needs to be reset to zero to reset the counter back to its starting state.

```
search @event @browser @session
[#click element: [#start-button]]
  [#goal-field value]
  timer = [#timer]
  new-goal-mins = convert[value: value, to: "number"]
  new-goal-secs = new-goal-mins * 60

commit
  timer <- [goal: new-goal-secs, elapsed: 0, running: true]
```

### Stopping a running timer
When the user clicks the stop timer button, the timer is simply stopped.

```
search @event @browser @session
  [#click element: [#stop-button]]
  timer = [#timer]

commit
  timer.running := false
```

## Timer behavior
When a timer is running, its elapsed time should increment every second. It might seem to make sense to use Eve's clock's seconds fields to increment the timer, but that doesn't result in the correct behavior. Unless the user happens to start a timer right when the current second changes, the timer's first second is too short. So, instead of seconds, the timer uses Eve's frames, which is hard-coded for 60 frames per second.

### Setup
The timer needs to keep track of the last processed frame, so it can know when 60 frames (one second) have elapsed. The timer is created with an initial frame value of 0.

```
search
  timer = [#timer]

commit
  timer.last-frame := 0
```

### Running
When the timer is running, its elapsed time is incremented once every 60 frames.

```eve
search
  [#time frames]
  timer = [#timer running: true, last-frame, elapsed]
  frames - last-frame > 60

commit
  timer <- [elapsed: elapsed + 1, last-frame: frames]
```

### Finished
When the timer hits the goal time, it's done.

```
search
timer = [#timer running: true, elapsed, goal]
  elapsed = goal

commit
  timer <- [running: false, completed: true]
```

## Time conversion
Part of this program needs to convert time from a number of seconds to the standard, human-friendly representation of `M:SS`. This calculation needs to be performed multiple times. In Eve, the unit of composition is a pattern-matched record, not a function. (In fact, you can't even define your own functions yet.) That means this solution will be a bit different.

To perform this calculation, the code will write #`time-val` records, which must have a `seconds` field. All #`time-val` records are automatically extended to have a `formatted` field, which contains the seconds converted into `M:SS` format.

```
search
  time = [#time-val seconds]
  mins = floor[value: seconds / 60]
  secs = mod[value: seconds, by: 60]
  secs-padded = if secs > 9 then secs else "0{{secs}}"
  formatted = "{{mins}}:{{secs-padded}}"

commit
  time.formatted := formatted
```
