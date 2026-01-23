# ZOH Language Spec

ZOH is an embedded scripting language built for storytelling but authored and run like a general programming language.

## Everything is a Verb

Everything in ZOH is a *verb*. Characters `/converse`. Scenes `/show`. Music `/play`. You're not writing a story—you're directing a cast.

Even the core mechanics of ZOH are built on verbs. Want to keep track of a variable? `/set` it. Want to control the flow? `/if` it. Want to loop? `/while` it.

Extensibility is inherent to the language. Define new verbs to fit your needs.

Everything is a *verb*. This uniform structure removes the need for special syntax per feature and makes scripts easier to reason about.

## Ownership to Story Playback

Compared to options like Ink or Twine, ZOH moves story text back into a "content parameter" instead of the "script body". For example, instead of writing just `The quick brown fox jumps over the lazy dog`, you write `/converse "The quick brown fox jumps over the lazy dog"`.

ZOH inverts the relationship between the story script and the player implementation so it can do more than show text line by line. Instead of requiring developers to add effects or mechanisms around the story, ZOH scripts act as stage directions that drive the player to present everything in the scene.

So while ZOH sacrifices some of the conciseness of Ink or Twine, it lets authors interact with the player implementation more directly in the same language.

## Do this, if you can

The same script can power a text console, a visual novel, or a 3D cinematic. ZOH doesn't care what your game looks like; that's the runtime's job. Write the story once, present it however you want.

One core principle of ZOH is Progressive Enhancement. As an embedded scripting language, ZOH is expected to be implemented by runtimes in various languages, on various platforms, minimalistic or full-fledged.

The idea is that any ZOH script should run in any runtime that implements the core protocol, while runtimes extended with additional verbs can provide richer experiences and represent the script better to the user.

This is because verbs are intentions executed by the runtime. On paper, no verb is required; the actual behavior of any verb is up to the runtime.

In practice, the language itself is built on verbs (including fundamental features like variable management and flow control), so a runtime must implement the core verbs and comply with their behavior definitions to run a ZOH script.

## At a Glance

```zoh
The Last Coffee Shop
===

@opening
:: Set the stage
/show [id: "bg"] [fade: 1.5] "cafe_night.png";
/play [id: "ambience"] [loops: -1] "rain_and_jazz.ogg";

*player_name <- "stranger";
*trust <- 0;

:: Fork parallel contexts—the world keeps moving while you talk
====+ @cafe_atmosphere;
====+ @barista_idle;

/converse [By: "Narrator"] "11:47 PM. The cafe closes at midnight. Rain scribbles on the glass.";
/converse "A woman in a red coat sits alone by the window, a sealed envelope on the table.";

====> @first_approach;

@first_approach
/choose/
    prompt: "The rain picks up outside."
    true  "Sit across from her"         "sit"
    true  "Order something first"       "order"
    true  "Leave—this isn't your story" "leave"
/; -> *choice;

/switch/ *choice
    "sit"   /jump ?, "sit_down";
    "order" /jump ?, "at_counter";
    "leave" /jump ?, "walk_away";
/; -> *dest;
/do *dest;

@at_counter
/converse [By: "Barista"] "Last call. What'll it be?";
/prompt "Your order:"; -> *order;
/converse [By: "Barista"] "Coming right up.";
/sleep 1.5;

*trust [resolve] <- `*trust + 1`;    :: Small gesture of patience
====> @sit_down;

@sit_down
/push <stop_idle>, true;    :: Signal the barista to stop her idle loop

/converse [By: "Woman"] "You're not from around here.";

/choose/
    prompt: "She's studying your face like it belongs to a name she's trying to place."
    true                "How can you tell?"   "deflect"
    `*trust > 0`        "Neither are you."    "mirror"
    true                "Say nothing"         "silence"
/; -> *response;

/if *response, is: "mirror", /sequence/
    *trust [resolve] <- `*trust + 2`;
    /converse [By: "Woman"] "...No. I'm not.";
    /converse "For the first time, she smiles.";
/;;

/if *response, is: "silence", /sequence/
    /converse "The silence stretches. Outside, a car glides past with no headlights.";
    /play [id: "car"] "car_passing.ogg";
/;;

/push <ending>, *trust;
====> @closing_time;

@walk_away  
/converse "Some doors are better left unopened. You leave with the rain.";
/push <ending>, -1;
====> @closing_time;

@closing_time
/pull <ending>; -> *final_trust;

/if `*final_trust < 0`, /sequence/
    /converse [By: "Narrator"] "The cafe light flickers off behind you. Whatever she needed will wait for someone else.";
    /stop [fade: 3] "ambience";
    /exit;
/;;

/if `*final_trust >= 3`, /sequence/
    /converse [By: "Woman"] "Wait—";
    /converse "She slides a folded napkin across the table. A map, a time, and a single word: \"basement\".";
    /show [fade: 0.5] "napkin_note.png";
/;;

/converse [By: "Narrator"] "The barista dims the lights. Midnight, and the street goes quiet.";
/stop [fade: 2] "ambience";
/sleep 2;

:: ═══════════════════════════════════════════════════════════
:: PARALLEL CONTEXTS — These run independently alongside the main context above
:: ═══════════════════════════════════════════════════════════

@cafe_atmosphere
:: Ambient events that fire periodically
/loop 999, /sequence/
    /sleep /rand 5.0, 8.0;;
    /roll "thunder.ogg", "dishes.ogg", "door_chime.ogg"; -> *sound;
    /play *sound;
/;;

@barista_idle
:: The barista has her own life until you interrupt it
/while `true`, /sequence/
    /pull <stop_idle>, timeout: 8; -> *stop;
    /if *stop, /exit;;
    /roll 
        "She wipes down the counter.",
        "She counts the tips, then looks toward the door.",
        "She hums along to the music."
    ; -> *action;
    /converse [By: "—"] *action;
/;;
```

This example shows:
- **Parallel contexts** (`====+`) running ambient cafe life alongside the main dialogue
- **Channel synchronization** (`/push`, `/pull`) to coordinate between contexts  
- **Conditional choices** with expressions (`` `*trust > 0` ``)
- **Presentation verbs** (`/show`, `/play`, `/stop`) directing visuals and audio
- **Block form** (`/verb/ ... /;`) for readable multi-line structures
- **Randomization** (`/rand`, `/roll`) for dynamic ambient events
- **Variable state** tracking player decisions across the scene

...


Please refer to [spec.md](spec.md) for the language spec.
