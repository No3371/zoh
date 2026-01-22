# Standard Attributes

Standard attributes are common attribute definitions that runtimes should adapt to.

## Scope

Specify the scope of the operation. Accept `"string"` or `*string`. Valid values are `context` or `story`.

Used by: `/set`, `/drop`, `/read`, `/defer`

## Fade

The fade duration in seconds. Accept `double` or `*double`.

Used by: `/show`, `/hide`, `/play`

## Easing

The easing function to use for the fade. Accept `string` or `*string`.

## Loops

The number of loops to repeat. Accept `int` or `*int`.

## Delay

The delay duration in seconds. Accept `double` or `*double`.

## Interval

The interval duration in seconds. Accept `double` or `*double`.

## Opacity

The opacity value. Accept `double` or `*double`. Valid values are 0.0 to 1.0.

## Pos

The position vector. Accept `[list]` or `*[list]` of 3 double values. 

## PosX

The PosX attribute indicates a double for x position. Accept `double` or `*double`. 

## PosY

The PosY attribute indicates a double for y position. Accept `double` or `*double`. 

## PosZ

The PosZ attribute indicates a double for z position. Accept `double` or `*double`. 

## Scale

The scale vector. Accept `[list]` or `*[list]` of 3 double values. 

## ScaleX

The scale X attribute. Accept `double` or `*double`. 

## ScaleY

The scale Y attribute. Accept `double` or `*double`. 

## ScaleZ

The scale Z attribute. Accept `double` or `*double`. 

## Width

The Width attribute indicates a double for width values. Accept `double` or `*double`. 

## Height

The Height attribute indicates a double for height values. Accept `double` or `*double`. 

## Anchor

The Anchor attribute indicates an string for anchor values. Accept `string` or `*string`. Valid values are `top-left`, `top-center`, `top-right`, `center-left`, `center`, `center-right`, `bottom-left`, `bottom-center`, `bottom-right`.

## By

The speaker name. Accept `string` or `*string`.

## Portrait

The avatar image for the speaker. Accept `string` or `*string`.

## Style

The style of the text or choice. Accept `string` or `*string`.

## Wait

Specify if the playback should block until the user responds. Accept `boolean` or `*boolean`.

## Volume

The volume level. Accept `double` or `*double`. Defaults to 1.0.

## Required

Indicates that the operation is required to succeed or return a value. Used by `/set`, `/get`, `/read`.

## Resolve

Indicates that the value should be resolved before operation. Used by `/set`.

## Typed

Specify the type for variable declaration. Accept `string` or `*string`.

## OneOf

Checks the variable to be one of the values in the provided list. Accept `[list]` or `*[list]`.

## Clone

Indicates if the forked context should be a clone of the current context. Accept `boolean` or `*boolean`.

## Inline

Indicates if the called context should copy values back to the caller. Accept `boolean` or `*boolean`.
