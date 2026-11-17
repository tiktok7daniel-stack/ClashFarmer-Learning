# Clash Farmer learning repo — instructions for Claude

This private repo is the learning inbox of the Clash Farmer bot (source: tiktok7daniel-stack/CoC_Bot).
The bot uploads every game screen it did not understand to `inbox/`, and pictures of armies it learned to `armies/`.
Your job: teach the bot. Follow `LEARNING.md` exactly:

1. List `inbox/**/*.jpg` whose file name (without `.jpg`) is not a key in `knowledge/status.json` -> `items`.
2. Open each picture and its `.json` (the OCR words the bot read, the phase, what it already tried).
3. Add or extend a rule in `knowledge/screens.json` (match words + safe actions), or note that nothing is needed.
4. Mark every handled picture in `knowledge/status.json` with `done: true`, the rule id and an Arabic `note_ar`
   (short, what the screen is and what the bot will do now).
5. Never write actions that spend gems, buy, log out, leave a clan or exit the game.
6. Commit as "learn: N screens" and push to `main`. The bot fetches the rules by itself and rebuilds README.md.

Write user-facing text (`note_ar`, `title_ar`) in Egyptian Arabic.

## Screens the bot got stuck on (`inbox/stuck/`)

When the bot stays on the same unknown screen for 20 s (or sees it 3 times, or its watchdog fires) it uploads it
once to `inbox/stuck/<account>_<time>.jpg/.json` (the `.json` has the phase it was in and the words it read).
These come first: they are what blocks the bot right now. Write a rule with safe actions that gets it out of
that screen (back to the village or on with the task) and mark it like any inbox picture (key = file name).

Stuck pictures with phase `upgrade_window` are the window after "Upgrade" in the builder menu that the bot
did not recognise: tell what button confirms the upgrade (never gems / magic items) and add it to
`knowledge/map.json` -> `upgrade_window`; the bot code (`src/builder_menu.py` in CoC_Bot) may need the change too.

## Screens the bot did not understand (`inbox/unknown/`)

Automatic learning: the bot sends every unknown screen at once (when it stays still for a moment, max 60 a day) to
`inbox/unknown/<account>_<time>.jpg/.json` and carries on. `why` in the `.json` says what it was:
- `screen`: a game screen -> a rule in `knowledge/screens.json` like any inbox picture.
- `vision`: a battle base the YOLO model hardly recognised -> label its buildings (YOLO format) for the next
  training of `assets/models/coc_vision.onnx` (`scripts/train_vision.py` in CoC_Bot) and note it in status.json.
Mark them in `knowledge/status.json` like any inbox picture (key = file name).

## Capture sessions (`captures/<session>_<account>/`)

The user can also play while the bot photographs every new screen ("capture mode"), review the pictures and
send them here in one commit. Each session folder has `NNN.jpg` + `NNN.json` (OCR words, `tap` = where the user
tapped before that screen, as fractions of the screen) and a `README.md` table.
When the user says in the chat that they sent a capture:
- go through every picture in order (the taps show the path the user took through the game)
- add rules to `knowledge/screens.json` for screens the bot must be able to handle (popups, menus, buttons on the way)
- for army screens, write the composition to `knowledge/armies.json` (`{"<name or session/NNN>": {"troops": {...}, "spells": {...}, "heroes": [...]}}`)
- mark each picture in `knowledge/status.json` with key `captures/<session folder>/<NNN>` and an Egyptian-Arabic `note_ar`
- reply to the user in Arabic with what was learned from each picture

### Battles inside a capture (`battle_NN/`)
When the user attacked while capturing, the folder `battle_NN/` holds pictures every 2 s and at every deploy, and
`battle.json` with `tap_list` (time in seconds from the first deploy, x/y screen fractions) and `frame_list`.
Study the technique: army used, deploy order, where (edges / funnel / corner), timing of heroes and spells, the result.
Write it to `knowledge/strategies.json`: `{"<session>/battle_NN": {"army": {...}, "plan_ar": "<Egyptian Arabic summary of the technique>",
"deploy": [{"what": "...", "where": "left edge", "t": 0.0}], "result": {...}, "rating": 1-5}}` and tell the user what you learned.
The battle is also saved in the bot as a replayable recording ("learned <date> #N").

### Pictures the user took (manual sessions)
`captures/<session>/` from the "Take picture" button: each `NNN.json` has a `note` written by the user (Arabic) saying
what the screen is or what goes wrong there — follow it. The user is walking through the whole village on purpose:
build `knowledge/map.json` = every screen / menu / button with where it is (x/y fractions) and how to reach it
(`{"screen_id": {"title_ar", "how_to_open": [taps...], "buttons": {"name": [x, y]}, "source": "captures/..."}}`),
then propose new bot features based on it.

### Fingerprints (recognising screens by how they look)
Every rule in `knowledge/screens.json` carries `fingerprints`: the dhash (capture_session.dhash, two ints as strings)
of every picture marked with that rule. The bot uses them when it can't read the words on a screen (distance <= 26).
After marking new pictures, recompute the fingerprints for all rules (pictures from status.json -> their rule).

### Screen model (the bot's brain)
`knowledge/model.json` = features of every labelled picture; the bot recognises screens by look in ~1 ms with it
(and also teaches itself while it plays). After marking new pictures, rebuild it from the bot's repo:
`python scripts/build_screen_model.py <this repo>` — it prints the leave-one-out accuracy (fix wrong labels it
reveals) and also writes the bot's shipped copy `assets/screen_model.json` (commit that in the bot repo too).

## Battle pictures (`inbox/battles/`) — training the bot's eyes

Every base the bot attacks is photographed once (zoomed out) and sent with what the vision model found (`.json`:
`objects` = [name, conf, x1, y1, x2, y2] fractions). To correct one, write YOLO labels to
`knowledge/battle_labels/<file name>.txt` (`class cx cy w h`, classes in CoC_Bot `assets/models/coc_vision.json`).
Retrain from the bot's repo: `python scripts/train_vision.py <dataset> --pseudo <this repo>` (corrected pictures as
they are, the rest labelled by the current model when it is sure) — ship the new ONNX only if its test mAP is higher.

## Daily automatic learning

A scheduled task runs this every night: everything in `inbox/` not yet in status.json (stuck first, then unknown,
then the rest), rules + status + fingerprints, `scripts/build_screen_model.py` (both brains), commit and push here
and the rebuilt `assets/screen_model.json` / `assets/screen_cnn.json` / `assets/screen_map.json` to the bot's branch.
`knowledge/map.json` also drives the bot's navigator: keep button names and `how_to_open` exact.

## The program's errors (`inbox/errors/`)

Every ERROR the bot logs is sent here: `<account>_<time>_<n>.json` (message, where, phase, full `trace`, bot_version)
+ the screenshot taken at that moment (`.jpg`) when there is one. The same error is sent once a day. These are bugs
in the bot's code: fix them in CoC_Bot (branch ccr-d344ab20-onpy71), mark them in status.json (key = file name,
`rule: "bug_fixed"` or `"no_fix_needed"`, note_ar = what it was and what changed).
