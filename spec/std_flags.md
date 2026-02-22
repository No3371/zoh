# Standard Flags

Standard flags are common flag definitions suggested to be honored.

## Interactive
The flag indicates if the playback should be interactive. Accept `boolean` or `*boolean`. Default to `true`.

Used by `/converse` verb.

## Instant
The flag indicates if unnecessary playback of verb behaviors should be skipped or instantly completed to result state. Accept `boolean` or `*boolean`. Default to `false`.

Use case: toggle of effects/animation, skipping of typewriter style dialogues, etc.

## Pace
The flag determines how long does the context wait to execute next verb.

Defaults to `0`, which means the context instantly execute next verb once the previous verb is finished.