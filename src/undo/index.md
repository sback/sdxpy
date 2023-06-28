---
syllabus:
-   FIXME
depends:
-   viewer
---

-   Bring over viewer app
    -   `app.py`, `buffer.py`, `cursor.py`, `window.py`, `util.py`
    -   `Window`, `Cursor`, `Buffer`, `App`
    -   Add call to `._add_log` to `App._interact` (another unanticipated hook)

-   Create headless versions
    -   `HeadlessScreen` replaces the `curses` screen
        -   Takes keystrokes as input (what we're simulating)
        -   Automatically generate Ctrl-X when out of keystrokes
        -   Store current state of display in rectangular grid
        -   Would make more sense for `App` to have a method that gets keys
    -   `HeadlessWindow` requires a size
        -   Violates Liskov Substitution Principle
    -   `HeadlessApp` fills in logging methods
    -   First tests make sure we can move around
    -   `headless.py`
    -   `test_headless.py`

-   Create insert/delete version
    -   `InsertDeleteBuffer` in `insert_delete.py`
        -   Add methods to insert and delete in buffer
	-   Delete character *under* cursor, not to the left
    -   `InsertDeleteApp` provides `_get_key` in the app to return (family, key)
        -   `_interact` dispatches to one of two cases (special-purpose or generic + key)
        -   Could define per-key method to make customization easier
    -   `test_insert_delete.py`
    -   But one test fails: empty screen (cursor isn't on top of a character)
        -   Our focus is undo, so we'll ignore this for now
	-   Tackle it in the exercises

-   Record history of insertions and deletions
    -   `HistoryApp` in `history.py` creates a list `_history`
    -   Modify `_do_INSERT` and `_do_DELETE` to append records
    -   Could add more logic to history recording, but this approach is broken anyway
    -   Insert, move, move, undo: where does it leave the cursor?
    -   Next step is to create objects that record actions

-   `action.py` is the fully object-oriented version
    -   `Action` records the app and has `do` and `undo`
    -   Derive `Insert` to:
        -   insert a character and a position
	-   delete that saved character
    -   Derive `Exit` to stop the app running
        -   no `undo` action (should never arise)
    -   `_interact` is now:
        -   get the action name
	-   find a handler
	-   call it to construct an object
	-   `do` that object
	-   append the action to the history so that we can undo it
    -   works until we try to undo (see `undoable.py`), at which point we get stuck in a loop
        -   undo calls undo calls undo
	-   so write `undoable.py` with "save this?"

---

## The Problem

-   Want to change files as well as viewing them

-   So modify the file viewer of [%x viewer %] to allow editing

-   And since people make mistakes when editing, implement undo

---

## Starting Point

-   `Window` can draw lines and report its size

-   `Buffer` stores lines of text,
    keeps track of a viewport,
    and transforms buffer coordinates to screen coordinates

-   `Cursor` knows its position in the buffer
    and can move up, down, left, and right

-   `App` makes a window, a buffer, and a cursor,
    then maps keys to actions

[% figure
   slug="undo-classes"
   img="classes.svg"
   alt="Classes in file viewer"
   caption="Relationships between classes in file viewer"
%]

## A Headless Screen

-   Create a [%g headless "headless" %] screen for testing

-   Store current state of display in rectangular grid

-   Take a list of keystrokes (for simulation)

    -   Would have made more sense for `App` to have a method
        that gets keystrokes

## A Headless Screen

[% inc file="headless.py" keep="screen" %]

## Bad But Necessary

-   Also need to define `HeadlessWindow` to take a size
    and pass it to the screen

[% inc file="headless.py" keep="window" %]

-   Violates the
    [%i "Liskov Substitution Principle" %][%/i%]

## Logging

-   Record keys, cursor position, and screen contents for testing

[% inc file="headless.py" keep="app" %]

## Testing

[% inc file="test_headless.py" keep="example" %]

-   Last key is always `CONTROL_X` (exit)

## Insertion and Deletion

[% inc file="insert_delete.py" keep="buffer" %]

-   Delete character *under* the cursor, not to the left

-   A little [%i "defensive programming" %][%/i%] as well

## Application

[% inc file="insert_delete.py" keep="app" %]

[% inc file="insert_delete.py" keep="action" %]

## Application

[% inc file="insert_delete.py" keep="dispatch" %]

-   Add `_get_key` to the application

    -   Return family (for generic handlers) and key (specific)

    -   Could provide per-key methods to make customization easier

## Testing

-   Write a function to make the fixture and run the test

[% inc file="test_insert_delete.py" keep="fixture" %]

-   Tests are straightforward

[% inc file="test_insert_delete.py" keep="example" %]

## Edge Case

-   Can't delete when in an empty screen

[% inc file="test_insert_delete.py" keep="empty" %]

-   Our focus is implementing undo, so leave this for an exercise

## Recording History

-   In order to undo things we have to:

    1.  keep track of *actions* and reverse them, or

    2.  keep track of *state* and restore it

-   Recording actions requires less space but can be trickier to implement

-   Have actions append entries to a log

## The Simple Approach

[% inc file="history.py" keep="app" %]

-   What about undoing cursor movement?

-   And do we write an interpreter for these log records?

## Verbs as Nouns

-   Use the [%g command_pattern "Command" %] design pattern

-   Each action (verb) is an object (noun)
    with methods to go forward and backward

-   Every action is derived from
    an [%i "abstract base class" %][%/i%]

[% inc file="action.py" keep="Action" %]

## Insertion and Deletion

[% inc file="action.py" keep="Insert" %]

[% inc file="action.py" keep="Delete" %]

## Movement

[% inc file="action.py" keep="Move" %]

-   Give `Cursor` methods to move in a particular direction (by name)
    and move to a particular location

## Application

[% inc file="action.py" keep="interact" %]

-   Create the action object

-   Call its `.do` method

-   Modify all action methods to take a key to simplify the code a little

## Application

[% inc file="action.py" keep="actions" %]

-   And it *almost* works!

-   Our first implementation of `Undo` creates an infinite loop
    because it puts itself on the undo stack
    and then does the action on the top of the stack

## Finally

-   Modify `Action` to have a `.save` method that returns `True`

-   Override in `Undo`

[% inc file="undoable.py" keep="Undo" %]

-   Only add object to undo stack if `.save` is `True`

-   More general design would give `Action` a `.post_action` method
    that by default adds the action to the undo stack

## Summary

[% figure
   slug="undo-concept-map"
   img="concept_map.svg"
   alt="Concept map of undo"
   caption="Concept map"
%]